# 21 - Terminal Commands

## 1. Introduction
As a Database Engineer or Backend Developer on the AI Travel Assistant project, the terminal is your primary interface. This document provides a cheat sheet of the most critical, frequently used commands for managing Git, Docker, PostgreSQL, and Redis.

## 2. Purpose
Instead of searching StackOverflow every time you need to reset a Docker volume or debug a slow PostgreSQL query, use this centralized reference. These commands move you from a beginner level (basic Git) to production-level troubleshooting.

## 3. Git Commands
Used for version control and collaborating on the `Sahayatri` repository.

```bash
# Clone the repository
git clone https://github.com/mrtej117/Sahayatri.git

# Check the status of your changed files
git status

# Stage all modified files for a commit
git add .

# Commit changes with a meaningful message
git commit -m "Update schema to include user preferences"

# Push changes to the main branch
git push origin main

# Pull the latest changes from other developers (with rebase to keep history clean)
git pull --rebase origin main
```

## 4. Docker Compose Commands
Used to manage the local database infrastructure (PostgreSQL, Redis, pgAdmin).

```bash
# Boot the entire database stack in the background
docker-compose up -d

# Stop the database stack (preserves your data)
docker-compose down

# ⚠️ Stop the stack AND DELETE all database volumes (Total Reset)
docker-compose down -v

# View real-time logs for all containers
docker-compose logs -f

# View real-time logs for a specific container (e.g., postgres)
docker-compose logs -f postgres

# See which containers are currently running and their ports
docker ps
```

## 5. PostgreSQL Commands
Used for inspecting and querying the relational data and `pgvector` embeddings.

```bash
# Access the interactive PostgreSQL CLI (psql) inside the Docker container
docker exec -it ai-travel-postgres psql -U postgres -d ai_travel

# Once inside the psql prompt:

# List all tables
\dt

# Describe a specific table (shows columns, types, and indexes)
\d long_term_memories

# Turn on query execution time tracking
\timing on

# Check query performance (EXPLAIN ANALYZE)
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

# Exit psql
\q
```

## 6. Redis Commands
Used for debugging Short-Term Memory (STM) and rate limiting.

```bash
# Access the interactive Redis CLI inside the Docker container
docker exec -it ai-travel-redis redis-cli

# Once inside the redis-cli prompt:

# Check if the server is responding (should return PONG)
PING

# Monitor all incoming Redis commands in real-time (Great for debugging the Backend API)
MONITOR

# See memory usage statistics
INFO memory

# Find a specific user's chat session key
KEYS chat:session:*

# Check the Time-To-Live (expiration) of a specific key
TTL chat:session:12345

# Read the contents of a Short-Term Memory list
LRANGE chat:session:12345 0 -1

# Exit redis-cli
exit
```

## 7. Useful Development & Troubleshooting Commands
When things go wrong, these commands will help you diagnose the issue.

```bash
# Problem: A port is already in use. Find out what process is blocking port 5432 (Postgres)
# On Windows (PowerShell):
Get-Process -Id (Get-NetTCPConnection -LocalPort 5432).OwningProcess

# On macOS/Linux:
lsof -i :5432

# Problem: Docker is out of hard drive space. Clean up unused containers, networks, and images
docker system prune -a

# Test if the local PostgreSQL port is actively accepting connections
# On macOS/Linux:
nc -zv localhost 5432
```

## 8. Summary
Mastering these terminal commands empowers you to rapidly build, debug, and reset the AI Travel Assistant's database architecture. If you encounter an architecture-level issue, consult the master blueprint in the next document: **ARCHITECTURE.md**.
