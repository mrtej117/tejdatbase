# 07 - Docker Database Infrastructure

## 1. What is Docker?
Docker is an open-source platform that allows developers to package software into standardized, isolated units called **Containers**. For a database engineer, Docker is a tool that allows you to run complex database systems (like PostgreSQL and Redis) locally on your laptop exactly as they would run on a Linux server in the cloud, without actually installing the databases directly onto your laptop's operating system (OS).

## 2. Why Docker Exists
Historically, setting up a database for local development was a nightmare. A developer using a MacBook, another using Windows, and a production server running Ubuntu would all require completely different installation steps for PostgreSQL. This led to the infamous phrase: *"It works on my machine!"* Docker exists to guarantee that software runs exactly the same way regardless of the underlying operating system.

## 3. Problems Docker Solves
- **Environment Inconsistency:** Docker ensures that the database version, configuration, and dependencies are identical for every developer on the team.
- **Port Clashing:** If you already have PostgreSQL installed locally for a different project, running a second instance normally causes port conflicts. Docker isolates network ports.
- **Messy Uninstalls:** Removing a database from your OS often leaves residual files. With Docker, when you delete a container, it vanishes without a trace.

## 4. Images
A Docker **Image** is a read-only template or "blueprint." It contains the database software, the OS dependencies, and the default configuration. For example, the `postgres:15-alpine` image contains a lightweight Linux OS (Alpine) and PostgreSQL version 15. You download images from a registry like Docker Hub.

## 5. Containers
A Docker **Container** is a running instance of an Image. If an image is the blueprint of a house, the container is the actual house. You can run multiple containers from a single image simultaneously. In our AI Travel Assistant, the PostgreSQL server is a running container.

## 6. Volumes
By default, containers are ephemeral. If you delete a database container, all the data inside it is destroyed. **Volumes** solve this. A volume maps a folder inside the container to a folder safely on your laptop's hard drive. Even if the PostgreSQL container is deleted, the data in the volume remains. 

## 7. Networks
Docker creates virtual networks. By default, containers are completely isolated from your laptop and from each other. To allow the Backend API to talk to the database, they must be attached to the same Docker **Network**. 

## 8. Docker Compose
While Docker handles individual containers, **Docker Compose** is a tool that orchestrates multiple containers at once using a single YAML file (`docker-compose.yml`). Instead of typing ten long commands to start PostgreSQL, Redis, and pgAdmin, Docker Compose starts the entire database infrastructure with one command: `docker-compose up`.

## 9. Why PostgreSQL, Redis, and pgAdmin run inside Docker
For the AI Travel Assistant, we need:
- **PostgreSQL (with pgvector):** For Relational and Long-Term Memory.
- **Redis:** For Short-Term Memory and Rate Limiting.
- **pgAdmin:** A web-based GUI to visually inspect the database.
Installing these natively on every developer's machine is extremely tedious. Running them in Docker ensures the entire database stack can be booted in seconds with zero manual configuration.

## 10. Container Networking
Inside a Docker Compose network, containers can talk to each other using their service names as hostnames. 
- The Backend API doesn't connect to `localhost:5432`. It connects to `postgres:5432`.
- `localhost` is only used when connecting from your laptop *outside* the Docker network (e.g., using a local database client like DBeaver).

## 11. Persistent Database Storage
When running PostgreSQL in Docker, the database files are typically stored inside `/var/lib/postgresql/data` within the container. To make this persistent, we mount a Docker volume to this path.

## 12. Docker Volumes
In `docker-compose.yml`, volumes look like this:
```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```
This tells Docker: "Take the `postgres_data` volume on my host machine and map it to the database directory inside the container."

## 13. Docker Architecture
```mermaid
flowchart TD
    dev([Developer / Laptop])
    
    subgraph desktop [Docker Desktop Engine]
        subgraph compose [Docker Compose Network]
            pg[(PostgreSQL Container\nPort 5432)]
            redis[(Redis Container\nPort 6379)]
            pgadmin[pgAdmin Container\nPort 5050]
        end
        
        vol_pg[(Docker Volume\npostgres_data)]
        vol_redis[(Docker Volume\nredis_data)]
    end

    dev -->|docker-compose up| desktop
    dev -->|localhost:5050| pgadmin
    dev -->|localhost:5432| pg
    
    pgadmin -->|Internal Network| pg
    pg <-->|Reads/Writes| vol_pg
    redis <-->|Reads/Writes| vol_redis
```

## 14. Local Development Workflow
1. Clone the GitHub repository.
2. Ensure Docker Desktop is running.
3. Run `docker-compose up -d`.
4. Run database migrations to construct the schema.
5. Start the Backend API (FastAPI) locally.

## 15. Development vs Production
**Critical Distinction:** We *only* use Docker Compose for the database layer in **Local Development**.
In **Production**, we use managed cloud services (Neon for PostgreSQL, Upstash for Redis). We do not self-host production databases in Docker containers because managing high-availability, failovers, and backups for Docker databases requires dedicated DevOps teams (e.g., Kubernetes).

## 16. Docker Compose Architecture (Conceptual Connection)
*How FastAPI connects to the Docker databases:*
When FastAPI runs on your laptop, it looks at a `.env` file for its connection strings.
- **Database URL:** `postgresql://user:pass@localhost:5432/ai_travel`
- **Redis URL:** `redis://localhost:6379/0`
Docker exposes (publishes) ports `5432` and `6379` from the isolated containers to your laptop's `localhost`, allowing FastAPI to connect as if the databases were installed natively.

```mermaid
sequenceDiagram
    participant FA as FastAPI (Running on Laptop)
    participant Host as Laptop Localhost (Port 5432)
    participant Docker as Docker Container (PostgreSQL)
    
    FA->>Host: Connect to localhost:5432
    Host->>Docker: Port-forward traffic to Container
    Docker-->>FA: Connection Established
```

## 17. Security
- **Never commit `.env` files** containing Docker database passwords to GitHub.
- While local Docker databases use simple passwords (e.g., `POSTGRES_PASSWORD=postgres`), production databases (Neon) require highly secure, rotated passwords.

## 18. Performance
- On Windows and macOS, Docker runs inside a lightweight Linux Virtual Machine. This can sometimes cause slow disk I/O performance. Ensure you are using the latest version of Docker Desktop with "VirtioFS" enabled for optimal volume performance.

## 19. Common Mistakes
- **Forgetting Volumes:** Running a database container without a volume map. The moment the container is stopped or recreated, all database tables and rows are permanently lost.
- **Port Conflicts:** Trying to run `docker-compose up` when a local installation of PostgreSQL is already running on port 5432.

## 20. Best Practices
- Use official, verified images from Docker Hub (e.g., `postgres:15` or `ankane/pgvector`).
- Always run containers in detached mode (`-d`) so your terminal isn't locked.
- Clean up unused volumes regularly to free up hard drive space using `docker volume prune`.

## 21. Terminal Commands
```bash
# Start the database infrastructure in the background
docker-compose up -d

# View logs for the PostgreSQL container
docker-compose logs -f postgres

# Stop the infrastructure (keeps data)
docker-compose down

# Stop the infrastructure AND delete the database data (Full Reset)
docker-compose down -v

# List all running containers
docker ps
```

## 22. Troubleshooting
- **Error: "port is already allocated"** 
  - *Fix:* You have another service using that port. Stop your local PostgreSQL/Redis service, or change the exposed port in `docker-compose.yml` (e.g., `5433:5432`).
- **Error: "database system is shutting down" / Infinite Restart Loop**
  - *Fix:* The Docker volume might be corrupted. Run `docker-compose down -v` to destroy the corrupted volume and start fresh.

## 23. Mermaid Diagrams
*(All Mermaid diagrams in this document are guaranteed to use GitHub-compatible syntax like `flowchart TD` and `sequenceDiagram`.)*

## 24. Summary
Docker completely revolutionizes local database development by eliminating installation friction. By using Docker Compose, any developer can spin up the complex AI Travel Assistant database stack (PostgreSQL + pgvector + Redis + pgAdmin) with a single command. While powerful for local development, we transition to managed cloud providers (Neon/Upstash) for production to ensure maximum reliability and scale.
