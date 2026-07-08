# 02 - Database Architecture

## 1. Introduction
This document details the core database architecture of the AI Travel Assistant. It delves into how PostgreSQL, `pgvector`, and Redis interact at a structural level to provide a seamless, highly available, and high-performance data layer capable of supporting both transactional consistency and generative AI workloads.

## 2. Purpose
The purpose of this architecture is to provide a blueprint for deploying, connecting, and maintaining the three core persistence technologies (Relational, Vector, and In-Memory). It establishes the rules for how application services (Backend API vs. Memory Agent) communicate with the data layer.

## 3. Problem Statement
A naive database architecture typically connects the application directly to the database. In an AI application, this leads to three major problems:
1. **Connection Exhaustion:** Frequent, small reads/writes for conversational memory quickly exhaust database connection limits.
2. **Compute Bottlenecks:** Heavy vector similarity searches (K-Nearest Neighbors) can starve transactional queries (like processing a payment or finalizing a booking) of CPU resources.
3. **Latency:** Fetching conversational context from disk-backed storage for every LLM prompt generation adds unacceptable latency to the chat experience.

## 4. Internal Working
To mitigate these problems, our architecture utilizes **Connection Pooling** and **Separation of Workloads**:
- **PgBouncer (Connection Pooling):** Placed in front of PostgreSQL. It multiplexes thousands of incoming application connections down to a few dozen physical connections to the database.
- **Workload Routing:** The Backend API handles high-consistency, low-latency relational CRUD operations. The Memory Agent handles compute-heavy vector similarity searches.
- **Caching Layer:** Redis acts as a buffer. All high-frequency, low-latency read/writes for immediate chat state (Short-Term Memory) hit Redis entirely, bypassing PostgreSQL until a "Memory Snapshot" is required.

## 5. Architecture
Below is the detailed deployment architecture of the database layer.

```mermaid
architecture-beta
    group vpc(cloud)[Virtual Private Cloud]
    
    group app_subnet(server)[Application Subnet] in vpc
    service backend(server)[Backend API (Node.js/Python)] in app_subnet
    service agent(server)[Memory Agent (Python)] in app_subnet

    group db_subnet(database)[Database Subnet (Private)] in vpc
    service pooler(server)[PgBouncer\n(Connection Pooler)] in db_subnet
    service postgres(database)[Neon PostgreSQL\n(Primary)] in db_subnet
    service pgvector(database)[pgvector Extension] in db_subnet
    service redis(database)[Upstash Redis\n(Primary)] in db_subnet

    backend:B --> T:pooler
    agent:B --> T:pooler
    pooler:B --> T:postgres
    postgres:B --> T:pgvector
    
    agent:R --> L:redis
    backend:R --> L:redis
```

## 6. Data Flow
1. **API Request**: The Backend API receives a request to view a booking. It requests a connection from PgBouncer.
2. **Pooler Multiplexing**: PgBouncer assigns an available physical PostgreSQL connection to the API.
3. **Transaction**: The query executes against PostgreSQL. Once complete, the physical connection is returned to the pool, but the logical connection to the API remains open.
4. **AI Generation**: The Memory Agent needs context to answer a prompt. It first queries Redis for the last 5 messages (sub-millisecond latency).
5. **Semantic Search**: The Agent then queries PostgreSQL (via PgBouncer) using `pgvector` to find the top 3 similar past trips.

## 7. Diagrams (Mermaid)
*Network & Connection Flow Topology*

```mermaid
flowchart LR
    Client([Client Apps / Web]) --> API[Backend API]
    Client --> Agent[Memory Agent]
    
    subgraph Managed Cloud Services
        subgraph Neon [Neon Serverless Postgres]
            Pooler[PgBouncer]
            subgraph DB [PostgreSQL Instance]
                Rel[Relational Tables]
                Vec[pgvector Indexes]
            end
            Pooler --> DB
        end
        
        subgraph Upstash [Upstash Redis]
            Cache[(Redis Cluster)]
        end
    end
    
    API ==>|Transaction Mode| Pooler
    Agent ==>|Session Mode| Pooler
    Agent -->|Direct TCP| Cache
    API -.->|Rate Limits| Cache
```

## 8. Best Practices
- **Transaction Mode Pooling**: Configure PgBouncer to use `transaction` mode. This is the most efficient pooling method for stateless web APIs, returning connections to the pool immediately after a transaction completes.
- **Timeouts**: Enforce `statement_timeout` in PostgreSQL to strictly kill any vector searches that take longer than 3-5 seconds.
- **Separate Credentials**: Create separate database roles (users) for the Backend API (relational access) and Memory Agent (relational + vector access).

## 9. Common Mistakes
- **Connecting directly to PostgreSQL**: Bypassing PgBouncer in production will lead to "Too many clients" errors as the application scales.
- **Large Vector Payloads in Redis**: Storing heavy embeddings (e.g., arrays of 1536 floats) in Redis. Redis should only store string-based conversational history or metadata; embeddings belong in `pgvector`.
- **Public IP Exposure**: Leaving PostgreSQL or Redis ports (5432 / 6379) bound to `0.0.0.0` without IP whitelisting or VPC peering.

## 10. Production Recommendations
- Use **Neon's Autoscaling**: Configure Neon to scale compute up during peak booking hours and scale down to zero during idle hours to save costs.
- Use **Read Replicas**: If analytical workloads (reporting) increase, route them to a dedicated read-only replica.

## 11. Step-by-Step Implementation
1. Provision the primary PostgreSQL database on Neon.
2. Enable connection pooling in the Neon dashboard.
3. Provision the Redis cluster on Upstash.
4. Define network security groups/VPC peering between the app servers and the databases.
5. Configure the Backend API and Memory Agent to use the pooled connection strings.

## 12. Folder Structure
The infrastructure and architecture code should be organized as follows:

```text
/infrastructure
├── /terraform
│   ├── main.tf             # Core infrastructure
│   ├── neon_postgres.tf    # Neon DB provisioning & pooler config
│   ├── upstash_redis.tf    # Redis cluster provisioning
│   └── variables.tf
└── /scripts
    └── test_db_connection.py # Health check scripts
```

## 13. SQL Examples
```sql
-- Enforcing a statement timeout (e.g., 5 seconds) to prevent vector search lockups
ALTER ROLE ai_memory_agent SET statement_timeout = '5000ms';

-- Creating separate roles for the architecture
CREATE ROLE backend_api_user WITH LOGIN PASSWORD 'secure_password';
CREATE ROLE memory_agent_user WITH LOGIN PASSWORD 'secure_password';

-- Granting specific privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO backend_api_user;
```

## 14. Terminal Commands
```bash
# Test connection to the Neon Pooler
psql "postgresql://user:password@ep-cool-snowflake-123456-pooler.us-east-2.aws.neon.tech/neondb?sslmode=require"

# Monitor active connections (run inside psql)
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
```

## 15. Deployment Considerations
- Use CI/CD to inject database connection strings as secrets into the application environment at deployment time. Never hardcode them.
- Neon databases natively support branching. Use database branches for automated end-to-end testing in pull requests.

## 16. Security Considerations
- Require SSL/TLS (`sslmode=require`) for all connections between the application servers, PostgreSQL, and Redis.
- Rotate database passwords every 90 days.

## 17. Performance Optimization
- For `pgvector`, build the index *after* performing bulk initial data loads to ensure the graph (HNSW) or clusters (IVFFlat) are built optimally.
- Use Redis `EXPIRE` (TTL) on all Short-Term Memory keys to automatically purge stale chat sessions and free up RAM.

## 18. References
- [PgBouncer Official Documentation](https://www.pgbouncer.org/)
- [Neon Connection Pooling](https://neon.tech/docs/connect/connection-pooling)
- [Upstash Redis Best Practices](https://upstash.com/docs/redis/overall/bestpractices)
