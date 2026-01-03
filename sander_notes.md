
## 1. System Architecture
The application utilizes a **Retrieval-Augmented Generation (RAG)** architecture to provide voters with accurate, county-specific information from local politicians.

### The RAG Workflow
* **Retrieval:** When a user asks a question, the system generates a vector embedding and performs a similarity search in the database.
* **Augmentation:** The system retrieves the top human-readable text snippets (quotes) associated with the matching vectors.
* **Generation:** An LLM (e.g., GPT-4o) reads these human-readable quotes and synthesizes a neutral, cited response.

---

## 2. Tech Stack: Supabase & Nuxt.js

### Backend: Supabase (PostgreSQL + pgvector)
* **Vector Storage:** Uses the `pgvector` extension to store 1536-dimensional embeddings (OpenAI `text-embedding-3-small` standard).
* **Metadata Filtering:** Implements a strict `WHERE county_id = 'user-selected-county'` clause within the vector search function to prevent data leakage between regions.
* **Automation:** A database trigger fires a **Supabase Edge Function** on every insert, automatically generating embeddings via OpenAI and updating the row asynchronously.
* **Portability:** Standard Postgres architecture allows for full database dumps via Supabase CLI or `pg_dump`.

### Frontend: Nuxt.js (Vue 3)
* **Secure API Handling:** Uses **Nitro server routes** (`/server/api/search.post.ts`) to securely handle OpenAI API keys and Supabase service roles.
* **Real-time Interaction:** Employs reactive components to capture user input (county selection and query) and display AI-generated answers with source citations.

---

## 3. Database Schema & Search Logic
The core search logic resides in a PostgreSQL **Remote Procedure Call (RPC)** function:

```sql
CREATE OR REPLACE FUNCTION match_opinions (
  query_embedding vector(1536),
  filter_county_id text,
  match_count int
)
RETURNS TABLE (id uuid, content text, politician_name text, similarity float)
LANGUAGE plpgsql AS $$
BEGIN
  RETURN QUERY
  SELECT id, content, politician_name, 1 - (embedding <=> query_embedding)
  FROM politician_opinions
  WHERE county_id = filter_county_id
  ORDER BY embedding <=> query_embedding
  LIMIT match_count;
END;
$$;