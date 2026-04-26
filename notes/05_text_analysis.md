# Text Analysis

## Pipeline

```
char filter (O-n) → tokenizer (1) → token filter (0-n)
```

## Test an analyzer

```json
POST index_name/_analyze
{
  "field": "body",
  "text": "my text"
}

POST _analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase"],
  "text": "my text"
}
```

## Built-in analyzers

| Analyzer | Behaviour |
|---|---|
| `standard` | splits on whitespace/punctuation, lowercase |
| `whitespace` | splits on whitespace only, keeps punctuation |
| `english` | standard + stemming + stop words |
| `pattern` | splits on regex pattern |
| `fingerprint` | deduplicates tokens, good for clustering |

## Custom analyzer

```json
"analysis": {
  "char_filter": {
    "my_char_filter": {
      "type": "mapping",
      "mappings": ["α => alpha", "β => beta"]
    }
  },
  "tokenizer": {
    "my_tokenizer": {
      "type": "pattern",
      "pattern": "[-]"
    }
  },
  "filter": {
    "my_filter": {
      "type": "edge_ngram",
      "min_gram": 1,
      "max_gram": 10
    }
  },
  "analyzer": {
    "my_analyzer": {
      "type": "custom",
      "char_filter": ["my_char_filter"],
      "tokenizer": "my_tokenizer",
      "filter": ["lowercase", "my_filter"]
    }
  }
}
```

## Extending a built-in analyzer

Elasticsearch has no inheritance — you rebuild the built-in analyzer from its components and add your own filters. Here's the `english` analyzer extended with a custom stopword list and synonym filter:

```json
PUT my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "my_stop": {
          "type": "stop",
          "stopwords": ["the", "a", "an", "this", "is"]
        },
        "my_synonyms": {
          "type": "synonym_graph",
          "synonyms_path": "synonyms.txt",
          "updateable": true
        },
        "english_keywords": {
          "type": "keyword_marker",
          "keywords": ["elastic", "kibana"]
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "english_possessive_stemmer": {
          "type": "stemmer",
          "language": "possessive_english"
        }
      },
      "analyzer": {
        "my_english": {
          "tokenizer": "standard",
          "filter": [
            "english_possessive_stemmer",
            "lowercase",
            "my_stop",
            "english_keywords",
            "english_stemmer",
            "my_synonyms"
          ]
        }
      }
    }
  }
}
```

The built-in `english` analyzer uses: possessive stemmer → lowercase → stop → keyword_marker → stemmer. Rebuilding it explicitly gives you control over each step.

> Synonyms must go **after** the stemmer so synonym tokens are in the same form as indexed tokens.

## Common token filters

| Filter | Behaviour |
|---|---|
| `lowercase` | lowercases all tokens |
| `stop` | removes stop words |
| `stemmer` | reduces words to root form |
| `asciifolding` | é→e, ü→u, ñ→n |
| `synonym_graph` | applies synonyms |
| `edge_ngram` | prefix tokens for autocomplete |
| `shingle` | multi-word tokens, proximity scoring |
| `keep` | only keep specified tokens |

## Normalizer

Like an analyzer but for `keyword` fields — no tokenization:

```json
PUT my_index
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": {
          "type": "custom",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "city": {
        "type": "keyword",
        "normalizer": "lowercase_normalizer"
      }
    }
  }
}
```

## Index vs search analyzer

```json
"body": {
  "type": "text",
  "analyzer": "my_index_analyzer",
  "search_analyzer": "my_search_analyzer"
}
```

## Synonyms

```json
"my_synonym_filter": {
  "type": "synonym_graph",
  "synonyms_path": "synonyms.txt",
  "updateable": true   // search time only
}
```

Reload without restart (search time analyzers only):

```
POST index_name/_reload_search_analyzers
```

Synonyms file format:

```
# explicit one-way
dns => domain name service

# equivalent (two-way)
sneakers, trainers, tennis shoes
```

**Synonym strategy:**

- Index time → hyponyms (snowshoes => shoes), narrow to broad
- Query time → true synonyms (equivalent terms)
- Avoid hypernyms at query time — too many tokens

## Autocomplete options

| Type | How | Matches | Speed |
|---|---|---|---|
| `search_as_you_type` | field type, edge ngrams + shingles | mid-string | good |
| `completion` | field type, in-memory FST | prefix only | fastest |
| `edge_ngram` filter | custom analyzer | prefix only | good |

## Shingles

- Multi-word tokens: `"quick brown"`, `"brown fox"`
- Improves proximity scoring without phrase queries
- Use on short fields only (title) — expensive on long fields
- Use `output_unigrams: false` on a dedicated shingle field to avoid double-counting
