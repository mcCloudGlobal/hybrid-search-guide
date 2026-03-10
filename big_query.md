# Hybrid search setup in BigQuery

To start this tutorial you need a Google Cloud project with billing enabled. Navigate to [Big Query Studio](https://console.cloud.google.com/bigquery) and make sure your project is selected in project selector in top leftcorner of the console.

## 1. Enable APIs

Open Cloud Shell by naviagting to [this URL](https://shell.cloud.google.com/) or by clicking `Activate Cloud Shell` icon in the top right corner of Cloud Console and follow with commands:

```bash
gcloud config set project YOUR_PROJECT

export GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project)

echo "Current project set to: $GOOGLE_CLOUD_PROJECT"

gcloud services enable aiplatform.googleapis.com bigquery.googleapis.com bigqueryconnection.googleapis.com --project="$GOOGLE_CLOUD_PROJECT"
```

If re-/authentication is required at any time follow with these commands:
```bash
gcloud config set project YOUR_PROJECT

gcloud auth application-default login
```

## 2. Create BQ dataset and import data

In Cloud Shell run:

```bash
gh repo clone mcCloudGlobal/hybrid-search-guide

cd hybrid-search-guide

export GCP_LOCATION="europe-west1"

bq mk --location=$GCP_LOCATION --dataset $GOOGLE_CLOUD_PROJECT:demo_shop_1

bq load --source_format=NEWLINE_DELIMITED_JSON --autodetect $GOOGLE_CLOUD_PROJECT:demo_shop_1.products ./products.jsonl
```

## 3. Create model endpoint

Read full reference [here](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-generate-embedding#supported-text-models).

In [Big Query Studio](https://console.cloud.google.com/bigquery), within your project, open new query window and execute followin query:

```sql
CREATE OR REPLACE MODEL `demo_shop_1.embedding_model`
REMOTE WITH CONNECTION DEFAULT
OPTIONS (ENDPOINT = 'gemini-embedding-001');
```

To understand how embeddings and semantic search work review the following [Overview article](https://docs.cloud.google.com/bigquery/docs/vector-search-intro).

## 4. Create table with embeddings

Read full reference [here](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-generate-embedding).

In new query window read, analyse and execute:

```sql
CREATE OR REPLACE TABLE `demo_shop_1.products_with_embeddings` AS
SELECT
  * EXCEPT(title, content, statistics, status),
FROM
  AI.GENERATE_EMBEDDING(
    MODEL `demo_shop_1.embedding_model`,
    (
      SELECT
        *, -- Pulls all your base product columns (id, name, attributes, etc.)

        -- 💡 BIGQUERY PRO-TIP: Alias your product name as 'title' 
        -- so the Vertex model uses it to weigh the embedding!
        name AS title,
        
        -- 🧠 The payload MUST be aliased as 'content'
        ARRAY_TO_STRING(
          [
            IF(category IS NOT NULL, CONCAT('Category: ', category), NULL),
            IF(attributes.deterministic.item_type IS NOT NULL, CONCAT('Item Type: ', attributes.deterministic.item_type), NULL),
            IF(attributes.deterministic.color IS NOT NULL, CONCAT('Color: ', attributes.deterministic.color), NULL),
            IF(attributes.deterministic.material IS NOT NULL, CONCAT('Material: ', attributes.deterministic.material), NULL),
            IF(attributes.semantic.style IS NOT NULL, CONCAT('Style: ', attributes.semantic.style), NULL),
            IF(attributes.semantic.vibe IS NOT NULL, CONCAT('Vibe: ', attributes.semantic.vibe), NULL),
            IF(attributes.semantic.product_concept IS NOT NULL, CONCAT('Concept: ', attributes.semantic.product_concept), NULL),
            IF(attributes.semantic.customer_description IS NOT NULL, CONCAT('Description: ', attributes.semantic.customer_description), NULL)
          ], 
          '\n'
        ) AS content
      FROM
        `demo_shop_1.products`
    ),
    STRUCT('RETRIEVAL_DOCUMENT' AS task_type) -- Confirms this is our searchable corpus
  );
```

## 5. Query example row

In new query window execute ans understand single row. Mind the `embedding` column in vector format:

```sql
SELECT * FROM `demo_shop_1.products_with_embeddings` LIMIT 1;
```

## 6. Example semantic match

Review code below that lets you search products by semantic match. Execute and anaylse results. VECTOR_SEARCH reference is [here](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/search_functions#vector_search) and AI.GENERATE_EMBEDDING reference is here.

```sql
SELECT
 query.content AS user_search_intent,
  base.name AS product_name,
  base.category,
  base.attributes.semantic.style AS vibe,
  base.attributes.deterministic.price AS price,
  base.attributes.semantic.customer_description,
  distance
FROM VECTOR_SEARCH(
  
  -- ARGUMENT 1: The base table - mind its a ref
  TABLE `demo_shop_1.products_with_embeddings`,
  
  -- ARGUMENT 2: The vector column in the base table
  'embedding', 
  
  -- ARGUMENT 3: The Query Vector (Scalar Subquery)
  (
    SELECT content, embedding 
    FROM AI.GENERATE_EMBEDDING(
      MODEL `demo_shop_1.embedding_model`,
      (SELECT 'cozy vintage sweater' AS content),
      STRUCT('RETRIEVAL_QUERY' AS task_type)
    )
  ),
  
  -- KWARGS: Semantic Search Knobs
  top_k => 5,
  distance_type => 'COSINE'
)
ORDER BY distance ASC;
```

## 7. Example hybrid search

Now combine powers of regular projection (SELECT query with WHERE clause) with semantic search powers:

```sql
SELECT
 query.content AS user_search_intent,
  base.name AS product_name,
  base.category,
  base.attributes.semantic.style AS vibe,
  base.attributes.deterministic.price AS price,
  base.attributes.semantic.customer_description,
  distance
FROM VECTOR_SEARCH(
  
  -- ARGUMENT 1: Deterministic pre-filtering (this time Common Table Expression)
  (  SELECT * FROM `demo_shop_1.products_with_embeddings`
  WHERE attributes.deterministic.price <= 100.0
    AND 'M' IN UNNEST(attributes.deterministic.sizes)),
  
  -- ARGUMENT 2: The vector column in the base table
  'embedding', 
  
  -- ARGUMENT 3: The Query Vector (Scalar Subquery)
  (
    SELECT content, embedding 
    FROM AI.GENERATE_EMBEDDING(
      MODEL `demo_shop_1.embedding_model`,
      (SELECT 'cozy vintage sweater' AS content),
      STRUCT('RETRIEVAL_QUERY' AS task_type)
    )
  ),
  
  -- KWARGS: Semantic Search Knobs
  top_k => 5,
  distance_type => 'COSINE'
)
ORDER BY distance ASC;
```

## 8. ALTERNATIVE WITH AI.EMBED - Preview

To use latest syntax (at time of writing, 6th March 2026, in Preview) apply [AI.EMBED](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-embed) instead of [AI.GENERATE_EMBEDDING](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-generate-embedding).

```sql
SELECT
  base.name AS product_name,
  base.category,
  base.attributes.semantic.style AS vibe,
  base.attributes.deterministic.price AS price,
  base.attributes.semantic.customer_description,
  distance
FROM VECTOR_SEARCH(
  
  -- ARGUMENT 1: The Base Table (Inline Pre-filtering)
  (  SELECT * FROM `demo_shop_1.products_with_embeddings`
  WHERE attributes.deterministic.price <= 100.0
    AND 'M' IN UNNEST(attributes.deterministic.sizes)),
  
  -- ARGUMENT 2: The vector column in the base table
  'embedding', 
  
  -- ARGUMENT 3: The Query Vector (Scalar Subquery)
  (SELECT AI.EMBED('cozy vintage sweater', endpoint => 'gemini-embedding-001').RESULT),
  
  -- KWARGS: Semantic Search Knobs
  top_k => 1, --return only 1 best match
  distance_type => 'EUCLIDEAN' --'EUCLIDEAN','DOT_PRODUCT','COSINE'
)
ORDER BY distance ASC;
```

When ready proceed to [AI agents development in ADK](./adk_start.md) to start building agents in ADK.
