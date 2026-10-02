# Part 42: Search Engine
## Steps 601-615: Full-Text Search ใน Nim

---

## 🎯 เป้าหมายของ Part นี้

- Inverted index
- TF-IDF ranking
- Fuzzy search (Levenshtein)
- Search with filters
- Autocomplete / prefix search
- Highlight matching terms

---

## Step 601: Inverted Index

```nim
import tables, strutils, sequtils, algorithm, math, strformat

# ============================
# Tokenizer
# ============================

proc tokenize(text: string): seq[string] =
  ## Split text into lowercase word tokens, remove punctuation
  var tokens: seq[string]
  var current = ""
  
  for ch in text.toLower():
    if ch.isAlphaNumeric():
      current &= ch
    elif current.len > 0:
      if current.len >= 2:  # skip single chars
        tokens.add(current)
      current = ""
  
  if current.len >= 2:
    tokens.add(current)
  
  return tokens

proc stem(word: string): string =
  ## Very simple English stemmer (Porter-lite)
  var w = word
  if w.endsWith("ing") and w.len > 5: w = w[0..^4]
  elif w.endsWith("tion") and w.len > 6: w = w[0..^5]
  elif w.endsWith("tions") and w.len > 7: w = w[0..^6]
  elif w.endsWith("ness") and w.len > 6: w = w[0..^5]
  elif w.endsWith("ment") and w.len > 6: w = w[0..^5]
  elif w.endsWith("ers") and w.len > 5: w = w[0..^4]
  elif w.endsWith("es") and w.len > 4: w = w[0..^3]
  elif w.endsWith("ed") and w.len > 4: w = w[0..^3]
  elif w.endsWith("s") and w.len > 3: w = w[0..^2]
  return w

# Stop words
const stopWords = ["the", "a", "an", "and", "or", "but", "in", "on",
                   "at", "to", "for", "of", "with", "by", "from",
                   "is", "are", "was", "were", "be", "been", "being",
                   "have", "has", "had", "do", "does", "did",
                   "will", "would", "could", "should", "may", "might",
                   "this", "that", "these", "those", "it", "its"].toHashSet()

proc analyze(text: string): seq[string] =
  tokenize(text)
    .filterIt(it notin stopWords)
    .mapIt(stem(it))

# ============================
# Inverted Index
# ============================

type
  Posting = object
    docId: int
    frequency: int
    positions: seq[int]   # word positions in document

  InvertedIndex = object
    index: Table[string, seq[Posting]]
    docCount: int

proc newIndex(): InvertedIndex =
  InvertedIndex(index: initTable[string, seq[Posting]](), docCount: 0)

proc addDocument(idx: var InvertedIndex, docId: int, text: string) =
  let tokens = analyze(text)
  inc idx.docCount
  
  # Count frequencies and positions
  var termInfo: Table[string, tuple[freq: int, positions: seq[int]]]
  
  for i, token in tokens:
    if token notin termInfo:
      termInfo[token] = (0, @[])
    termInfo[token].freq += 1
    termInfo[token].positions.add(i)
  
  for term, info in termInfo:
    if term notin idx.index:
      idx.index[term] = @[]
    idx.index[term].add(Posting(
      docId: docId,
      frequency: info.freq,
      positions: info.positions
    ))

proc getPostings(idx: InvertedIndex, term: string): seq[Posting] =
  let stemmed = stem(term.toLower())
  if stemmed in idx.index:
    return idx.index[stemmed]
  return @[]

proc vocabSize(idx: InvertedIndex): int = idx.index.len

# ============================
# TF-IDF Scoring
# ============================

proc tf(freq: int, docTermCount: int): float =
  ## Term frequency (normalized)
  if docTermCount == 0: return 0.0
  return float(freq) / float(docTermCount)

proc idf(numDocs: int, docsWithTerm: int): float =
  ## Inverse document frequency
  if docsWithTerm == 0: return 0.0
  return ln(float(numDocs) / float(docsWithTerm) + 1.0)

proc tfIdf(termFreq, docLen, numDocs, docsWithTerm: int): float =
  tf(termFreq, docLen) * idf(numDocs, docsWithTerm)

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Inverted Index Demo ==="
  
  var idx = newIndex()
  
  let docs = [
    (1, "Nim is a fast compiled language with Python-like syntax"),
    (2, "Python is a popular interpreted programming language"),
    (3, "Go is a compiled language developed by Google"),
    (4, "Rust is a systems programming language focused on safety"),
    (5, "Nim compiles to C and JavaScript for fast performance"),
  ]
  
  for (id, text) in docs:
    idx.addDocument(id, text)
    echo fmt"Indexed doc {id}: {text[0..min(40, text.len-1)]}..."
  
  echo fmt"\nVocabulary size: {idx.vocabSize()}"
  
  echo "\nSearch 'nim fast':"
  for term in ["nim", "fast"]:
    let postings = idx.getPostings(term)
    echo fmt"  '{term}' -> {postings.len} docs: {postings.mapIt(it.docId)}"
  
  echo "\nTF-IDF scores for 'fast':"
  let fastPostings = idx.getPostings("fast")
  for p in fastPostings:
    let score = tfIdf(p.frequency, 10, idx.docCount, fastPostings.len)
    echo fmt"  doc {p.docId}: {score:.4f}"
  
  echo "\nAnalysis:"
  echo fmt"  analyze('Programming languages') = {analyze(\"Programming languages\")}"
  echo fmt"  stem('compiled') = {stem(\"compiled\")}"
  echo fmt"  stem('programming') = {stem(\"programming\")}"

demo()
```

---

## Step 602: Full Search Engine

```nim
import tables, strutils, sequtils, algorithm, math, strformat, options

# ============================
# Document store
# ============================

type
  Field = object
    name: string
    value: string
    boost: float   # field weight multiplier

  Document = object
    id: int
    fields: seq[Field]
    metadata: Table[string, string]

  SearchHit = object
    docId: int
    score: float
    highlights: Table[string, string]
    metadata: Table[string, string]

  SearchResult = object
    hits: seq[SearchHit]
    total: int
    took: float   # milliseconds

# ============================
# Fuzzy matching (Levenshtein)
# ============================

proc editDistance(s, t: string): int =
  ## Levenshtein distance between two strings
  let m = s.len
  let n = t.len
  
  var dp = newSeqWith(m + 1, newSeq[int](n + 1))
  
  for i in 0..m: dp[i][0] = i
  for j in 0..n: dp[0][j] = j
  
  for i in 1..m:
    for j in 1..n:
      if s[i-1] == t[j-1]:
        dp[i][j] = dp[i-1][j-1]
      else:
        dp[i][j] = 1 + min(dp[i-1][j],    # delete
                       min(dp[i][j-1],      # insert
                           dp[i-1][j-1]))   # replace
  
  return dp[m][n]

proc fuzzyMatch(term, candidate: string, maxDist = 2): bool =
  if abs(term.len - candidate.len) > maxDist: return false
  editDistance(term, candidate) <= maxDist

# ============================
# Highlight
# ============================

proc highlightText(text: string, terms: seq[string],
                   preTag = "<em>", postTag = "</em>"): string =
  var result = text
  
  for term in terms:
    # Simple case-insensitive highlight
    let lower = result.toLower()
    let pos = lower.find(term.toLower())
    if pos >= 0:
      let matchedWord = result[pos..pos + term.len - 1]
      result = result[0..<pos] & preTag & matchedWord & postTag &
               result[pos + term.len..^1]
  
  return result

# ============================
# Main search engine
# ============================

type
  Engine = object
    documents: Table[int, Document]
    index: Table[string, seq[tuple[docId: int, freq: int]]]
    docCounter: int

proc newEngine(): Engine =
  Engine(
    documents: initTable[int, Document](),
    index: initTable[string, seq[tuple[docId: int, freq: int]]](),
    docCounter: 0
  )

proc tokenizeSimple(text: string): seq[string] =
  text.toLower().split({' ', ',', '.', '!', '?', ';', ':', '"', '\''})
    .filterIt(it.len >= 2)

proc addDoc(engine: var Engine, fields: seq[Field],
            metadata: Table[string, string] = initTable[string, string]()): int =
  inc engine.docCounter
  let docId = engine.docCounter
  
  engine.documents[docId] = Document(id: docId, fields: fields, metadata: metadata)
  
  # Index each field
  var termFreqs: Table[string, int]
  
  for field in fields:
    let tokens = tokenizeSimple(field.value)
    for token in tokens:
      termFreqs[token] = termFreqs.getOrDefault(token, 0) + int(field.boost)
  
  for term, freq in termFreqs:
    if term notin engine.index:
      engine.index[term] = @[]
    engine.index[term].add((docId, freq))
  
  return docId

proc search(engine: Engine, query: string,
            limit = 10, offset = 0,
            fuzzy = false): SearchResult =
  let start = epochTime()
  let queryTerms = tokenizeSimple(query)
  
  if queryTerms.len == 0:
    return SearchResult(hits: @[], total: 0, took: 0.0)
  
  # Score each document
  var scores: Table[int, float]
  
  for term in queryTerms:
    var matchedPostings: seq[tuple[docId: int, freq: int]]
    
    # Exact match
    if term in engine.index:
      matchedPostings = engine.index[term]
    elif fuzzy:
      # Fuzzy match: find similar terms in index
      for indexTerm, postings in engine.index:
        if fuzzyMatch(term, indexTerm, maxDist = 2):
          matchedPostings.add(postings)
    
    let docsWithTerm = matchedPostings.len
    let numDocs = engine.documents.len
    
    for (docId, freq) in matchedPostings:
      let idfScore = if docsWithTerm > 0:
        ln(float(numDocs) / float(docsWithTerm) + 1.0)
      else: 0.0
      
      scores[docId] = scores.getOrDefault(docId, 0.0) +
                      (float(freq) * idfScore)
  
  # Sort by score
  var ranked: seq[tuple[docId: int, score: float]]
  for docId, score in scores:
    ranked.add((docId, score))
  ranked.sort(proc(a, b: tuple[docId: int, score: float]): int =
    cmp(b.score, a.score))  # descending
  
  let total = ranked.len
  let page = ranked[offset..min(offset + limit - 1, ranked.len - 1)]
  
  var hits: seq[SearchHit]
  for (docId, score) in page:
    let doc = engine.documents[docId]
    var highlights: Table[string, string]
    
    for field in doc.fields:
      let highlighted = highlightText(field.value, queryTerms)
      if highlighted != field.value:
        highlights[field.name] = highlighted
    
    hits.add(SearchHit(
      docId: docId,
      score: score,
      highlights: highlights,
      metadata: doc.metadata
    ))
  
  let took = (epochTime() - start) * 1000.0
  return SearchResult(hits: hits, total: total, took: took)

# ============================
# Autocomplete (prefix trie)
# ============================

type
  TrieNode = object
    children: Table[char, ref TrieNode]
    isEnd: bool
    count: int     # how many times this completion was searched

  Trie = object
    root: ref TrieNode

proc newTrie(): Trie =
  Trie(root: new TrieNode)

proc insert(trie: var Trie, word: string) =
  var node = trie.root
  for ch in word.toLower():
    if ch notin node.children:
      node.children[ch] = new TrieNode
    node = node.children[ch]
  node.isEnd = true
  inc node.count

proc search(trie: Trie, prefix: string): seq[string] =
  var node = trie.root
  let lower = prefix.toLower()
  
  # Navigate to prefix
  for ch in lower:
    if ch notin node.children:
      return @[]
    node = node.children[ch]
  
  # BFS to collect completions
  var results: seq[string]
  var stack: seq[tuple[n: ref TrieNode, word: string]]
  stack.add((node, lower))
  
  while stack.len > 0:
    let (curr, word) = stack.pop()
    if curr.isEnd:
      results.add(word)
    for ch, child in curr.children:
      if results.len >= 10: break
      stack.add((child, word & ch))
  
  return results

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Search Engine Demo ==="
  
  var engine = newEngine()
  
  # Index documents
  let docs = [
    (@[
      Field(name: "title", value: "Introduction to Nim Programming", boost: 2.0),
      Field(name: "body", value: "Nim is a compiled language with great performance", boost: 1.0),
    ], {"category": "tutorial", "author": "Alice"}.toTable()),
    (@[
      Field(name: "title", value: "Python vs Nim Performance", boost: 2.0),
      Field(name: "body", value: "Comparing Python and Nim speed benchmarks", boost: 1.0),
    ], {"category": "benchmarks", "author": "Bob"}.toTable()),
    (@[
      Field(name: "title", value: "Building REST APIs with Nim", boost: 2.0),
      Field(name: "body", value: "Creating high performance web APIs using Nim backend framework", boost: 1.0),
    ], {"category": "tutorial", "author": "Alice"}.toTable()),
    (@[
      Field(name: "title", value: "Nim Memory Management Guide", boost: 2.0),
      Field(name: "body", value: "Understanding ARC ORC and garbage collection in Nim", boost: 1.0),
    ], {"category": "advanced", "author": "Charlie"}.toTable()),
  ]
  
  for (fields, metadata) in docs:
    let id = engine.addDoc(fields, metadata)
    echo fmt"Indexed doc {id}: {fields[0].value}"
  
  echo "\n--- Search 'nim performance' ---"
  let r1 = engine.search("nim performance")
  echo fmt"Found {r1.total} results in {r1.took:.2f}ms:"
  for hit in r1.hits:
    echo fmt"  Doc {hit.docId} (score: {hit.score:.3f})"
    for field, text in hit.highlights:
      echo fmt"    [{field}]: {text}"
  
  echo "\n--- Fuzzy search 'perfomance' (typo) ---"
  let r2 = engine.search("perfomance", fuzzy = true)
  echo fmt"Found {r2.total} results"
  
  echo "\n--- Autocomplete ---"
  var trie = newTrie()
  for word in ["nim", "nimble", "nimbly", "node", "nodejs", "numpy", "next"]:
    trie.insert(word)
  
  echo fmt"Completions for 'ni': {trie.search(\"ni\")}"
  echo fmt"Completions for 'no': {trie.search(\"no\")}"

demo()
```

---

## Step 603-615: Search Filters and Facets

```nim
import tables, strutils, sequtils, algorithm, math, strformat, times, json

# ============================
# Faceted search
# ============================

type
  FilterOp = enum
    foEq, foGt, foGte, foLt, foLte, foIn, foRange

  Filter = object
    field: string
    op: FilterOp
    value: string
    values: seq[string]  # for foIn

  FacetResult = object
    value: string
    count: int

  SearchFilters = object
    query: string
    filters: seq[Filter]
    sortBy: string
    sortAsc: bool
    page: int
    pageSize: int

  SearchDoc = object
    id: int
    title: string
    price: float
    category: string
    brand: string
    rating: float
    tags: seq[string]
    inStock: bool
    createdAt: float

var productDb: seq[SearchDoc] = @[
  SearchDoc(id: 1, title: "Nim Handbook", price: 29.99, category: "books",
            brand: "NimPress", rating: 4.8, tags: @["programming", "nim"],
            inStock: true, createdAt: epochTime() - 86400),
  SearchDoc(id: 2, title: "Go Programming", price: 34.99, category: "books",
            brand: "GoPress", rating: 4.5, tags: @["programming", "go"],
            inStock: true, createdAt: epochTime() - 86400 * 5),
  SearchDoc(id: 3, title: "Mechanical Keyboard", price: 149.99, category: "hardware",
            brand: "KeyCo", rating: 4.2, tags: @["keyboard", "tech"],
            inStock: false, createdAt: epochTime() - 86400 * 2),
  SearchDoc(id: 4, title: "Python Crash Course", price: 24.99, category: "books",
            brand: "NimPress", rating: 4.6, tags: @["programming", "python"],
            inStock: true, createdAt: epochTime()),
  SearchDoc(id: 5, title: "USB Hub 7-port", price: 39.99, category: "hardware",
            brand: "TechCo", rating: 3.9, tags: @["usb", "tech"],
            inStock: true, createdAt: epochTime() - 86400 * 10),
]

proc matchesFilter(doc: SearchDoc, f: Filter): bool =
  case f.field
  of "category":
    case f.op
    of foEq: return doc.category == f.value
    of foIn: return doc.category in f.values
    else: return true
  of "brand":
    case f.op
    of foEq: return doc.brand == f.value
    of foIn: return doc.brand in f.values
    else: return true
  of "price":
    let price = f.value.parseFloat()
    case f.op
    of foGt: return doc.price > price
    of foGte: return doc.price >= price
    of foLt: return doc.price < price
    of foLte: return doc.price <= price
    else: return true
  of "rating":
    let rating = f.value.parseFloat()
    case f.op
    of foGte: return doc.rating >= rating
    else: return true
  of "in_stock":
    return doc.inStock == (f.value == "true")
  else:
    return true

proc filterAndSearch(params: SearchFilters): tuple[docs: seq[SearchDoc], total: int] =
  var results = productDb
  
  # Text search
  if params.query.len > 0:
    let q = params.query.toLower()
    results = results.filterIt(
      it.title.toLower().contains(q) or
      it.category.toLower().contains(q) or
      it.tags.anyIt(it.contains(q))
    )
  
  # Apply filters
  for f in params.filters:
    results = results.filterIt(it.matchesFilter(f))
  
  let total = results.len
  
  # Sort
  if params.sortBy == "price":
    if params.sortAsc:
      results.sort(proc(a, b: SearchDoc): int = cmp(a.price, b.price))
    else:
      results.sort(proc(a, b: SearchDoc): int = cmp(b.price, a.price))
  elif params.sortBy == "rating":
    results.sort(proc(a, b: SearchDoc): int = cmp(b.rating, a.rating))
  elif params.sortBy == "newest":
    results.sort(proc(a, b: SearchDoc): int = cmp(b.createdAt, a.createdAt))
  
  # Paginate
  let start = (params.page - 1) * params.pageSize
  let end_ = min(start + params.pageSize, results.len)
  
  if start >= results.len:
    return (@[], total)
  
  return (results[start..<end_], total)

proc getFacets(docs: seq[SearchDoc]): Table[string, seq[FacetResult]] =
  var catCounts: Table[string, int]
  var brandCounts: Table[string, int]
  var inStockCount = 0
  
  for doc in docs:
    catCounts[doc.category] = catCounts.getOrDefault(doc.category, 0) + 1
    brandCounts[doc.brand] = brandCounts.getOrDefault(doc.brand, 0) + 1
    if doc.inStock: inc inStockCount
  
  var result: Table[string, seq[FacetResult]]
  
  var catFacets: seq[FacetResult]
  for cat, count in catCounts:
    catFacets.add(FacetResult(value: cat, count: count))
  catFacets.sort(proc(a, b: FacetResult): int = cmp(b.count, a.count))
  result["category"] = catFacets
  
  var brandFacets: seq[FacetResult]
  for brand, count in brandCounts:
    brandFacets.add(FacetResult(value: brand, count: count))
  brandFacets.sort(proc(a, b: FacetResult): int = cmp(b.count, a.count))
  result["brand"] = brandFacets
  
  result["in_stock"] = @[FacetResult(value: "true", count: inStockCount),
                          FacetResult(value: "false", count: docs.len - inStockCount)]
  
  return result

# ============================
# Demo
# ============================

proc demo() =
  echo "=== Faceted Search Demo ==="
  
  # Search with filters
  let params = SearchFilters(
    query: "programming",
    filters: @[
      Filter(field: "in_stock", op: foEq, value: "true"),
      Filter(field: "price", op: foLte, value: "35.0"),
    ],
    sortBy: "rating",
    sortAsc: false,
    page: 1,
    pageSize: 10
  )
  
  let (results, total) = filterAndSearch(params)
  echo fmt"\nQuery: '{params.query}'"
  echo fmt"Filters: in_stock=true, price<=35"
  echo fmt"Results: {results.len}/{total}"
  
  for doc in results:
    echo fmt"  [{doc.id}] {doc.title} - ${doc.price:.2f} ★{doc.rating:.1f}"
  
  # Facets on all programming books
  let allProgParams = SearchFilters(
    query: "programming", filters: @[],
    sortBy: "", sortAsc: true, page: 1, pageSize: 100
  )
  let (allResults, _) = filterAndSearch(allProgParams)
  let facets = getFacets(allResults)
  
  echo "\nFacets:"
  for facetName, facetValues in facets:
    echo fmt"  {facetName}:"
    for f in facetValues:
      echo fmt"    {f.value}: {f.count}"

demo()
```

---

## 📝 สรุป Part 42

| Steps | หัวข้อ |
|-------|--------|
| 601 | Inverted index, TF-IDF, tokenizer, stemmer |
| 602 | Full search engine with highlights + autocomplete |
| 603-615 | Faceted search, filters, pagination |

---

**← [Part 41: Rate Limiting](part_41_rate_limiting.md) | [Part 43: Email System →](part_43_email.md)**
