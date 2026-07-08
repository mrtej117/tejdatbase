# 24 - Technology Decisions

## 1. Introduction
Architecture is entirely about trade-offs. There is no "perfect" database; there is only the *right* database for a specific use case. This document formally justifies the technology choices made for the AI Travel Assistant.

## 2. PostgreSQL vs. MySQL
*Why we chose PostgreSQL for relational data.*
- **Advanced Data Types:** PostgreSQL natively supports `JSONB` and arrays, allowing us to store dynamic user preferences (like dietary restrictions) without creating dozens of complex join tables. MySQL's JSON support is less mature.
- **Extensibility:** The PostgreSQL extension ecosystem is unmatched. It allows us to seamlessly install `pgvector`, effectively turning our relational database into a cutting-edge AI vector store.

## 3. PostgreSQL vs. MongoDB (NoSQL)
*Why we didn't use a Document database.*
- **ACID Compliance:** Travel itineraries involve money, booking dates, and strict foreign keys (e.g., a trip *must* belong to a registered user). Relational databases mathematically guarantee this integrity.
- **Joins:** MongoDB struggles with complex joins. An AI travel app constantly joins users, preferences, trips, and destinations together to build context.

## 4. pgvector vs. Dedicated Vector DBs (Pinecone, Qdrant)
*Why we store embeddings inside PostgreSQL instead of a specialized vector database.*
- **Single Source of Truth:** If we stored vectors in Pinecone and users in PostgreSQL, what happens when a user deletes their account? We would have to orchestrate a distributed delete across two cloud providers. With `pgvector`, deleting a user triggers an `ON DELETE CASCADE`, instantly wiping their memories.
- **Metadata Filtering:** `pgvector` allows us to write queries like `WHERE user_id = X ORDER BY embedding <=> Y`. Performing this across two distinct databases (Pinecone + PostgreSQL) requires complex network calls and risks massive data leaks if the filtering fails.

## 5. Redis vs. PostgreSQL for Caching
*Why we didn't just use PostgreSQL for Short-Term Memory.*
- **Disk I/O:** PostgreSQL persists to a hard drive (SSD). Redis is 100% in-memory (RAM). A fast-paced chat conversation requires sub-millisecond read/writes. Writing every single "hello" to a PostgreSQL SSD wastes expensive compute and slows down the LLM response time.
- **Data Expiration (TTL):** Redis natively supports Time-To-Live. We can tell Redis, "Delete this chat context in 2 hours." Implementing this in PostgreSQL requires writing complex cron jobs to sweep and delete old rows continuously.

## 6. Neon vs. AWS RDS / Supabase
*Why we chose Neon for PostgreSQL hosting.*
- **Autoscaling to Zero:** Travel apps have distinct seasonality and daily usage spikes. Neon dynamically scales CPU based on real-time load and scales to zero at 4:00 AM, drastically cutting hosting costs.
- **Instant Branching:** Supabase and RDS require heavy, time-consuming cloning to create a staging database. Neon uses Copy-on-Write storage, allowing us to create a 1TB clone of production in exactly 1 second for CI/CD testing.

## 7. Upstash vs. ElastiCache
*Why we chose Upstash for Redis.*
- **Serverless REST API:** Upstash allows us to interact with Redis via standard HTTP REST calls. This is a game-changer if the Backend API is deployed on serverless functions (like Vercel or AWS Lambda), which struggle to maintain the persistent TCP connections required by traditional Redis (like AWS ElastiCache).

## 8. Docker Compose for Local Development
*Why not just install Postgres and Redis directly on macOS/Windows?*
- **Onboarding:** Installing PostgreSQL natively often results in version mismatches (e.g., Dev A uses PG14, Dev B uses PG16). Docker ensures that with a single `docker-compose up -d` command, every developer on earth is running the exact same version of PostgreSQL and Redis in an isolated environment.

## 9. OpenAI `text-embedding-3-small`
*Why this specific embedding model?*
- **Cost vs Performance:** It represents the current industry sweet spot. It provides 1536-dimensional semantic depth with incredibly high benchmark accuracy, but costs fractions of a cent per prompt.
- **Normalization:** It natively outputs normalized vectors, meaning we can use the highly optimized Inner Product (`<#>`) or Cosine Distance (`<=>`) operators in `pgvector` interchangeably.

## 10. Future Migration Strategy
If the AI Travel Assistant goes hyper-viral and reaches 100 million users:
- **Database:** We may outgrow a single PostgreSQL instance. At that scale, we would migrate to a horizontally sharded PostgreSQL variant like **Citus** or migrate to a globally distributed database like **CockroachDB**.
- **Vector Search:** If the `long_term_memories` table hits 50 billion rows, `pgvector` might struggle. At that extreme scale, we would finally migrate vectors out of PostgreSQL into a dedicated distributed vector engine like **Milvus**, accepting the complexity of distributed transactions in exchange for limitless scale.

## 11. Summary
Every technology in this stack was chosen to maximize developer velocity, reduce operational costs, and guarantee data safety. By combining Neon, Upstash, and `pgvector`, we have built an enterprise-grade AI architecture that requires almost zero traditional DevOps overhead.
