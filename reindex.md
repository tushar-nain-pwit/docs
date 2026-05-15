# Elasticsearch Reindex Plan — Fix Money Field Precision

## Summary

All 10 money/cost/rate fields across your 5 indexes are mapped incorrectly (`long` or `float` instead of `double`), causing precision loss in queries/aggregations. However, **the original decimal values are preserved in `_source`**, so the Reindex API can fully recover them without touching PostgreSQL.

## Strategy

1. Create new indexes (`*_v2`) with corrected mappings
2. Reindex from old → new (reads `_source`, re-parses with correct types)
3. Verify data integrity
4. Swap aliases (zero-downtime cutover)
5. Drop old indexes after grace period

---

## Step 1: Create new indexes with corrected mappings

Run these curl commands to create the `*_v2` indexes. I've **only changed the money fields to `double`** — everything else stays the same.

**Replace `localhost:9200` with your actual Elasticsearch host:port.**

### 1a. `account_transactions_v2`

```bash
curl -X PUT "localhost:9200/account_transactions_v2?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "account_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "account_payment_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "channel_type_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "country_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "cr_amount": { "type": "double" },
      "created_at": { "type": "date" },
      "currency_id": { "type": "long" },
      "date_key": { "type": "date" },
      "description": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "dr_amount": { "type": "double" },
      "flag": { "type": "boolean" },
      "id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "item_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "month": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "product_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "transaction_date": { "type": "date" },
      "transaction_type": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "transaction_type_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "updated_at": { "type": "date" },
      "user_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "year": { "type": "long" }
    }
  }
}
'
```

### 1b. `call_units_v2`

```bash
curl -X PUT "localhost:9200/call_units_v2?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "account_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "call_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "call_leg_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "campaign_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "channel_type_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "cost": { "type": "double" },
      "country_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "duration": { "type": "double" },
      "item_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "organization_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "processed_at": { "type": "date" },
      "pulse": { "type": "long" },
      "rate": { "type": "double" },
      "server_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "type": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "unit_minutes": { "type": "double" },
      "units": { "type": "long" },
      "usecasemodel_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      }
    }
  }
}
'
```

### 1c. `session_units_v2`

```bash
curl -X PUT "localhost:9200/session_units_v2?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "account_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "campaign_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "channel_type_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "cost": { "type": "double" },
      "country_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "item_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "medium": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "organization_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "processed_at": { "type": "date" },
      "rate": { "type": "double" },
      "session_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "total_messages": { "type": "long" },
      "units": { "type": "long" },
      "usecasemodel_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      }
    }
  }
}
'
```

### 1d. `template_units_v2`

```bash
curl -X PUT "localhost:9200/template_units_v2?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "account_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "campaign_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "channel_type_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "cost": { "type": "double" },
      "country_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "item_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "medium": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "organization_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "processed_at": { "type": "date" },
      "rate": { "type": "double" },
      "template_category": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "units": { "type": "long" },
      "usecasemodel_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      }
    }
  }
}
'
```

### 1e. `organization_transactions_v2`

```bash
curl -X PUT "localhost:9200/organization_transactions_v2?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "channel_type_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "country_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "cr_amount": { "type": "double" },
      "created_at": { "type": "date" },
      "currency_id": { "type": "long" },
      "date_key": { "type": "date" },
      "description": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "dr_amount": { "type": "double" },
      "flag": { "type": "boolean" },
      "id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "item_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "month": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "organization_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "organization_payment_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "transaction_date": { "type": "date" },
      "transaction_type": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "updated_at": { "type": "date" },
      "user_id": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword", "ignore_above": 256 }}
      },
      "year": { "type": "long" }
    }
  }
}
'
```

---

## Step 2: Reindex from old → new

Run these curl commands. **Adjust `wait_for_completion` based on your dataset size** — for large indexes (millions of docs), set it to `false` and monitor via the Tasks API.

### 2a. Reindex `account_transactions`

```bash
curl -X POST "localhost:9200/_reindex?wait_for_completion=true&pretty" -H 'Content-Type: application/json' -d'
{
  "source": { "index": "account_transactions" },
  "dest": { "index": "account_transactions_v2" }
}
'
```

### 2b. Reindex `call_units`

```bash
curl -X POST "localhost:9200/_reindex?wait_for_completion=true&pretty" -H 'Content-Type: application/json' -d'
{
  "source": { "index": "call_units" },
  "dest": { "index": "call_units_v2" }
}
'
```

### 2c. Reindex `session_units`

```bash
curl -X POST "localhost:9200/_reindex?wait_for_completion=true&pretty" -H 'Content-Type: application/json' -d'
{
  "source": { "index": "session_units" },
  "dest": { "index": "session_units_v2" }
}
'
```

### 2d. Reindex `template_units`

```bash
curl -X POST "localhost:9200/_reindex?wait_for_completion=true&pretty" -H 'Content-Type: application/json' -d'
{
  "source": { "index": "template_units" },
  "dest": { "index": "template_units_v2" }
}
'
```

### 2e. Reindex `organization_transactions`

```bash
curl -X POST "localhost:9200/_reindex?wait_for_completion=true&pretty" -H 'Content-Type: application/json' -d'
{
  "source": { "index": "organization_transactions" },
  "dest": { "index": "organization_transactions_v2" }
}
'
```

**For very large indexes:** If reindex takes >30 minutes, use `wait_for_completion=false` and poll:
```bash
curl -X GET "localhost:9200/_tasks?actions=*reindex&detailed&pretty"
```

---

## Step 3: Verify data integrity

For each index, confirm:
1. **Document counts match**
2. **Aggregations now work correctly** (test a few SUM/AVG on the money fields)
3. **Spot-check a few records** to ensure decimals are indexed

### 3a. Count check

```bash
curl -X GET "localhost:9200/account_transactions/_count?pretty"
curl -X GET "localhost:9200/account_transactions_v2/_count?pretty"

curl -X GET "localhost:9200/call_units/_count?pretty"
curl -X GET "localhost:9200/call_units_v2/_count?pretty"

curl -X GET "localhost:9200/session_units/_count?pretty"
curl -X GET "localhost:9200/session_units_v2/_count?pretty"

curl -X GET "localhost:9200/template_units/_count?pretty"
curl -X GET "localhost:9200/template_units_v2/_count?pretty"

curl -X GET "localhost:9200/organization_transactions/_count?pretty"
curl -X GET "localhost:9200/organization_transactions_v2/_count?pretty"
```

Counts should be identical for each pair.

### 3b. Aggregation check (before/after)

**Old index (wrong totals):**
```bash
curl -X GET "localhost:9200/call_units/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "total_cost": { "sum": { "field": "cost" }},
    "total_rate": { "sum": { "field": "rate" }}
  }
}
'
```

**New index (should match PostgreSQL):**
```bash
curl -X GET "localhost:9200/call_units_v2/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "total_cost": { "sum": { "field": "cost" }},
    "total_rate": { "sum": { "field": "rate" }}
  }
}
'
```

Compare to PostgreSQL:
```sql
SELECT SUM(cost), SUM(rate) FROM call_units;
```

The `_v2` index sum should now match PG. The old index will be systematically lower.

### 3c. Spot-check a decimal value

Pick a record from `_source` that you know has decimals (e.g. `cost: 1.88`):

```bash
curl -X GET "localhost:9200/call_units_v2/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "term": { "_id": "52097853" }},
  "aggs": { "cost_for_this_doc": { "sum": { "field": "cost" }}}
}
'
```

The agg should return `1.88`, not `1`.

---

## Step 4: Update your ingestion pipeline (if needed)

Before you swap aliases, **make sure your ingestion code points to the new indexes** (or dual-writes to both). Otherwise, new data written after the swap will go to the old index.

If you're using Logstash, update the `output.elasticsearch.index` setting:
```
output {
  elasticsearch {
    index => "call_units_v2"
  }
}
```

If you're using a custom script, change the target index name in the code.

**Optional: Dual-write during transition** (safer for large systems):
- Write to both `call_units` and `call_units_v2` for 24 hours
- Monitor for errors
- Then proceed to alias swap

---

## Step 5: Swap aliases (zero-downtime cutover)

Create aliases that point to the `_v2` indexes. Your application should **always query the alias**, never the index name directly. This makes future reindexes trivial.

### 5a. Create aliases (if they don't exist)

```bash
curl -X POST "localhost:9200/_aliases?pretty" -H 'Content-Type: application/json' -d'
{
  "actions": [
    { "add": { "index": "account_transactions_v2", "alias": "account_transactions_live" }},
    { "add": { "index": "call_units_v2", "alias": "call_units_live" }},
    { "add": { "index": "session_units_v2", "alias": "session_units_live" }},
    { "add": { "index": "template_units_v2", "alias": "template_units_live" }},
    { "add": { "index": "organization_transactions_v2", "alias": "organization_transactions_live" }}
  ]
}
'
```

### 5b. Update application to use aliases

Change all ES queries from:
```bash
curl -X GET "localhost:9200/call_units/_search?pretty"
```
to:
```bash
curl -X GET "localhost:9200/call_units_live/_search?pretty"
```

### 5c. Swap aliases (if they already exist and point to old indexes)

```bash
curl -X POST "localhost:9200/_aliases?pretty" -H 'Content-Type: application/json' -d'
{
  "actions": [
    { "remove": { "index": "account_transactions", "alias": "account_transactions_live" }},
    { "add":    { "index": "account_transactions_v2", "alias": "account_transactions_live" }},
    
    { "remove": { "index": "call_units", "alias": "call_units_live" }},
    { "add":    { "index": "call_units_v2", "alias": "call_units_live" }},
    
    { "remove": { "index": "session_units", "alias": "session_units_live" }},
    { "add":    { "index": "session_units_v2", "alias": "session_units_live" }},
    
    { "remove": { "index": "template_units", "alias": "template_units_live" }},
    { "add":    { "index": "template_units_v2", "alias": "template_units_live" }},
    
    { "remove": { "index": "organization_transactions", "alias": "organization_transactions_live" }},
    { "add":    { "index": "organization_transactions_v2", "alias": "organization_transactions_live" }}
  ]
}
'
```

This operation is **atomic** — all swaps happen in one API call, no downtime.

---

## Step 6: Monitor and drop old indexes

### 6a. Monitor for 24–48 hours

- Check that dashboards/reports show correct totals
- Watch for application errors
- Verify no queries are still hitting the old indexes:
  ```bash
  curl -X GET "localhost:9200/_cat/indices?v&pretty"
  ```
  Look for request counts on the old index names — should be zero.

### 6b. Drop old indexes (after grace period)

Once confident the new indexes are stable:

```bash
curl -X DELETE "localhost:9200/account_transactions?pretty"
curl -X DELETE "localhost:9200/call_units?pretty"
curl -X DELETE "localhost:9200/session_units?pretty"
curl -X DELETE "localhost:9200/template_units?pretty"
curl -X DELETE "localhost:9200/organization_transactions?pretty"
```

**Keep a backup** (snapshot) of the old indexes before deletion, just in case:
```bash
curl -X PUT "localhost:9200/_snapshot/my_backup/snapshot_before_drop?pretty" -H 'Content-Type: application/json' -d'
{
  "indices": "account_transactions,call_units,session_units,template_units,organization_transactions"
}
'
```

---

## Rollback plan (if something goes wrong)

If you discover an issue after the alias swap, **instantly** rollback:

```bash
curl -X POST "localhost:9200/_aliases?pretty" -H 'Content-Type: application/json' -d'
{
  "actions": [
    { "remove": { "index": "account_transactions_v2", "alias": "account_transactions_live" }},
    { "add":    { "index": "account_transactions", "alias": "account_transactions_live" }},
    
    { "remove": { "index": "call_units_v2", "alias": "call_units_live" }},
    { "add":    { "index": "call_units", "alias": "call_units_live" }},
    
    { "remove": { "index": "session_units_v2", "alias": "session_units_live" }},
    { "add":    { "index": "session_units", "alias": "session_units_live" }},
    
    { "remove": { "index": "template_units_v2", "alias": "template_units_live" }},
    { "add":    { "index": "template_units", "alias": "template_units_live" }},
    
    { "remove": { "index": "organization_transactions_v2", "alias": "organization_transactions_live" }},
    { "add":    { "index": "organization_transactions", "alias": "organization_transactions_live" }}
  ]
}
'
```

You're back to the old indexes in <1 second. Fix the issue in the `_v2` indexes, then swap again.

---

## Summary of changes

| Index | Field | Old Type | New Type |
|---|---|---|---|
| `account_transactions` | `dr_amount` | `float` | `double` |
| `account_transactions` | `cr_amount` | `long` | `double` |
| `organization_transactions` | `dr_amount` | `long` | `double` |
| `organization_transactions` | `cr_amount` | `long` | `double` |
| `call_units` | `cost` | `long` | `double` |
| `call_units` | `rate` | `long` | `double` |
| `call_units` | `duration` | `float` | `double` |
| `call_units` | `unit_minutes` | `float` | `double` |
| `session_units` | `cost` | `long` | `double` |
| `session_units` | `rate` | `long` | `double` |
| `template_units` | `cost` | `long` | `double` |
| `template_units` | `rate` | `long` | `double` |

**All other fields remain unchanged.**
