# 18 - Performance Optimization

## 1. Introduction
Performance Optimization is the discipline of ensuring the database responds to queries in milliseconds, even when scaling to millions of users. For an AI Travel Assistant, a slow database translates directly into a slow AI response, completely destroying the illusion of a fluid conversation.

## 2. Purpose
This document teaches developers how to proactively tune PostgreSQL, `pgvector`, and Redis. It moves beyond simply "making it work" to "making it work at a massive scale."

## 3. PostgreSQL Indexing
Without an index, PostgreSQL performs a **Sequential Scan** (checking every single row one by one). With an index, it uses a B-Tree to find the row instantly.

### Relational Indexes
Always index foreign keys and columns frequently used in `WHERE` clauses.
```sql
-- Standard B-Tree index on a foreign key
CREATE INDEX idx_itineraries_user_id ON itineraries(user_id);

-- Partial index (only index upcoming trips, saving RAM)
CREATE INDEX idx_upcoming_trips ON itineraries(start_date) WHERE start_date > NOW();
```

## 4. Query Optimization & EXPLAIN ANALYZE
You cannot fix what you cannot measure. The `EXPLAIN ANALYZE` command tells you exactly how PostgreSQL executes a query.

```sql
EXPLAIN ANALYZE 
SELECT * FROM itineraries WHERE user_id = 'a1b2c3d4-e5f6-7890-1234-56789abcdef0';
```
**Output to watch for:**
- `Seq Scan on itineraries`: **BAD.** This means you are missing an index.
- `Index Scan using idx_itineraries_user_id`: **GOOD.** The database used your B-Tree index.
- `Execution Time: 0.045 ms`: Extremely fast.

## 5. pgvector Optimization
Vector searches are notoriously CPU intensive. Performing an exact Nearest Neighbor search on 1 million vectors will crash the server. We optimize this using **HNSW (Hierarchical Navigable Small World)** indexes.

### HNSW Tuning
When creating an HNSW index, you tune two parameters:
- `m`: The maximum number of connections per layer. (Default: 16. Increase to 24 for higher recall at the cost of build time).
- `ef_construction`: The size of the dynamic list during index creation. (Default: 64. Increase to 128 for better accuracy).

```sql
CREATE INDEX ltm_hnsw_idx ON long_term_memories 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 24, ef_construction = 128);
```

## 6. Redis Optimization
Redis is blazing fast, but it is single-threaded. If you send a command that takes 1 second to process, *every other user* is blocked for 1 second.
- **Avoid `KEYS *`:** Never use this in production. It blocks the entire server. Use `SCAN` instead.
- **Use Pipelining:** If the Backend API needs to save 5 new session keys, don't make 5 separate network requests. Send them all at once using a Redis Pipeline.

## 7. Connection Pooling
PostgreSQL forks a new OS process for every connection. If 1,000 users connect simultaneously, the server will crash from RAM exhaustion.
- **The Fix:** Use **PgBouncer** (automatically included in Neon's pooled connection strings). PgBouncer holds a small number of real PostgreSQL connections open and rapidly multiplexes the 1,000 user requests through them.

## 8. Database Server Tuning (PostgreSQL Conf)
If you were managing PostgreSQL manually on a VM, you would tune `postgresql.conf`. (Neon manages this automatically based on your compute tier).
- `shared_buffers = 25% of RAM` (How much RAM PostgreSQL uses for caching).
- `work_mem = 16MB` (Memory for sorting operations like `ORDER BY`. If this is too small, Postgres writes temp files to the slow disk).

## 9. Scaling Recommendations
- **Scale Up (Vertical):** Need faster complex queries? Upgrade the Neon compute node to more CPU cores.
- **Scale Out (Horizontal):** Need to handle 10,000 concurrent users simply reading their itineraries? Add **Read Replicas** in Neon and route `SELECT` queries to the replicas, keeping the Primary node free for `INSERT`/`UPDATE` operations.

## 10. Common Mistakes
- **Over-indexing:** Creating an index on every single column. Indexes speed up `SELECT` queries but slow down `INSERT` and `UPDATE` queries (because the index must be rewritten). Only index what you actually query.
- **N+1 Query Problem:** The Backend API queries a user, then runs a `SELECT` query in a loop for every single itinerary the user owns. **Fix:** Use SQL `JOIN`s to fetch all data in a single network round-trip.

## 11. Security Considerations
When running `EXPLAIN ANALYZE` in a production environment, be aware that it actually executes the query. If you run `EXPLAIN ANALYZE DELETE FROM users;`, it **WILL** delete the users! (Use `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;` to be safe).

## 12. Summary
Performance optimization is a continuous cycle of measuring, indexing, and tuning. By leveraging B-Trees for relational data, HNSW for vector data, and PgBouncer for connections, the database will handle immense scale effortlessly. However, to know *when* to optimize, we need visibility into the system. This brings us to our next document: **Monitoring**.
