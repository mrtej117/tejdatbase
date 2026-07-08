# 20 - Development Setup

## 1. Introduction
Welcome to the AI Travel Assistant engineering team! Before you can start writing code, you need to configure your local development environment. This document provides a step-by-step guide to installing the required software and booting the database stack.

## 2. Purpose
A consistent development environment eliminates the "it works on my machine" problem. By following this guide, every developer (whether on Windows, macOS, or Linux) will run the exact same versions of PostgreSQL, pgvector, and Redis.

## 3. Required Software
Before proceeding, ensure you have the following installed on your machine:
- **VS Code:** The recommended IDE for this project.
- **Git:** For version control.
- **Python (3.10+):** For running the FastAPI backend and Memory Agent.
- **Docker Desktop:** Essential for running our database stack locally. Ensure it is running and the Docker icon is visible in your system tray.

### Optional but Recommended:
- **DBeaver or DataGrip:** Advanced GUI tools for inspecting PostgreSQL if you prefer them over pgAdmin.

## 4. Understanding the Local Stack
As detailed in [07 - Docker Database Infrastructure](07_Docker_Database.md), we do not install PostgreSQL or Redis directly onto our operating systems. Instead, we use Docker Compose to spin up three isolated containers:
1. **PostgreSQL (Port 5432):** The primary database with the `pgvector` extension pre-installed.
2. **Redis (Port 6379):** The in-memory data store for Short-Term Memory.
3. **pgAdmin (Port 5050):** A web-based GUI for viewing the PostgreSQL database.

## 5. Project Initialization (Step-by-Step)

### Step 1: Clone the Repository
Open your terminal and clone the GitHub repository:
```bash
git clone https://github.com/mrtej117/Sahayatri.git
cd Sahayatri
```

### Step 2: Configure Environment Variables
In the root directory, copy the example environment file to create your local `.env` file:
```bash
cp .env.example .env
```
Ensure your `.env` file contains the following local connection strings:
```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/ai_travel
REDIS_URL=redis://localhost:6379/0
PGADMIN_DEFAULT_EMAIL=admin@admin.com
PGADMIN_DEFAULT_PASSWORD=admin
```

### Step 3: Boot the Docker Stack
Use Docker Compose to download the images and start the containers in detached (`-d`) mode.
```bash
docker-compose up -d
```
*Note: The first time you run this, it may take a few minutes to download the PostgreSQL and Redis images.*

### Step 4: Verify the Stack
Check that all containers are running successfully:
```bash
docker ps
```
You should see three containers labeled `ai-travel-postgres`, `ai-travel-redis`, and `ai-travel-pgadmin` with the status `Up`.

### Step 5: Database Migrations
The database is currently empty. We must execute the SQL scripts (outlined in [10 - SQL Guide](10_SQL_Guide.md)) to create the tables.
If you are using Alembic (Python), run:
```bash
alembic upgrade head
```
*(Alternatively, if executing raw SQL, you can connect via pgAdmin and paste the DDL script).*

## 6. Accessing pgAdmin (GUI)
1. Open your web browser and navigate to `http://localhost:5050`.
2. Log in using `admin@admin.com` and password `admin`.
3. Right-click **Servers** > **Register** > **Server**.
4. **General Tab:** Name it "Local AI Travel DB".
5. **Connection Tab:** 
   - Host name/address: `postgres` *(Crucial: Use the Docker container name, not localhost)*
   - Port: `5432`
   - Username: `postgres`
   - Password: `postgres`
6. Click Save. You can now visually inspect your `users`, `long_term_memories`, and `itineraries` tables.

## 7. Local Development Workflow
Your daily workflow should look like this:
1. Open Docker Desktop.
2. Open terminal in the project root and run `docker-compose up -d`.
3. Start the Backend API (e.g., `uvicorn main:app --reload`).
4. Write code, test the AI's memory retrieval, and verify database changes in pgAdmin.
5. When finished for the day, run `docker-compose down` to stop the containers (your data is safely persisted in Docker volumes).

## 8. Common Mistakes
- **Port Conflicts:** If `docker-compose up` fails with "port is already allocated", you likely have a local installation of PostgreSQL running on your laptop. You must stop the local PostgreSQL service first.
- **Connecting pgAdmin improperly:** Remember that pgAdmin is running *inside* the Docker network. It must connect to the database using the hostname `postgres`, not `localhost`.

## 9. Summary
Your local environment is now an exact replica of the production database topology. By utilizing Docker, we eliminate configuration drift and allow new engineers to onboard in minutes rather than days. For a quick reference of the commands you'll use every day, proceed to **Terminal Commands**.
