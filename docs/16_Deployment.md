# 16 - Deployment

## 1. Introduction
Deployment is the process of moving the database architecture from a developer's local laptop to a live, internet-accessible production environment. This document bridges the gap between the theoretical architecture of the AI Travel Assistant and its physical manifestation in the cloud.

## 2. Purpose
A database that only runs on `localhost` cannot serve real users. We must deploy our infrastructure securely, reliably, and cost-effectively. This document explains the complete deployment pipeline, from local Docker testing to serverless cloud hosting via Neon and Upstash.

## 3. Local Development Using Docker
Before deploying to production, engineers build and test locally. As outlined in [07 - Docker Database Infrastructure](07_Docker_Database.md), we use Docker to simulate the production environment.

### Docker Compose Workflow
Developers spin up the entire database stack instantly using `docker-compose.yml`.
1. The developer runs `docker-compose up -d`.
2. A PostgreSQL container (with `pgvector`) boots on port `5432`.
3. A Redis container boots on port `6379`.
4. The Backend API connects via `localhost`.

*Development Environment Variables:*
```env
# .env.local
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/ai_travel
REDIS_URL=redis://localhost:6379/0
```

## 4. Production Architecture
In production, we abandon Docker for the database layer. Managing highly available database clusters inside Docker on raw virtual machines requires dedicated DevOps engineers. Instead, we use **Managed Serverless Providers**.

```mermaid
flowchart TD
    subgraph Production Environment
        Client([Mobile / Web App])
        
        subgraph Backend [Backend Infrastructure]
            API[FastAPI / Node.js API]
        end
        
        subgraph Cloud_DB [Serverless Database Layer]
            Neon[(Neon PostgreSQL)]
            Upstash[(Upstash Redis)]
        end
    end

    Client -->|HTTPS| API
    API -->|Pooled Connection| Neon
    API -->|REST / TCP| Upstash
```

## 5. Neon PostgreSQL Deployment
As discussed in [05 - Neon Serverless PostgreSQL](05_Neon_PostgreSQL.md), Neon separates compute and storage.
1. **Creation:** Create a new project in the Neon Dashboard.
2. **Configuration:** Enable the `pgvector` extension via the SQL editor.
3. **Connection:** Copy the **Pooled Connection String**. Standard PostgreSQL connections cap out quickly. The pooled string routes traffic through Neon's internal PgBouncer.

*Production Environment Variable:*
```env
# .env.production
DATABASE_URL=postgresql://user:pass@ep-cool-snowflake-1234.us-east-2.aws.neon.tech/ai_travel?sslmode=require&options=project%3Dendpoint
```

## 6. Upstash Redis Deployment
For Short-Term Memory, we use **Upstash**, a serverless Redis provider perfectly tailored for transient AI conversational state.
1. **Creation:** Create a Redis database in the Upstash Dashboard. Select a region closest to your Backend API (e.g., AWS `us-east-1`).
2. **Eviction Policy:** Crucially, set the eviction policy to `allkeys-lru` (Least Recently Used) so old chats are automatically deleted if RAM fills up.

*Production Environment Variable:*
```env
REDIS_URL=rediss://default:secure_pass@global-upstash.io:32000
```

## 7. CI/CD Overview
Continuous Integration / Continuous Deployment (CI/CD) ensures that database schema changes are applied safely.
1. Developer pushes a schema change to GitHub (e.g., a new Alembic or Prisma migration file).
2. GitHub Actions detects the change.
3. GitHub Actions connects to a **Neon Database Branch** (a safe, isolated clone of production).
4. The action runs the migration and unit tests against the branch.
5. If successful, the branch is merged or the migration is applied to the production Neon endpoint.

## 8. Scaling Strategy
- **Redis (Upstash):** Scales automatically based on requests per second. If the Memory System gets overwhelmed, Upstash adds throughput seamlessly.
- **PostgreSQL (Neon):** Neon's autoscaling dynamically adds Compute Units (CPU/RAM) during high traffic (e.g., holiday travel season) and scales to zero (or a minimum baseline) during off-peak hours.

## 9. Cost Considerations
- **Neon:** You are billed for active Compute Time and Storage volume. Ensure runaway queries are killed using statement timeouts to prevent billing spikes.
- **Upstash:** Billed per request. This makes polling extremely expensive. The Backend API should never use `setInterval` to check Redis continuously.

## 10. Security Considerations
- **VPC Peering:** If your Backend API is on AWS, peer your VPC directly with Neon and Upstash so database traffic never touches the public internet.
- **Secret Management:** Never hardcode connection strings. Inject them at runtime using a secure vault (like AWS Secrets Manager or Vercel Environment Variables).

## 11. Common Mistakes
- **Using the Non-Pooled Neon URL:** Using a direct PostgreSQL connection string in a serverless backend (like AWS Lambda) will instantly exhaust the database's connection limit, taking down the entire app. Always use the `-pooler` URL.
- **Deploying to Mismatched Regions:** If your API is in London (eu-west-2) and your Neon Database is in Tokyo (ap-northeast-1), every database query will take 250ms of network latency. Deploy them in the exact same region.

## 12. Troubleshooting
**Issue:** The app connects to the database locally but fails in production.
- *Fix:* Ensure `sslmode=require` is appended to your production database URL. Cloud providers reject unencrypted connections by default.

## 13. Summary
Deploying the AI Travel Assistant requires a shift in mindset from "managing servers" to "orchestrating managed services." By leveraging Neon and Upstash, we inherit enterprise-grade scaling and uptime out of the box. However, even the best cloud providers suffer outages or accidental data deletions. In the next document, we will explore **Backup and Recovery**.
