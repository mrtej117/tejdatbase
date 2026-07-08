# 23 - Project Folder Structure

## 1. Introduction
A consistent, logical folder structure is crucial for any enterprise-grade application. It allows new developers to instantly locate database models, migration scripts, and configuration files without getting lost in a chaotic codebase.

## 2. Purpose
This document explains the structural layout of the AI Travel Assistant repository. While this documentation suite specifically focuses on the database architecture, we will outline the backend structure to demonstrate where the database code lives.

## 3. Root Directory Overview
```text
Sahayatri/
│
├── .github/                  # GitHub Actions CI/CD workflows
├── docs/                     # ⬅️ You are here. Architecture & DB documentation.
├── backend/                  # The FastAPI backend and Memory Agent
├── frontend/                 # The React/Next.js client interface
│
├── docker-compose.yml        # Orchestrates Local PostgreSQL, Redis, pgAdmin
├── .env.example              # Template for required environment variables
├── .gitignore                # Files Git should ignore (e.g., actual .env files)
├── README.md                 # Project entry point and high-level overview
├── CONTRIBUTING.md           # Guide for how to submit pull requests
└── LICENSE                   # MIT Open Source License
```

## 4. The Backend Database Structure
Inside the `/backend` folder, the database logic is cleanly separated from the API routing logic using a classic MVC-like domain structure.

```text
backend/
│
├── alembic/                  # 🚀 Database Migration Engine
│   ├── versions/             # Auto-generated SQL version files (e.g., 'add_users_table.py')
│   └── env.py                # Alembic configuration linking to our DB models
│
├── src/                      # Source Code
│   ├── api/                  # FastAPI routers (Endpoints)
│   │
│   ├── database/             # 🗄️ Core Database Logic
│   │   ├── connection.py     # Connection pooling logic (AsyncPG / SQLAlchemy)
│   │   ├── session.py        # Dependency injection for DB sessions
│   │   └── models/           # SQLAlchemy ORM definitions
│   │       ├── users.py      # user and user_preferences tables
│   │       ├── memory.py     # long_term_memories table (pgvector)
│   │       └── travel.py     # itineraries and destinations tables
│   │
│   ├── cache/                # ⚡ Redis Logic
│   │   ├── client.py         # Upstash / Local Redis connection
│   │   └── stm.py            # Short-Term Memory read/write functions
│   │
│   ├── agent/                # 🧠 AI Memory Agent
│   │   ├── extractor.py      # Uses LLM to extract facts from Redis STM
│   │   └── vector_utils.py   # Calls OpenAI to generate embeddings
│   │
│   └── core/                 # App configuration and Pydantic schemas
│
├── alembic.ini               # Alembic root config
├── requirements.txt          # Python dependencies (SQLAlchemy, pgvector, redis)
└── main.py                   # FastAPI application entry point
```

## 5. Why This Structure? (Best Practices)
- **Separation of Concerns:** Notice how `models/` (the structure of the DB) is completely separated from `api/` (the HTTP requests). This allows a Database Engineer to safely modify tables without breaking HTTP routes.
- **Migration Isolation:** The `alembic/` folder lives alongside the `src/` code. We never manually execute `CREATE TABLE` in production. We write Python models, and Alembic generates the versioned migration scripts ensuring production and local environments match exactly.
- **Cache Isolation:** Redis logic (`cache/`) is entirely separate from PostgreSQL logic (`database/`). If we ever replaced Redis with Memcached, the PostgreSQL code wouldn't need to change.

## 6. Common Mistakes
- **Putting SQL inside API routes:** Writing raw `SELECT * FROM users` directly inside the `api/` folder. This makes the code untestable and creates massive security vulnerabilities (SQL Injection). Always keep SQL and ORM queries inside the `database/` layer.

## 7. Summary
By keeping our folders strictly organized by domain (Database, Cache, Agent, API), a developer can intuitively know exactly where a bug resides. If `pgvector` throws an error, they know to check `database/models/memory.py`. 

In the next document, we will formally justify **why** we chose these specific technologies (like `pgvector` and Redis) over the hundreds of available alternatives.
