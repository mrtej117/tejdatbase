# 14 - Retrieval Pipeline

## 1. Introduction
The Retrieval Pipeline is the process of fetching the correct information from the database *before* the AI generates a response. If the Memory Agent is how the AI "learns" and "remembers," the Retrieval Pipeline is how the AI "recalls."

## 2. Purpose
An LLM cannot directly query a PostgreSQL database on its own. When a user asks a question, the backend must instantly search the database, retrieve the most relevant facts (both semantic memories and rigid relational data), and inject them into the LLM's prompt. This guarantees the AI's response is highly personalized and factually accurate.

## 3. The Retrieval Steps
The pipeline executes sequentially in milliseconds:
1. **Query Embedding:** Convert the user's raw text prompt into a vector.
2. **Vector Search:** Query `pgvector` for semantic matches.
3. **Metadata Filtering:** Narrow the search using standard SQL.
4. **SQL Retrieval:** Query relational tables for concrete facts (e.g., booked trips).
5. **Context Assembly:** Combine all retrieved data.
6. **Prompt Construction:** Inject the assembled data into the LLM's system prompt.

## 4. Query Embedding
When the user types: *"Can you recommend a hotel for my next trip?"*
The Backend API immediately sends this text to the Embedding Model (OpenAI) to get the query vector: `[0.22, -0.88, 0.44, ...]`.

## 5. Vector Search & Metadata Filtering
Using the query vector, the Backend queries PostgreSQL. Crucially, we use **Metadata Filtering**. We don't want to search the entire `long_term_memories` table; we *only* want to search memories belonging to this specific user.

```sql
SELECT memory_text, (embedding <=> '[0.22, -0.88, ...]'::vector) as distance
FROM long_term_memories
WHERE user_id = 'a1b2c3d4-e5f6-7890-1234-56789abcdef0' -- Metadata Filter
ORDER BY distance ASC
LIMIT 3;
```
*Result returned from pgvector:* "User prefers Marriott hotels.", "User needs wheelchair-accessible rooms."

## 6. SQL Retrieval (Relational Data)
Simultaneously, the pipeline executes standard SQL to fetch the user's hard data from the rigid tables.
```sql
SELECT title, start_date, end_date 
FROM itineraries 
WHERE user_id = 'a1b2c3d4...' AND start_date > NOW();
```
*Result returned:* Trip Title: "Summer in Miami", Start: "2024-07-10".

## 7. Context Assembly & Prompt Construction
The backend takes the semantic memories and the relational SQL data and dynamically constructs the Working Memory (System Prompt).

**Assembled System Prompt sent to LLM:**
```text
You are an AI Travel Assistant.
Answer the user's question using ONLY the context provided below.

--- USER CONTEXT ---
Relevant Memories:
- User prefers Marriott hotels.
- User needs wheelchair-accessible rooms.

Upcoming Trips:
- Summer in Miami (Starts: 2024-07-10)

--- USER PROMPT ---
Can you recommend a hotel for my next trip?
```

## 8. Response Generation
The LLM processes the constructed prompt and generates:
*"Since you are heading to Miami on July 10th, I recommend the Miami Marriott Biscayne Bay. I've ensured they have excellent wheelchair-accessible rooms available for your dates."*

## 9. Architecture Diagram
```mermaid
flowchart TD
    User([User]) -->|Prompt| API[Backend API]
    API -->|1. Embed Query| OpenAI[OpenAI Embedding API]
    OpenAI -->> API: Vector
    
    API -->|2. Search LTM| PGV[(PostgreSQL\npgvector)]
    API -->|3. Search Trips| PG[(PostgreSQL\nRelational)]
    
    PGV -->> API: "Prefers Marriott"
    PG -->> API: "Going to Miami"
    
    API -->|4. Assemble Prompt| LLM[Chat LLM]
    LLM -->> API: "I recommend the Miami Marriott..."
    API --> User
```

## 10. Best Practices
- **Parallel Queries:** Execute the `pgvector` search and the relational SQL queries simultaneously (using `asyncio.gather` in Python or `Promise.all` in Node.js) to cut latency in half.
- **Distance Thresholds:** If the best matching vector has a cosine distance of `0.85` (meaning it's highly unrelated to the prompt), discard it. Only inject memories with a distance `< 0.25` into the prompt.

## 11. Common Mistakes
- **No Metadata Filtering:** Failing to filter by `user_id` in the `pgvector` query. This is a catastrophic data breach, as the AI will retrieve and reveal another user's private memories.
- **Prompt Stuffing:** Retrieving 50 memories instead of the top 3, exceeding the LLM's context window and causing an expensive API error.

## 12. Security
Always use parameterized SQL queries when performing Metadata Filtering. Never directly interpolate the `user_id` into the SQL string.

## 13. Summary
The Retrieval Pipeline is a high-speed orchestrator that gathers semantic and relational data, combining them to provide the LLM with perfect context. When this Retrieval Pipeline is connected directly to the AI's response generation, it forms what the industry calls RAG (Retrieval-Augmented Generation), which we will cover next.
