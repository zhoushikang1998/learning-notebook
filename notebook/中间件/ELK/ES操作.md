```nginx
GET /_cat/indices?v


GET /bank/_search
{
  "query": {"match_all": {}},
  "sort": [
    {
      "account_number": "asc"
    }
  ],
  "from": 10,
  "size": 10
}

GET /bank/_search
{
  "query": {"match": {"address": "mill lane"}}
}

GET /bank/_search
{
  "query": {"match_phrase": {"address": "mill lane"}}
}

GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "age": "40"
        }}
      ],
      "must_not": [
        {
          "match": {
            "state": "ID"
          }
        }
      ]
    }
  }
}

GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": [
        {
          "range": {
            "balance": {
              "gte": 20000,
              "lte": 30000
            }
          }
        }
      ]
    }
  }
}

GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "avarage_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}

GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

