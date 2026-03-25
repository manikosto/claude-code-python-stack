---
name: postgres-patterns
description: PostgreSQL database patterns for query optimization, schema design, indexing, and security.
origin: ECC
---

# PostgreSQL Patterns

Quick reference for PostgreSQL best practices.

## When to Activate

- Writing SQL queries or migrations
- Designing database schemas
- Troubleshooting slow queries
- Implementing Row Level Security

## Index Cheat Sheet

| Query Pattern | Index Type | Example |
|--------------|------------|---------|
| `WHERE col = value` | B-tree (default) | `CREATE INDEX idx ON t (col)` |
| `WHERE col > value` | B-tree | `CREATE INDEX idx ON t (col)` |
| `WHERE a = x AND b > y` | Composite | `CREATE INDEX idx ON t (a, b)` |
| `WHERE jsonb @> '{}'` | GIN | `CREATE INDEX idx ON t USING gin (col)` |
| `WHERE tsv @@ query` | GIN | `CREATE INDEX idx ON t USING gin (col)` |
| Time-series ranges | BRIN | `CREATE INDEX idx ON t USING brin (col)` |

## Data Type Quick Reference

| Use Case | Correct Type | Avoid |
|----------|-------------|-------|
| IDs | `bigint` | `int`, random UUID |
| Strings | `text` | `varchar(255)` |
| Timestamps | `timestamptz` | `timestamp` |
| Money | `numeric(10,2)` | `float` |
| Flags | `boolean` | `varchar`, `int` |

## Common Patterns

```sql
-- Composite Index (equality first, then range)
CREATE INDEX idx ON orders (status, created_at);

-- Covering Index
CREATE INDEX idx ON users (email) INCLUDE (name, created_at);

-- Partial Index
CREATE INDEX idx ON users (email) WHERE deleted_at IS NULL;

-- UPSERT
INSERT INTO settings (user_id, key, value) VALUES (123, 'theme', 'dark')
ON CONFLICT (user_id, key) DO UPDATE SET value = EXCLUDED.value;

-- Cursor Pagination (O(1) vs OFFSET O(n))
SELECT * FROM products WHERE id > $last_id ORDER BY id LIMIT 20;

-- Queue Processing
UPDATE jobs SET status = 'processing'
WHERE id = (
  SELECT id FROM jobs WHERE status = 'pending'
  ORDER BY created_at LIMIT 1
  FOR UPDATE SKIP LOCKED
) RETURNING *;
```

## Anti-Pattern Detection

```sql
-- Find unindexed foreign keys
SELECT conrelid::regclass, a.attname
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i WHERE i.indrelid = c.conrelid AND a.attnum = ANY(i.indkey)
  );

-- Find slow queries
SELECT query, mean_exec_time, calls FROM pg_stat_statements
WHERE mean_exec_time > 100 ORDER BY mean_exec_time DESC;
```
