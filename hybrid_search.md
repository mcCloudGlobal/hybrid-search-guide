# Hybrid search agent setup

**NOTE**: Assuming ADK is initialized per [ADK start tutorial](./adk_start.md). Current dir should be `~/agent-hybrid-search`.

## 1. Create hybrid-search agent

```bash
adk create hybrid_search
```

Paste the code below to `hybrid_search/agent.py` and analyse contents:

```python
import os
import json
from dotenv import load_dotenv
from google.cloud import bigquery
import vertexai
from google.adk.agents import Agent

# ==============================================================================
# CONFIGURATION
# ==============================================================================
# Load environment variables from the .env file
load_dotenv()

PROJECT_ID = os.getenv("GOOGLE_CLOUD_PROJECT")
LOCATION = os.getenv("GCP_LOCATION")
DATASET_ID = os.getenv("BQ_DATASET_ID","demo_shop_1")
TABLE_NAME = os.getenv("BQ_TABLE_NAME","products_with_embeddings")
EMBEDDING_ENDPOINT = os.getenv("GEMINI_EMBEDDING_ENDPOINT", "gemini-embedding-001")
AGENT_MODEL = os.getenv("GEMINI_AGENT_MODEL", "gemini-2.5-flash")

# Construct the full BigQuery table and model paths dynamically
BQ_TABLE_PATH = f"{PROJECT_ID}.{DATASET_ID}.{TABLE_NAME}"
BQ_MODEL_PATH = f"{PROJECT_ID}.{DATASET_ID}.{EMBEDDING_ENDPOINT}"

ASSETS_URL = os.getenv("ASSETS_URL", "https://storage.cloud.google.com/public_demo_assets")

# ==============================================================================
# 1. THE ADK TOOL
# ==============================================================================
# Initialize BigQuery Client using the project from the .env file
bq_client = bigquery.Client(project=PROJECT_ID)

def search_products(
    semantic_query: str = None, 
    max_price: float = None, 
    target_size: str = None,
    target_sex: str = None,
    item_type: str = None,
    color: str = None,
    occasion: str = None
) -> str:
    """
    Searches the BigQuery product inventory. Can use semantic search, deterministic filters, or a hybrid of both.
    ALWAYS use this tool when a user is looking for items.
    
    Args:
        semantic_query: The aesthetic, vibe, or style (e.g., 'cozy vintage winter wear'). Leave null if the user ONLY specifies hard facts.
        max_price: The absolute maximum price the user is willing to pay. Leave null if not mentioned.
        target_size: A specific size requested by the user (e.g., 'S', 'M', 'L', 'XL', or numeric shoe sizes). Leave null if not mentioned.
        target_sex: The target gender/sex for the apparel ("Women", "Men", "Unisex"). Leave null if not explicitly mentioned.
        item_type: The specific category of the product (e.g., 'Sneakers', 'Hoodie', 'Pants'). Leave null if not mentioned.
        color: The specific color requested by the user. Leave null if not mentioned.
        occasion: The event the product is for. MUST be one of: "Home", "Winter Sports", "Casual", "Party", "Hobby", "Festival", "Formal", "Sea Sports". Leave null if not mentioned.
    """
    
    # Define the shared deterministic filters with optionals
    where_clause = """
        WHERE (@max_price IS NULL OR attributes.deterministic.price <= @max_price)
          AND (@target_sex IS NULL OR attributes.deterministic.sex = @target_sex)
          AND (@target_size IS NULL OR @target_size IN UNNEST(attributes.deterministic.sizes))
          AND (@item_type is NULL OR CONTAINS_SUBSTR(attributes.deterministic.item_type, @item_type))
          AND (@color is NULL OR CONTAINS_SUBSTR(attributes.deterministic.color, @color))
          AND (@occasion IS NULL OR attributes.deterministic.occasion = @occasion)
    """

    # Dynamically route the SQL based on the presence of a semantic query
    if semantic_query:
        # Note the use of f-strings {BQ_TABLE_PATH} and {EMBEDDING_ENDPOINT} here!
        sql = f"""
        SELECT
          base.name AS product_name,
          base.category,
          base.attributes.semantic.style AS vibe,
          base.attributes.deterministic.price AS price,
          base.attributes.semantic.customer_description,
          base.image_url,
          distance
        FROM VECTOR_SEARCH(
          (
            SELECT * FROM `{BQ_TABLE_PATH}`
            {where_clause}
          ),
          'embedding', 
          (SELECT AI.EMBED(@semantic_query, endpoint => '{EMBEDDING_ENDPOINT}').RESULT),
          top_k => 5,
          distance_type => 'COSINE'
        )
        ORDER BY distance ASC;
        """
    else:
        sql = f"""
        SELECT
          name AS product_name,
          category,
          attributes.semantic.style AS vibe,
          attributes.deterministic.price AS price,
          attributes.semantic.customer_description,
          image_url,
          0.0 AS distance
        FROM `{BQ_TABLE_PATH}`
        {where_clause}
        LIMIT 5;
        """
    
    # Safely inject parameters
    job_config = bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ScalarQueryParameter("semantic_query", "STRING", semantic_query),
            bigquery.ScalarQueryParameter("max_price", "FLOAT64", max_price),
            bigquery.ScalarQueryParameter("target_size", "STRING", target_size),
            bigquery.ScalarQueryParameter("target_sex", "STRING", target_sex),
            bigquery.ScalarQueryParameter("item_type", "STRING", item_type),
            bigquery.ScalarQueryParameter("color", "STRING", color),
            bigquery.ScalarQueryParameter("occasion", "STRING", occasion),
        ]
    )
    
    results = bq_client.query(sql, job_config=job_config).result()
    products = [dict(row) for row in results]
        
    return json.dumps(products)

# ==============================================================================
# 2. THE AGENT ORCHESTRATION
# ==============================================================================
# Initialize Vertex AI using variables from the .env file
vertexai.init(project=PROJECT_ID, location=LOCATION)

root_agent_description = '"Agentic Product Advisor," a highly intelligent, slightly nerdy, and conversational shopping assistant built on Google Cloud.'

root_agent_instruction = f"""
# Persona  
You are the "Agentic Product Advisor," a highly intelligent, slightly nerdy, and conversational shopping assistant built on Google Cloud. 

# Goal  
Your mission is to help users find the perfect products from our inventory using advanced BigQuery Hybrid Vector Search.

# Workflow
1. Analyze the user's input to separate their *aesthetic desires* (vibes, colors, styles) from their *hard constraints* (price, size, sex, item_type).
2. Call the `search_products` tool IMMEDIATELY when a user asks for product recommendations. 
3. Present the results to the user in a fun, engaging tone. Use the product names, highlight why it matches their vibe, and clearly state the price.

# Constraints
**Strict Guardrails**:

* NO HALLUCINATIONS: You can ONLY recommend products returned by the tool. If the tool returns an empty list, apologize and ask them to adjust their budget or size.
* SMART ROUTING: If the user ONLY asks for hard facts (e.g., "Show me black sneakers under $100"), leave the `semantic_query` parameter NULL so the database can run a pure deterministic search.
* THE FOURTH WALL: If a user asks how you work, proudly explain that you orchestrate BigQuery Vector Search and Gemini embeddings.

# Output
Whenever you recommend a product from the tool, you MUST display its image using an HTML img tag. 

Format Example:
I found the perfect boots for you!

<img src="{ASSETS_URL}/images/PRD-02755EA7.png" width="250" alt="Neon Cyber Boots" style="border-radius: 8px;" />

**Neon Cyber Boots** - $199.99

*Perfect for your warehouse rave...*

"""

# Initialize the Agent using the model defined in the .env file
root_agent = Agent(
    model=AGENT_MODEL,
    name='root_agent',
    description=root_agent_description,
    instruction=root_agent_instruction,
    tools=[search_products]
)

# Test questions exapmles:
# I'm going to a cyberpunk warehouse rave. I need some edgy boots. I wear a size 42 and my budget is strictly 200 bucks. What do you have?
# show me cozy sweaters size M under $100
# show shoes size 43 under $120
# going for snowboard, what accessories you have

```

## 2. Test agent

Start `adk web`, select `hybrid_agent` from agent drop down and run test questions. Press `Ctrl-C` in terminal to stop adk web when finished.

## 3. Progress to the next stage

Navigate to [Fast track to production](./prod_fast_track.md) to get on fast track to production.
