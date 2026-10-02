# Part 61: Microservices Architecture
## Steps 886-900: Building Microservices in Nim

---

## 🎯 เป้าหมายของ Part นี้

- Microservice boundaries & contracts
- Inter-service communication (HTTP + async messaging)
- Saga pattern for distributed transactions
- Circuit breaker
- Service contract testing
- Docker Compose setup

---

## Step 886: Service Contracts

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, math, algorithm

# ============================
# Service contract (OpenAPI-inspired)
# ============================

type
  ContractField = object
    name: string
    type_: string
    required: bool
    format: string

  ContractOperation = object
    operationId: string
    method_: string
    path: string
    requestBody: seq[ContractField]
    responseFields: seq[ContractField]
    statusCodes: seq[int]

  ServiceContract = object
    name: string
    version: string
    baseUrl: string
    operations: Table[string, ContractOperation]

proc defineContract(name, version, baseUrl: string): ServiceContract =
  ServiceContract(
    name: name,
    version: version,
    baseUrl: baseUrl,
    operations: initTable[string, ContractOperation]()
  )

proc addOperation(c: var ServiceContract, op: ContractOperation) =
  c.operations[op.operationId] = op

# ============================
# User Service contract
# ============================

proc userServiceContract(): ServiceContract =
  var c = defineContract("user-service", "v1", "http://user-service:8080")

  c.addOperation(ContractOperation(
    operationId: "getUser",
    method_: "GET",
    path: "/users/{id}",
    requestBody: @[],
    responseFields: @[
      ContractField(name: "id", type_: "string", required: true),
      ContractField(name: "name", type_: "string", required: true),
      ContractField(name: "email", type_: "string", required: true),
      ContractField(name: "createdAt", type_: "string", required: false),
    ],
    statusCodes: @[200, 404]
  ))

  c.addOperation(ContractOperation(
    operationId: "createUser",
    method_: "POST",
    path: "/users",
    requestBody: @[
      ContractField(name: "name", type_: "string", required: true),
      ContractField(name: "email", type_: "string", required: true),
    ],
    responseFields: @[
      ContractField(name: "id", type_: "string", required: true),
      ContractField(name: "name", type_: "string", required: true),
      ContractField(name: "email", type_: "string", required: true),
    ],
    statusCodes: @[201, 400, 409]
  ))

  return c

# ============================
# Circuit breaker
# ============================

type
  CbState = enum
    cbClosed,    # normal operation
    cbOpen,      # rejecting calls
    cbHalfOpen   # testing recovery

  CircuitBreaker = object
    name: string
    state: CbState
    failureCount: int
    successCount: int
    threshold: int       # failures before opening
    timeout: float       # seconds before half-open
    openedAt: float
    halfOpenMaxCalls: int
    halfOpenCalls: int

proc newCircuitBreaker(name: string, threshold = 5, timeout = 30.0): CircuitBreaker =
  CircuitBreaker(
    name: name,
    state: cbClosed,
    failureCount: 0,
    successCount: 0,
    threshold: threshold,
    timeout: timeout,
    halfOpenMaxCalls: 3
  )

proc canCall(cb: var CircuitBreaker): bool =
  case cb.state
  of cbClosed: return true
  of cbOpen:
    if epochTime() - cb.openedAt > cb.timeout:
      cb.state = cbHalfOpen
      cb.halfOpenCalls = 0
      echo fmt"[CB:{cb.name}] Half-open"
      return true
    return false
  of cbHalfOpen:
    if cb.halfOpenCalls < cb.halfOpenMaxCalls:
      inc cb.halfOpenCalls
      return true
    return false

proc recordSuccess(cb: var CircuitBreaker) =
  inc cb.successCount
  case cb.state
  of cbClosed:
    cb.failureCount = 0
  of cbHalfOpen:
    if cb.halfOpenCalls >= cb.halfOpenMaxCalls:
      cb.state = cbClosed
      cb.failureCount = 0
      echo fmt"[CB:{cb.name}] Closed (recovered)"
  else: discard

proc recordFailure(cb: var CircuitBreaker) =
  inc cb.failureCount
  case cb.state
  of cbClosed:
    if cb.failureCount >= cb.threshold:
      cb.state = cbOpen
      cb.openedAt = epochTime()
      echo fmt"[CB:{cb.name}] Open (failures: {cb.failureCount})"
  of cbHalfOpen:
    cb.state = cbOpen
    cb.openedAt = epochTime()
    echo fmt"[CB:{cb.name}] Re-opened"
  else: discard

proc withCircuitBreaker[T](cb: var CircuitBreaker,
                            operation: proc(): Future[T] {.async.},
                            fallback: proc(): Future[T] {.async.}
                           ): Future[T] {.async.} =
  if not canCall(cb):
    echo fmt"[CB:{cb.name}] Rejected (circuit open)"
    return await fallback()

  try:
    let result = await operation()
    recordSuccess(cb)
    return result
  except CatchableError as e:
    recordFailure(cb)
    echo fmt"[CB:{cb.name}] Error: {e.msg}"
    return await fallback()

# ============================
# Distributed saga
# ============================

type
  SagaState = enum
    ssRunning, ssCompleted, ssCompensating, ssAborted

  SagaStep = object
    name: string
    execute: proc(): Future[bool] {.async.}
    compensate: proc(): Future[void] {.async.}

  Saga = object
    id: string
    steps: seq[SagaStep]
    completedSteps: seq[string]
    state: SagaState
    error: string

proc newSaga(id: string): Saga =
  Saga(id: id, steps: @[], completedSteps: @[], state: ssRunning)

proc addStep(saga: var Saga, name: string,
             execute: proc(): Future[bool] {.async.},
             compensate: proc(): Future[void] {.async.}) =
  saga.steps.add(SagaStep(name: name, execute: execute, compensate: compensate))

proc runSaga(saga: var Saga): Future[bool] {.async.} =
  echo fmt"[Saga:{saga.id}] Starting"

  for step in saga.steps:
    echo fmt"[Saga:{saga.id}] Step: {step.name}"
    try:
      let success = await step.execute()
      if success:
        saga.completedSteps.add(step.name)
      else:
        echo fmt"[Saga:{saga.id}] Step failed: {step.name}"
        saga.state = ssCompensating

        # Compensate completed steps in reverse
        for i in countdown(saga.completedSteps.len - 1, 0):
          let completedName = saga.completedSteps[i]
          for s in saga.steps:
            if s.name == completedName:
              echo fmt"[Saga:{saga.id}] Compensating: {completedName}"
              await s.compensate()

        saga.state = ssAborted
        return false

    except CatchableError as e:
      saga.error = e.msg
      saga.state = ssAborted
      return false

  saga.state = ssCompleted
  echo fmt"[Saga:{saga.id}] Completed"
  return true

# ============================
# Mock service calls
# ============================

var reservedInventory = false
var createdOrder = false
var chargedPayment = false

proc reserveInventory(): Future[bool] {.async.} =
  await sleepAsync(10)
  reservedInventory = true
  echo "  → Inventory reserved"
  return true

proc cancelInventory(): Future[void] {.async.} =
  await sleepAsync(5)
  reservedInventory = false
  echo "  → Inventory reservation cancelled"

proc createOrder(): Future[bool] {.async.} =
  await sleepAsync(10)
  createdOrder = true
  echo "  → Order created"
  return true

proc cancelOrder(): Future[void] {.async.} =
  await sleepAsync(5)
  createdOrder = false
  echo "  → Order cancelled"

var failPayment = false
proc chargePayment(): Future[bool] {.async.} =
  await sleepAsync(10)
  if failPayment:
    echo "  → Payment FAILED (simulated)"
    return false
  chargedPayment = true
  echo "  → Payment charged"
  return true

proc refundPayment(): Future[void] {.async.} =
  await sleepAsync(5)
  chargedPayment = false
  echo "  → Payment refunded"

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Microservices Demo ==="

  # Contract display
  echo "\n--- Service contracts ---"
  let userContract = userServiceContract()
  echo fmt"Service: {userContract.name} {userContract.version}"
  for opId, op in userContract.operations:
    echo fmt"  {op.method_} {op.path} -> {op.statusCodes}"

  # Circuit breaker
  echo "\n--- Circuit breaker ---"
  var cb = newCircuitBreaker("payment-service", threshold = 3, timeout = 1.0)

  # Simulate failures to trip breaker
  for i in 0..<4:
    discard await withCircuitBreaker(cb,
      proc(): Future[string] {.async.} =
        raise newException(IOError, "connection refused"),
      proc(): Future[string] {.async.} = return "fallback_response"
    )

  # Should now be open
  let blocked = not canCall(cb)
  echo fmt"Circuit is open: {blocked}"

  # Wait for timeout and recover
  await sleepAsync(1100)
  let canCallNow = canCall(cb)
  echo fmt"Can call after timeout: {canCallNow}"

  # Saga: successful
  echo "\n--- Saga: success scenario ---"
  var successSaga = newSaga("order_saga_1")
  successSaga.addStep("reserve_inventory", reserveInventory, cancelInventory)
  successSaga.addStep("create_order", createOrder, cancelOrder)
  successSaga.addStep("charge_payment", chargePayment, refundPayment)

  let success = await runSaga(successSaga)
  echo fmt"Saga result: {success} (state: {successSaga.state})"
  echo fmt"  inventory={reservedInventory} order={createdOrder} payment={chargedPayment}"

  # Saga: failure with compensation
  echo "\n--- Saga: failure + compensation ---"
  reservedInventory = false
  createdOrder = false
  chargedPayment = false
  failPayment = true

  var failSaga = newSaga("order_saga_2")
  failSaga.addStep("reserve_inventory", reserveInventory, cancelInventory)
  failSaga.addStep("create_order", createOrder, cancelOrder)
  failSaga.addStep("charge_payment", chargePayment, refundPayment)

  let failed = await runSaga(failSaga)
  echo fmt"Saga result: {failed} (state: {failSaga.state})"
  echo fmt"  inventory={reservedInventory} order={createdOrder}"

waitFor demo()
```

---

## 📝 สรุป Part 61

| Steps | หัวข้อ |
|-------|--------|
| 886 | Service contracts, circuit breaker pattern |
| 887-900 | Distributed saga (execute + compensate), mock service calls |

---

**← [Part 60: ORM](part_60_orm.md) | [Part 62: Performance →](part_62_performance.md)**
