# Db2 Vector Search with TO_EMBEDDING() - EAP Feature

This guide shows how to use **Db2's new EAP feature** for generating embeddings directly in SQL using the `TO_EMBEDDING()` function with a locally-hosted Granite model via llama.cpp.

## Prerequisites

- RHEL system with llama.cpp installed at `/more_storage/`
- Db2 with vector support and EAP features enabled
- Granite embedding model: `granite-embedding-30m-english-Q6_K.gguf`

## Setup Steps

### 1. Start llama.cpp Server

```bash
cd /more_storage/llama.cpp
build/bin/llama-server -m granite-embedding-30m-english-Q6_K.gguf --embedding --pooling cls -ub 8192
```

Server will run on `http://127.0.0.1:8080`

### 1. Connect to Db2:
```sql
CONNECT TO SAMPLE;
```

### 1. Drop table if it already exists:
```sql
DROP EXTERNAL MODEL granite30;
DROP TABLE ANSWERS;
```

### 2. Create Table

```sql
CREATE TABLE ANSWERS (
    id INT NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1 INCREMENT BY 1),
    content CLOB (100), 
    embedding VECTOR(384, FLOAT32),
    PRIMARY KEY (id)
);
```

### 3. Insert Sample Data

```sql
INSERT INTO ANSWERS (content, embedding) VALUES
  ('Toronto is the most populated city in Canada, with millions of residents.', NULL),
  ('The skyline of Toronto is dominated by a tall observation tower visited by tourists worldwide.', NULL),
  ('The local basketball team became national champions in 2019, making the city proud.', NULL),
  ('Travelers flying internationally often depart from Pearson, the main airport of the city.', NULL),
  ('Toronto lies along the edge of Lake Ontario, giving it a waterfront character.', NULL);
```

### 4. Register External Model

```sql
CREATE EXTERNAL MODEL granite30 
PROVIDER OPENAI 
ID 'granite-embedding-30m-english-Q6_K.gguf' 
TYPE TEXT_EMBEDDING RETURNING VECTOR(384, FLOAT32) 
URL 'http://127.0.0.1:8080/v1/embeddings';
```

### 5. Generate Embeddings with TO_EMBEDDING()

```sql
UPDATE ANSWERS SET embedding = TO_EMBEDDING(content USING granite30);
```

Verify embeddings were created:

```sql
SELECT 
  id,
  content,
  SUBSTR(CAST(embedding AS VARCHAR(2000)), 1, 200) || '...' AS vector_sample
FROM ANSWERS 
FETCH FIRST 1 ROWS ONLY;
```

### 6. Search with Vector Similarity

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

## Cleanup

```sql
DROP EXTERNAL MODEL granite30;
```

## Key Points

- **TO_EMBEDDING()** generates embeddings directly in SQL (Db2 EAP feature)
- Keep llama.cpp server running for all embedding operations
- Use COSINE, EUCLIDEAN, or MANHATTAN distance for similarity search
- Vector dimension (384) must match model output
