---
name: clickhouse-io
description: ClickHouse database patterns, query optimization, analytics, and data engineering best practices for high-performance analytical workloads.
origin: ECC
---

# ClickHouse Analytics Patterns

ClickHouse-specific patterns for high-performance analytics and data engineering.

## When to Activate

- Designing ClickHouse table schemas (MergeTree engine selection)
- Writing analytical queries (aggregations, window functions, joins)
- Optimizing query performance (partition pruning, projections, materialized views)
- Ingesting large volumes of data (batch inserts, Kafka integration)
- Implementing real-time dashboards or time-series analytics

## Table Design Patterns

### MergeTree Engine (Most Common)

```sql
CREATE TABLE events_analytics (
    date Date,
    event_id String,
    user_id String,
    event_type String,
    value Float64,
    created_at DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, user_id, event_type)
SETTINGS index_granularity = 8192;
```

### ReplacingMergeTree (Deduplication)

```sql
CREATE TABLE user_events (
    event_id String,
    user_id String,
    event_type String,
    timestamp DateTime,
    properties String
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (user_id, event_id, timestamp);
```

### AggregatingMergeTree (Pre-aggregation)

```sql
CREATE TABLE stats_hourly (
    hour DateTime,
    dimension String,
    total_value AggregateFunction(sum, Float64),
    total_count AggregateFunction(count, UInt32),
    unique_users AggregateFunction(uniq, String)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, dimension);
```

## Python Integration (clickhouse-driver)

```python
from clickhouse_driver import Client

client = Client(host='localhost', port=9000, database='analytics')

# Batch insert (efficient)
data = [(row['date'], row['user_id'], row['value']) for row in rows]
client.execute(
    'INSERT INTO events_analytics (date, user_id, value) VALUES',
    data,
    types_check=True
)

# Query with parameters
result = client.execute(
    'SELECT date, uniq(user_id) AS dau FROM events_analytics '
    'WHERE date >= %(start)s AND date <= %(end)s GROUP BY date ORDER BY date',
    {'start': '2025-01-01', 'end': '2025-01-31'}
)
```

## Python Integration (clickhouse-connect)

```python
import clickhouse_connect

client = clickhouse_connect.get_client(host='localhost', port=8123)

# Query
result = client.query('SELECT * FROM events_analytics LIMIT 100')
df = result.result_set  # As list of tuples

# Insert from DataFrame
import pandas as pd
df = pd.DataFrame(data)
client.insert_df('events_analytics', df)
```

## Materialized Views

```sql
CREATE MATERIALIZED VIEW stats_hourly_mv
TO stats_hourly
AS SELECT
    toStartOfHour(timestamp) AS hour,
    dimension,
    sumState(value) AS total_value,
    countState() AS total_count,
    uniqState(user_id) AS unique_users
FROM raw_events
GROUP BY hour, dimension;
```

## Query Optimization

```sql
-- Use indexed columns first
SELECT * FROM events_analytics
WHERE date >= '2025-01-01' AND user_id = 'user-123'
ORDER BY date DESC LIMIT 100;

-- Use ClickHouse-specific functions
SELECT
    toStartOfDay(created_at) AS day,
    uniq(user_id) AS dau,
    quantile(0.95)(value) AS p95
FROM events_analytics
WHERE created_at >= today() - INTERVAL 7 DAY
GROUP BY day ORDER BY day;
```

## Best Practices

1. **Partitioning**: By time (month or day), avoid too many partitions
2. **Ordering Key**: Most frequently filtered columns first
3. **Data Types**: Use smallest appropriate type, LowCardinality for repeated strings
4. **Avoid**: SELECT *, FINAL, too many JOINs, small frequent inserts
5. **Batch inserts**: Always batch, never insert row by row
