# Part 50: Data Pipeline & ETL
## Steps 721-735: Building Production Data Pipelines

---

## 🎯 เป้าหมายของ Part นี้

- ETL pipeline architecture
- Data sources (CSV, JSON, DB mock)
- Transformers (filter, map, aggregate)
- Data sinks (file, DB, API)
- Pipeline orchestration
- Error handling & dead letter

---

## Step 721: Pipeline Core Types

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, options, math

# ============================
# Core pipeline types
# ============================

type
  DataRecord = object
    id: string
    data: Table[string, JsonNode]
    metadata: Table[string, string]
    timestamp: float
    source: string

  PipelineError = object
    record: DataRecord
    stage: string
    error: string
    timestamp: float

  ExtractResult = object
    records: seq[DataRecord]
    totalRows: int
    errors: seq[PipelineError]
    durationMs: float

  TransformResult = object
    records: seq[DataRecord]
    dropped: int
    errors: seq[PipelineError]
    durationMs: float

  LoadResult = object
    loaded: int
    errors: seq[PipelineError]
    durationMs: float

  PipelineStats = object
    extracted: int
    transformed: int
    loaded: int
    dropped: int
    errors: int
    durationMs: float

# ============================
# Extractor interface
# ============================

type
  CsvExtractor = object
    path: string
    delimiter: char
    hasHeader: bool

proc extractCsv(e: CsvExtractor): ExtractResult =
  let start = epochTime()
  var records: seq[DataRecord]
  var errors: seq[PipelineError]

  # Simulate CSV data
  let csvData = """id,name,email,age,country
1,Alice,alice@example.com,30,US
2,Bob,bob@example.com,25,UK
3,Charlie,charlie@example.com,,TH
4,Diana,diana@example.com,28,US
5,Eve,bad-email,22,AU"""

  let lines = csvData.split('\n')
  if lines.len < 2:
    return ExtractResult(durationMs: (epochTime() - start) * 1000)

  let headers = lines[0].split(e.delimiter)

  for i in 1..<lines.len:
    let line = lines[i].strip()
    if line.len == 0: continue

    let parts = line.split(e.delimiter)
    var record = DataRecord(
      id: fmt"csv_{i}",
      data: initTable[string, JsonNode](),
      metadata: initTable[string, string](),
      timestamp: epochTime(),
      source: e.path
    )

    for j, header in headers:
      if j < parts.len:
        record.data[header] = %parts[j]
      else:
        record.data[header] = newJNull()

    records.add(record)

  let duration = (epochTime() - start) * 1000
  echo fmt"[Extract:CSV] Read {records.len} records from {e.path}"
  return ExtractResult(
    records: records,
    totalRows: records.len,
    errors: errors,
    durationMs: duration
  )

type
  JsonExtractor = object
    data: seq[JsonNode]
    sourceName: string

proc extractJson(e: JsonExtractor): ExtractResult =
  let start = epochTime()
  var records: seq[DataRecord]

  for i, node in e.data:
    var record = DataRecord(
      id: fmt"json_{i + 1}",
      data: initTable[string, JsonNode](),
      metadata: initTable[string, string](),
      timestamp: epochTime(),
      source: e.sourceName
    )

    if node.kind == JObject:
      for k, v in node:
        record.data[k] = v

    records.add(record)

  let duration = (epochTime() - start) * 1000
  echo fmt"[Extract:JSON] Read {records.len} records"
  return ExtractResult(
    records: records,
    totalRows: records.len,
    errors: @[],
    durationMs: duration
  )

# ============================
# Transformers
# ============================

type
  FilterTransformer = object
    field: string
    operator: string
    value: string

proc applyFilter(t: FilterTransformer, record: DataRecord): bool =
  if t.field notin record.data: return false
  let v = record.data[t.field]
  let sv = if v.kind == JString: v.getStr() else: $v

  case t.operator
  of "eq": return sv == t.value
  of "ne": return sv != t.value
  of "gt": 
    try: return parseFloat(sv) > parseFloat(t.value)
    except: return false
  of "lt":
    try: return parseFloat(sv) < parseFloat(t.value)
    except: return false
  of "contains": return t.value in sv
  of "not_empty": return sv.len > 0 and sv != "null"
  else: return true

type
  MapTransformer = object
    mappings: seq[tuple[srcField, dstField: string, transform: string]]

proc applyMap(t: MapTransformer, record: var DataRecord) =
  for m in t.mappings:
    if m.srcField notin record.data: continue
    let v = record.data[m.srcField]

    let transformed: JsonNode = case m.transform
      of "uppercase":
        %(if v.kind == JString: v.getStr().toUpperAscii() else: v.getStr())
      of "lowercase":
        %(if v.kind == JString: v.getStr().toLowerAscii() else: v.getStr())
      of "int":
        try: %(parseInt(v.getStr()))
        except: v
      of "float":
        try: %(parseFloat(v.getStr()))
        except: v
      of "trim":
        %(v.getStr().strip())
      else: v

    if m.dstField != m.srcField:
      record.data.del(m.srcField)
    record.data[m.dstField] = transformed

type
  EnrichTransformer = object
    field: string
    lookupTable: Table[string, JsonNode]
    targetField: string

proc applyEnrich(t: EnrichTransformer, record: var DataRecord) =
  if t.field notin record.data: return
  let key = record.data[t.field].getStr()
  if key in t.lookupTable:
    record.data[t.targetField] = t.lookupTable[key]

# ============================
# Loaders
# ============================

type
  JsonFileLoader = object
    path: string
    pretty: bool

var loadedRecords: seq[DataRecord] = @[]

proc loadToMemory(records: seq[DataRecord]): LoadResult =
  let start = epochTime()
  loadedRecords.add(records)
  let duration = (epochTime() - start) * 1000
  echo fmt"[Load:Memory] Loaded {records.len} records"
  return LoadResult(loaded: records.len, errors: @[], durationMs: duration)

proc loadToJson(loader: JsonFileLoader, records: seq[DataRecord]): LoadResult =
  let start = epochTime()
  var output = newJArray()

  for r in records:
    var obj = newJObject()
    for k, v in r.data:
      obj[k] = v
    output.add(obj)

  let json = if loader.pretty: output.pretty() else: $output
  writeFile(loader.path, json)

  let duration = (epochTime() - start) * 1000
  echo fmt"[Load:JSON] Wrote {records.len} records to {loader.path}"
  return LoadResult(loaded: records.len, errors: @[], durationMs: duration)

# ============================
# Pipeline orchestrator
# ============================

type
  PipelineStage = enum
    psExtract, psFilter, psMap, psEnrich, psLoad

  Pipeline = object
    name: string
    filters: seq[FilterTransformer]
    mappers: seq[MapTransformer]
    enrichers: seq[EnrichTransformer]
    stats: PipelineStats
    deadLetterQueue: seq[PipelineError]

proc newPipeline(name: string): Pipeline =
  Pipeline(
    name: name,
    filters: @[],
    mappers: @[],
    enrichers: @[],
    stats: PipelineStats(),
    deadLetterQueue: @[]
  )

proc addFilter(p: var Pipeline, field, operator, value: string) =
  p.filters.add(FilterTransformer(field: field, operator: operator, value: value))

proc addMapper(p: var Pipeline, mappings: seq[tuple[srcField, dstField: string, transform: string]]) =
  p.mappers.add(MapTransformer(mappings: mappings))

proc addEnricher(p: var Pipeline, field, targetField: string, lookup: Table[string, JsonNode]) =
  p.enrichers.add(EnrichTransformer(field: field, lookupTable: lookup, targetField: targetField))

proc run(p: var Pipeline, records: seq[DataRecord]): seq[DataRecord] =
  var current = records
  p.stats.extracted = records.len

  # Apply filters
  var filtered: seq[DataRecord]
  for rec in current:
    var keep = true
    for f in p.filters:
      if not applyFilter(f, rec):
        keep = false
        break
    if keep:
      filtered.add(rec)

  p.stats.dropped = current.len - filtered.len
  current = filtered

  # Apply mappers
  for i in 0..<current.len:
    for m in p.mappers:
      applyMap(m, current[i])

  # Apply enrichers
  for i in 0..<current.len:
    for e in p.enrichers:
      applyEnrich(e, current[i])

  p.stats.transformed = current.len
  return current

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Data Pipeline Demo ==="

  # Extract
  let csvExtractor = CsvExtractor(path: "users.csv", delimiter: ',', hasHeader: true)
  let extracted = extractCsv(csvExtractor)
  echo fmt"\nExtracted: {extracted.totalRows} records"

  # Build pipeline
  var pipeline = newPipeline("user_etl")

  # Filter: only records with non-empty age
  pipeline.addFilter("age", "not_empty", "")

  # Filter: only US users
  pipeline.addFilter("country", "eq", "US")

  # Map: normalize email to lowercase
  pipeline.addMapper(@[
    ("email", "email", "lowercase"),
    ("age", "age", "int"),
    ("name", "name", "trim")
  ])

  # Enrich: add country name
  var countryLookup: Table[string, JsonNode]
  countryLookup["US"] = %"United States"
  countryLookup["UK"] = %"United Kingdom"
  countryLookup["TH"] = %"Thailand"
  countryLookup["AU"] = %"Australia"
  pipeline.addEnricher("country", "country_name", countryLookup)

  # Run
  let transformed = pipeline.run(extracted.records)
  echo fmt"Transformed: {transformed.len} records (dropped {pipeline.stats.dropped})"

  # Load
  discard loadToMemory(transformed)

  # Show results
  echo "\nTransformed records:"
  for r in transformed:
    let name = r.data.getOrDefault("name", %"?").getStr()
    let email = r.data.getOrDefault("email", %"?").getStr()
    let countryName = r.data.getOrDefault("country_name", %"?").getStr()
    echo fmt"  {name} | {email} | {countryName}"

  echo fmt"\nPipeline stats:"
  echo fmt"  extracted: {pipeline.stats.extracted}"
  echo fmt"  transformed: {pipeline.stats.transformed}"
  echo fmt"  dropped: {pipeline.stats.dropped}"

demo()
```

---

## Step 722-735: Aggregation Pipeline & Batch Processing

```nim
import asyncdispatch, tables, strformat, times, sequtils, json, strutils, math, algorithm

# ============================
# Aggregation pipeline
# ============================

type
  AggFunc = enum
    afSum, afCount, afAvg, afMin, afMax, afCountDistinct

  AggSpec = object
    field: string
    func_: AggFunc
    alias: string

  GroupByResult = object
    key: string
    count: int
    aggregates: Table[string, float]

proc aggregate(records: seq[DataRecord], groupBy: string,
               specs: seq[AggSpec]): seq[GroupByResult] =
  var groups: Table[string, seq[DataRecord]]

  for r in records:
    let key = if groupBy in r.data: r.data[groupBy].getStr() else: "__null__"
    if key notin groups: groups[key] = @[]
    groups[key].add(r)

  var results: seq[GroupByResult]

  for key, group in groups:
    var aggs: Table[string, float]

    for spec in specs:
      let values: seq[float] = group
        .filterIt(spec.field in it.data)
        .mapIt(block:
          let v = it.data[spec.field]
          case v.kind
          of JFloat: v.getFloat()
          of JInt: float(v.getInt())
          of JString:
            try: parseFloat(v.getStr())
            except: 0.0
          else: 0.0
        )

      let aggValue = case spec.func_
        of afSum: values.foldl(a + b, 0.0)
        of afCount: float(values.len)
        of afAvg: if values.len > 0: values.foldl(a + b, 0.0) / float(values.len) else: 0.0
        of afMin: if values.len > 0: values.min() else: 0.0
        of afMax: if values.len > 0: values.max() else: 0.0
        of afCountDistinct:
          var distinct_: seq[float]
          for v in values:
            if v notin distinct_: distinct_.add(v)
          float(distinct_.len)

      aggs[spec.alias] = aggValue

    results.add(GroupByResult(
      key: key,
      count: group.len,
      aggregates: aggs
    ))

  results.sort(proc(a, b: GroupByResult): int = cmp(b.count, a.count))
  return results

# ============================
# Batch processor
# ============================

type
  BatchConfig = object
    batchSize: int
    maxRetries: int
    retryDelayMs: int
    concurrency: int

  BatchStats = object
    total: int
    processed: int
    failed: int
    retried: int
    durationMs: float

proc processBatch[T](items: seq[T], config: BatchConfig,
                     handler: proc(batch: seq[T]): Future[bool] {.async.}
                    ): Future[BatchStats] {.async.} =
  let start = epochTime()
  var stats = BatchStats(total: items.len)
  var i = 0

  while i < items.len:
    let batchEnd = min(i + config.batchSize, items.len)
    let batch = items[i..<batchEnd]

    var success = false
    for attempt in 1..config.maxRetries:
      try:
        success = await handler(batch)
        if success: break
      except CatchableError as e:
        echo fmt"[Batch] Attempt {attempt} failed: {e.msg}"
        if attempt < config.maxRetries:
          await sleepAsync(config.retryDelayMs * attempt)
          inc stats.retried

    if success:
      stats.processed += batch.len
    else:
      stats.failed += batch.len
      echo fmt"[Batch] Failed to process batch of {batch.len} items"

    i = batchEnd

  stats.durationMs = (epochTime() - start) * 1000
  return stats

# ============================
# Change Data Capture (CDC)
# ============================

type
  ChangeType = enum
    ctInsert, ctUpdate, ctDelete

  ChangeEvent = object
    type_: ChangeType
    table_: string
    recordId: string
    before: Option[DataRecord]
    after: Option[DataRecord]
    timestamp: float
    txId: string

  CdcLog = object
    events: seq[ChangeEvent]
    offset: int

var cdcLog = CdcLog(events: @[], offset: 0)

proc appendChange(log: var CdcLog, event: ChangeEvent) =
  log.events.add(event)

proc consumeChanges(log: var CdcLog, fromOffset: int, limit = 100): seq[ChangeEvent] =
  let endIdx = min(fromOffset + limit, log.events.len)
  if fromOffset >= log.events.len:
    return @[]
  return log.events[fromOffset..<endIdx]

# ============================
# Demo
# ============================

proc demo() {.async.} =
  echo "=== Aggregation & Batch Processing Demo ==="

  # Build sample dataset
  var records: seq[DataRecord]
  let orders = [
    ("user_1", "US", 99.99),
    ("user_2", "UK", 49.50),
    ("user_3", "US", 200.00),
    ("user_1", "US", 75.00),
    ("user_4", "TH", 30.00),
    ("user_2", "UK", 150.00),
    ("user_3", "US", 25.00),
  ]

  for i, (userId, country, amount) in orders:
    var rec = DataRecord(
      id: fmt"order_{i + 1}",
      data: initTable[string, JsonNode](),
      metadata: initTable[string, string](),
      timestamp: epochTime(),
      source: "orders_db"
    )
    rec.data["user_id"] = %userId
    rec.data["country"] = %country
    rec.data["amount"] = %amount
    records.add(rec)

  # Aggregate by country
  echo "\n--- Revenue by country ---"
  let countryAgg = aggregate(records, "country", @[
    AggSpec(field: "amount", func_: afSum, alias: "total_revenue"),
    AggSpec(field: "amount", func_: afAvg, alias: "avg_order"),
    AggSpec(field: "amount", func_: afCount, alias: "order_count"),
  ])

  for g in countryAgg:
    let rev = g.aggregates.getOrDefault("total_revenue", 0.0)
    let avg = g.aggregates.getOrDefault("avg_order", 0.0)
    let cnt = g.aggregates.getOrDefault("order_count", 0.0)
    echo fmt"  {g.key}: ${rev:.2f} total, ${avg:.2f} avg, {int(cnt)} orders"

  # Batch processing
  echo "\n--- Batch Processing ---"
  let config = BatchConfig(
    batchSize: 3,
    maxRetries: 2,
    retryDelayMs: 50,
    concurrency: 1
  )

  var processedCount = 0
  let stats = await processBatch(records, config, proc(batch: seq[DataRecord]): Future[bool] {.async.} =
    await sleepAsync(5)
    processedCount += batch.len
    echo fmt"  Processed batch of {batch.len} records"
    return true
  )

  echo fmt"Batch stats: {stats.processed} processed, {stats.failed} failed in {stats.durationMs:.0f}ms"

  # CDC simulation
  echo "\n--- Change Data Capture ---"
  var r1 = records[0]
  r1.data["amount"] = %110.00

  appendChange(cdcLog, ChangeEvent(
    type_: ctInsert,
    table_: "orders",
    recordId: "order_1",
    before: none(DataRecord),
    after: some(records[0]),
    timestamp: epochTime(),
    txId: "tx_001"
  ))

  appendChange(cdcLog, ChangeEvent(
    type_: ctUpdate,
    table_: "orders",
    recordId: "order_1",
    before: some(records[0]),
    after: some(r1),
    timestamp: epochTime(),
    txId: "tx_002"
  ))

  let changes = consumeChanges(cdcLog, 0)
  echo fmt"CDC events: {changes.len}"
  for c in changes:
    echo fmt"  [{c.type_}] {c.table_}.{c.recordId} @ tx:{c.txId}"

waitFor demo()
```

---

## 📝 สรุป Part 50

| Steps | หัวข้อ |
|-------|--------|
| 721 | ETL pipeline: extract (CSV/JSON), filter, map, enrich, load |
| 722-735 | Aggregation, batch processor with retry, Change Data Capture |

---

**← [Part 49: Service Mesh](part_49_service_mesh.md) | [Part 51: Config Management →](part_51_config_management.md)**
