# 14. Search Systems

> Status: **Done** — concise notes for all sub-topics below.

[← Back to master index](../README.md)

---

## At a glance

```mermaid
flowchart LR
    Docs[Documents] --> Index[Inverted Index]
    Index --> Query[Search Query]
    Query --> Rank[Ranked Results]
```

---

## Sub-topics

| # | Sub-topic |
|---|-----------|
| 14.1 | [Full Text Search](#141-full-text-search) |
| 14.2 | [Inverted Index](#142-inverted-index) |
| 14.3 | [Elasticsearch](#143-elasticsearch) |
| 14.4 | [Lucene](#144-lucene) |
| 14.5 | [Ranking](#145-ranking) |
| 14.6 | [Relevance Scoring](#146-relevance-scoring) |
| 14.7 | [Faceted Search](#147-faceted-search) |
| 14.8 | [Autocomplete](#148-autocomplete) |
| 14.9 | [Fuzzy Search](#149-fuzzy-search) |

---

## 14.1 Full Text Search

**Summary:** Finding documents containing words or phrases from natural-language queries — not just exact key lookups.

- Tokenizes text, applies stemming/stop words, matches against an index
- Supports boolean queries, phrases, wildcards, and relevance ranking
- Complements primary DB — search engines optimize read patterns DBs handle poorly

**References:** _None yet._

---

## 14.2 Inverted Index

**Summary:** Core search structure mapping each term → list of documents (postings) containing it.

- Query "quick fox" → intersect postings lists for "quick" and "fox"
- Each posting stores doc ID, term frequency, and positions for phrase/proximity
- Built at index time; updates require reindex or near-real-time segments

**References:** _None yet._

---

## 14.3 Elasticsearch

**Summary:** Distributed search and analytics engine built on Lucene — REST API, JSON docs, horizontal scale.

- Index → shards → replicas; cluster coordination via elected master
- Near-real-time: documents searchable ~1s after index request
- Use for log analytics (ELK), product search, and full-text over JSON documents

**References:** _None yet._

---

## 14.4 Lucene

**Summary:** Java search library powering Elasticsearch, Solr — segments, analyzers, and scoring at the core.

- Immutable segments merged in background; deletes marked, compacted later
- Analyzers chain tokenizers + filters (lowercase, stem, synonym)
- Understanding Lucene explains ES shard behavior and tuning knobs

**References:** _None yet._

---

## 14.5 Ranking

**Summary:** Ordering search results so the most useful documents appear first.

- Combines textual relevance, business signals (popularity, recency), and personalization
- Learning-to-rank models refine order from click-through data
- Balance precision (top results correct) vs recall (find all relevant docs)

**References:** _None yet._

---

## 14.6 Relevance Scoring

**Summary:** Numerical score measuring how well a document matches a query — classic: TF-IDF, BM25.

- **TF-IDF:** term frequency × inverse document frequency — rare terms weigh more
- **BM25:** probabilistic improvement over TF-IDF; default in Lucene/ES
- Field boosts (`title^2`) and function scores adjust ranking at query time

**References:** _None yet._

---

## 14.7 Faceted Search

**Summary:** Aggregated filters (facets) alongside results — e.g., brand, price range, category counts.

- Terms aggregation buckets documents by field value with counts
- Users narrow results without rewriting query — common in e-commerce
- Requires keyword or properly mapped fields for efficient aggregation

**References:** _None yet._

---

## 14.8 Autocomplete

**Summary:** Suggest completions as the user types — prefix matching on indexed terms.

- Edge n-grams or completion suggester index prefixes at index time
- `search_as_you_type` field type in Elasticsearch
- Rank suggestions by popularity, recency, or personalized history

**References:** _None yet._

---

## 14.9 Fuzzy Search

**Summary:** Tolerates typos and spelling variations via edit distance (Levenshtein).

- `fuzziness: AUTO` in Elasticsearch matches within 1–2 edits depending on term length
- Trade-off: more typos caught vs slower queries and false matches
- Combine with boosting exact matches above fuzzy ones

**References:** _None yet._

---

## Quick Reference

| Sub-topic | Core idea | Typical tool |
|-----------|-----------|--------------|
| **Full Text Search** | Natural language lookup | ES, Solr, OpenSearch |
| **Inverted Index** | term → documents | Lucene segments |
| **Elasticsearch** | Distributed search cluster | REST + JSON |
| **Lucene** | Search library core | Analyzers, BM25 |
| **Ranking / Scoring** | Order by relevance | BM25 + boosts |
| **Faceted Search** | Filter + count buckets | Aggregations |
| **Autocomplete** | Prefix suggestions | Edge n-grams |
| **Fuzzy Search** | Typo tolerance | Edit distance |

---

[← Back to master index](../README.md)
