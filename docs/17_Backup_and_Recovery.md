# 17 - Backup and Recovery

## 1. Introduction
Backup and Recovery (Disaster Recovery) is the definitive safety net of database engineering. No matter how resilient the architecture is, human error (like accidentally dropping a table) or catastrophic data center failures can destroy data. This document outlines how the AI Travel Assistant's data is protected and restored.

## 2. Purpose
The purpose of this document is to ensure that if the absolute worst-case scenario occurs, the engineering team can restore the database to a working state with minimal data loss and minimal downtime (Recovery Time Objective and Recovery Point Objective).

## 3. Data Classification
Before backing up, we must understand *what* we are backing up:
1. **Critical Data (PostgreSQL):** User accounts, billing, generated itineraries, and Long-Term Memory (LTM) embeddings. This data is irreplaceable.
2. **Transient Data (Redis):** Short-Term Memory (STM) chat sessions and rate-limit counters. If this is lost, users simply have to restart their current conversation.

## 4. PostgreSQL Backups (Neon)
Because we deployed on Neon (as detailed in [16 - Deployment](16_Deployment.md)), we benefit from its unique Storage Architecture.

### 4.1. Point-in-Time Recovery (PITR)
Traditional databases take a backup snapshot once every 24 hours. If your database crashes at 11:59 PM, you lose 23 hours and 59 minutes of data.
Neon's Page Server architecture continuously records every Write-Ahead Log (WAL) transaction. This enables **Point-in-Time Recovery**.
- **Scenario:** A developer accidentally executes `DELETE FROM users;` at 14:05:00.
- **Recovery:** You can literally revert the entire database to exactly 14:04:59, retrieving all deleted data instantly.

### 4.2. Backup Automation Workflow
```mermaid
sequenceDiagram
    participant PG as Neon Compute
    participant S as Neon Safekeepers
    participant P as Neon Page Server
    participant S3 as AWS S3 (Cold Storage)
    
    PG->>S: Streams WAL (every millisecond)
    S->>P: Reconstructs Data Pages
    P->>S3: Nightly automated snapshot (Cold Backup)
```

## 5. Redis Persistence (Upstash)
By default, Redis keeps data purely in RAM. If the server reboots, the data evaporates.
Upstash automatically configures persistence to ensure high availability:
- **Snapshots:** Upstash takes daily snapshots of the Redis memory state.
- **Multi-Zone Replication:** The data is replicated across multiple Availability Zones. If the primary Redis node crashes, a replica instantly takes over.
*Recommendation:* Because Short-Term Memory is transient, do not spend engineering hours building complex backup scripts for Redis. Rely on the provider's default resilience.

## 6. Disaster Recovery Procedures

### Level 1: Bad Migration (Logical Error)
If a schema migration corrupts a column:
1. Go to the Neon Dashboard.
2. Select the "Branches" menu.
3. Restore the main branch to the exact timestamp before the migration.
4. Total downtime: ~2 minutes.

### Level 2: Region Outage (Physical Error)
If the AWS `us-east-1` region goes offline entirely:
1. Neon's storage is replicated to Amazon S3.
2. You must spin up a new Neon project in an unaffected region (e.g., `us-west-2`).
3. Import the latest logical dump (`pg_dump`) or request Neon support to mount the S3 cold storage.
4. Update the Backend API's `DATABASE_URL` to point to the new endpoint.

## 7. Best Practices
- **Test Your Restores:** A backup is completely useless if you don't know how to restore it. Once a quarter, perform a "Fire Drill" where the team restores a backup to an isolated staging environment to verify data integrity.
- **Logical Backups:** Don't rely 100% on Neon's internal magic. Run a weekly `pg_dump` via a cron job and save the raw `.sql` file to an external cloud bucket (like Google Cloud Storage) to avoid vendor lock-in.

## 8. Common Mistakes
- **Backing up API Keys:** Never store API keys or application environment variables in the database. Backups often end up in less-secure storage buckets, and exposing a backup could expose your entire infrastructure.
- **Ignoring pgvector indexes:** When restoring a logical backup (`pg_dump`), remember that rebuilding HNSW vector indexes on millions of rows can take hours. Account for this in your Recovery Time Objective (RTO).

## 9. Terminal Commands (Logical Backups)
```bash
# Export the entire database schema and data to a file
pg_dump "postgresql://user:pass@endpoint.aws.neon.tech/ai_travel" -F c -f backup.dump

# Restore the database from the dump file
pg_restore -d "postgresql://user:pass@new-endpoint.aws.neon.tech/ai_travel" backup.dump
```

## 10. Summary
By utilizing Neon's Point-in-Time Recovery and scheduled logical backups, the AI Travel Assistant's long-term memory and user profiles are virtually immune to data loss. While restoring data is crucial, it's better if the database never struggles in the first place. In the next document, we will dive into **Performance Optimization** to ensure queries run blazingly fast.
