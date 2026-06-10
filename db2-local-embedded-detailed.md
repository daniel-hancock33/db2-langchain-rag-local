# Db2 Vector Search with TO_EMBEDDING() - EAP Feature

This guide shows how to use **Db2's new EAP feature** for generating embeddings directly in SQL using the `TO_EMBEDDING()` function with a locally-hosted Granite model via llama.cpp.

---

## Prerequisites

- RHEL system with llama.cpp source at `/more_storage/models/llama.cpp`
- Db2 with vector support and EAP features enabled
- Granite embedding model: `granite-embedding-30m-english-Q6_K.gguf`

---

## Setup Steps

### Step 1 — Install Build Dependencies

llama.cpp must be compiled from source. First install the required system tools: `cmake` for the build system, and `gcc`/`g++` as the C/C++ compilers.

```bash
sudo dnf install cmake gcc g++ -y
```

### Step 2 — Clone llama.cpp

Download the llama.cpp source code into the models directory.

```bash
cd /more_storage/models
git clone https://github.com/ggerganov/llama.cpp
```

### Step 3 — Build llama.cpp

Configure and compile the project. The `-j$(nproc)` flag parallelises the build across all available CPU cores to speed things up.

```bash
cd /more_storage/models/llama.cpp
cmake -B build
cmake --build build --config Release -j$(nproc)
```

### Step 4 — Start the Embedding Server

Launch llama.cpp as an OpenAI-compatible HTTP server. The `--embedding` flag enables the `/v1/embeddings` endpoint, `--pooling cls` uses CLS token pooling (required for Granite), and `-ub 8192` sets the batch size.

```bash
cd /more_storage/models/llama.cpp
build/bin/llama-server \
  -m /more_storage/models/granite-embedding-30m-english-Q6_K.gguf \
  --embedding \
  --pooling cls \
  -ub 8192 \
  --port 8082
```

> Server will be available at `http://127.0.0.1:8082` — keep this running for all subsequent steps.

---

## Database Setup

### Step 5 — Connect to Db2

Open a Db2 CLI session and connect to your database.

```sql
CONNECT TO SAMPLE;
```

### Step 6 — Clean Up Any Existing Objects

Drop the external model and table if they exist from a previous run, to start fresh.

```sql
DROP EXTERNAL MODEL granite30;
DROP TABLE ANSWERS;
```

### Step 7 — Create the Vector Table

Create a table to store text content alongside its vector embedding. The `embedding` column uses Db2's `VECTOR` type with 384 dimensions (matching Granite's output) and 32-bit float precision.

```sql
CREATE TABLE ANSWERS (
    id        INT NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1 INCREMENT BY 1),
    content   CLOB(100),
    embedding VECTOR(384, FLOAT32),
    PRIMARY KEY (id)
);
```

### Step 8 — Insert Sample Data

Populate the table with sample sentences about Toronto. Embeddings are left `NULL` for now — they will be generated in a later step.

```sql
INSERT INTO ANSWERS (content, embedding) VALUES
  ('Toronto is the most populated city in Canada, with millions of residents.', NULL),
  ('The skyline of Toronto is dominated by a tall observation tower visited by tourists worldwide.', NULL),
  ('The local basketball team became national champions in 2019, making the city proud.', NULL),
  ('Travelers flying internationally often depart from Pearson, the main airport of the city.', NULL),
  ('Toronto lies along the edge of Lake Ontario, giving it a waterfront character.', NULL);
```

### Step 9 — Register the External Embedding Model

Tell Db2 about the llama.cpp server using `CREATE EXTERNAL MODEL`. This registers the Granite model under the alias `granite30`, pointing to the running server's embeddings endpoint. The `PROVIDER OPENAI` clause means Db2 will use the OpenAI-compatible API format that llama.cpp exposes.

```sql
CREATE EXTERNAL MODEL granite30
  PROVIDER OPENAI
  ID 'granite-embedding-30m-english-Q6_K.gguf'
  TYPE TEXT_EMBEDDING RETURNING VECTOR(384, FLOAT32)
  URL 'http://127.0.0.1:8082/v1/embeddings';
```

---

## Generating & Querying Embeddings

### Step 10 — Generate Embeddings with TO_EMBEDDING()

Use Db2's `TO_EMBEDDING()` EAP function to call the external model for each row. This sends each `content` value to the llama.cpp server and stores the returned vector back into the `embedding` column.

```sql
UPDATE ANSWERS SET embedding = TO_EMBEDDING(content USING granite30);
```

### Step 11 — Verify the Embeddings

Confirm embeddings were stored by inspecting the first row. The vector is cast to VARCHAR and truncated for readability.

```sql
SELECT
  id,
  content,
  SUBSTR(CAST(embedding AS VARCHAR(2000)), 1, 200) || '...' AS vector_sample
FROM ANSWERS
FETCH FIRST 1 ROWS ONLY;
```

### Step 12 — Run a Vector Similarity Search

Search for the most semantically similar rows to a natural language question. `TO_EMBEDDING()` converts the query string to a vector on the fly, then `VECTOR_DISTANCE()` computes Euclidean distance against all stored embeddings. Lower distance = more similar.

```sql
SELECT
  id,
  content AS CONTEXT,
  VECTOR_DISTANCE(
    embedding,
    TO_EMBEDDING('Which towering structure shapes Toronto''s skyline and draws many visitors?' USING granite30),
    EUCLIDEAN
  ) AS DISTANCE
FROM ANSWERS
ORDER BY DISTANCE ASC
FETCH FIRST 2 ROWS ONLY;
```

---

## Cleanup

Remove the registered model when done. The table and data will persist unless explicitly dropped.

```sql
DROP EXTERNAL MODEL granite30;
```

---

## Key Points

- **`TO_EMBEDDING()`** calls the external model directly from SQL — no application-layer embedding code needed
- The llama.cpp server must be running for any operation that calls `TO_EMBEDDING()`
- Distance strategies: `COSINE`, `EUCLIDEAN`, or `MANHATTAN` — choose based on your model's recommendations
- Vector dimension (`384`) must exactly match the model's output size
- The `PROVIDER OPENAI` clause works with any OpenAI-compatible server, including llama.cpp
