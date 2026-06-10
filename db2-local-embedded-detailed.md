# Db2 Vector Search with TO_EMBEDDING() - EAP Feature

This guide shows how to use **Db2's new EAP feature** for generating embeddings directly in SQL using the `TO_EMBEDDING()` function with a locally-hosted Granite model via llama.cpp.

---

## Lab Environment Setup

### Environment Overview

This lab uses a single virtual machine on an IBM Cloud environment. All required components are pre-installed.

**Installed components:**

| Component | Version |
|---|---|
| Db2 AI Advanced Edition (Single Partition) | 12.1.5 |

**Pre-configured databases:** `demo_col`, `demo_row`

**Utility scripts:**

| Script | Purpose |
|---|---|
| `ghinfo` | Display environment details |

### Service Endpoints

| Service | Endpoint |
|---|---|
| **Db2 Host (for Genius Hub)** | `localhost` |
| **Db2 Host (for external tools)** | `YOUR-EXTERNAL-IP` |
| **Db2 Port** | `25011` |
| **SSH Access** | `ssh -i YOUR-FILE.pem YOUR-USER@YOUR-EXTERNAL-IP -p 2223` |

> Replace `YOUR-FILE.pem`, `YOUR-USER`, and `YOUR-EXTERNAL-IP` with the values provided for your lab environment.

### Default Credentials

**Genius Hub UI:**

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `Db2ghPassw0rd#1` |

**Db2 Users** (all share the same password):

| Username | Password |
|---|---|
| `db2inst1` | `Db2ghPassw0rd#1` |
| `db2demo` | `Db2ghPassw0rd#1` |

### SSH Access

Your instructor will provide your VM's public IP address and a personal PEM key file (e.g., `student_01.pem`). All students connect as the `db2demo` user.

**Step 1 — Set file permissions**

*Mac/Linux:*
```bash
cd ~/Desktop
chmod 600 student_01.pem
```

*Windows (PowerShell):*
```powershell
icacls student_01.pem /inheritance:r
icacls student_01.pem /grant:r "%USERNAME%:F"
```

*Windows (GUI):* Right-click the `.pem` file → Properties → Security → Advanced → Disable inheritance → Remove all inherited permissions → Add your Windows username with Full control.

**Step 2 — Connect**

*Mac/Linux/Windows PowerShell:*
```bash
ssh -i student_01.pem db2demo@52.118.191.168 -p 2223
```

*Windows (PuTTY):* Convert your `.pem` to `.ppk` using PuTTYgen (File → Load → Save private key), then open PuTTY with Host = your IP, Port = `2223`, and your `.ppk` under Connection → SSH → Auth.

On first connection, type `yes` when prompted about host authenticity. A successful login shows:
```
[db2demo@db2gh-demo ~]$
```

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

### Step 4 — Open the Firewall Port

Open the TCP port for the embedding server.

```bash
sudo firewall-cmd --permanent --add-port=8082/tcp
sudo firewall-cmd --reload
```

### Step 5 — Start the Embedding Server

Launch llama.cpp as an OpenAI-compatible HTTP server. The `--embedding` flag enables the `/v1/embeddings` endpoint, `--pooling cls` uses CLS token pooling (required for Granite), and `-ub 8192` sets the batch size.

```bash
cd /more_storage/models/llama.cpp
build/bin/llama-server \
  -m /more_storage/models/granite-embedding-30m-english-Q6_K.gguf \
  --embedding \
  --pooling cls \
  -ub 8192 \
  --port 8082 \
  --host 0.0.0.0
```

> Server will be available at `http://127.0.0.1:8082` — keep this running for all subsequent steps.

---

## Database Setup

All SQL commands are run as the `db2demo` local user via the `db2` CLI.

### Step 6 — Connect to Db2

As the OS user `db2demo` connect to the database `DEMO_ROW`.

```bash
db2 "CONNECT TO DEMO_ROW"
```

### Step 7 — Clean Up Any Existing Objects

Drop the external model and table if they exist from a previous run, to start fresh.

```bash
db2 "DROP EXTERNAL MODEL granite30"
db2 "DROP TABLE ANSWERS"
```

### Step 8 — Create the Vector Table

Create a table to store text content alongside its vector embedding. The `embedding` column uses Db2's `VECTOR` type with 384 dimensions (matching Granite's output) and 32-bit float precision.

```bash
db2 "CREATE TABLE ANSWERS (
    id        INT NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1 INCREMENT BY 1),
    content   CLOB(100),
    embedding VECTOR(384, FLOAT32),
    PRIMARY KEY (id)
)"
```

### Step 9 — Insert Sample Data

Populate the table with sample sentences about Toronto. Embeddings are left `NULL` for now — they will be generated in a later step.

```bash
db2 "INSERT INTO ANSWERS (content, embedding) VALUES
  ('Toronto is the most populated city in Canada, with millions of residents.', NULL),
  ('The skyline of Toronto is dominated by a tall observation tower visited by tourists worldwide.', NULL),
  ('The local basketball team became national champions in 2019, making the city proud.', NULL),
  ('Travelers flying internationally often depart from Pearson, the main airport of the city.', NULL),
  ('Toronto lies along the edge of Lake Ontario, giving it a waterfront character.', NULL)"
```

### Step 10 — Register the External Embedding Model

Tell Db2 about the llama.cpp server using `CREATE EXTERNAL MODEL`. This registers the Granite model under the alias `granite30`, pointing to the running server's embeddings endpoint. The `PROVIDER OPENAI` clause means Db2 will use the OpenAI-compatible API format that llama.cpp exposes.

```bash
db2 "CREATE EXTERNAL MODEL granite30
  PROVIDER OPENAI
  ID 'granite-embedding-30m-english-Q6_K.gguf'
  TYPE TEXT_EMBEDDING RETURNING VECTOR(384, FLOAT32)
  URL 'http://127.0.0.1:8082/v1/embeddings'"
```

---

## Generating & Querying Embeddings

### Step 11 — Generate Embeddings with TO_EMBEDDING()

Use Db2's `TO_EMBEDDING()` EAP function to call the external model for each row. This sends each `content` value to the llama.cpp server and stores the returned vector back into the `embedding` column.

```bash
db2 "UPDATE ANSWERS SET embedding = TO_EMBEDDING(content USING granite30)"
```

### Step 12 — Verify the Embeddings

Confirm embeddings were stored by inspecting the first row. The vector is cast to VARCHAR and truncated for readability.

```bash
db2 "SELECT id, content, SUBSTR(CAST(embedding AS VARCHAR(2000)), 1, 200) || '...' AS vector_sample
FROM ANSWERS
FETCH FIRST 1 ROWS ONLY"
```

### Step 13 — Run a Vector Similarity Search

Search for the most semantically similar rows to a natural language question. `TO_EMBEDDING()` converts the query string to a vector on the fly, then `VECTOR_DISTANCE()` computes Euclidean distance against all stored embeddings. Lower distance = more similar.

```bash
db2 "SELECT id, content AS CONTEXT,
  VECTOR_DISTANCE(
    embedding,
    TO_EMBEDDING('Which towering structure shapes Toronto''s skyline and draws many visitors?' USING granite30),
    EUCLIDEAN
  ) AS DISTANCE
FROM ANSWERS
ORDER BY DISTANCE ASC
FETCH FIRST 2 ROWS ONLY"
```

---

## Cleanup

Remove the registered model when done. The table and data will persist unless explicitly dropped.

```bash
db2 "DROP EXTERNAL MODEL granite30"
```

---

## Key Points

- **`TO_EMBEDDING()`** calls the external model directly from SQL — no application-layer embedding code needed
- The llama.cpp server must be running for any operation that calls `TO_EMBEDDING()`
- Distance strategies: `COSINE`, `EUCLIDEAN`, or `MANHATTAN` — choose based on your model's recommendations
- Vector dimension (`384`) must exactly match the model's output size
- The `PROVIDER OPENAI` clause works with any OpenAI-compatible server, including llama.cpp
