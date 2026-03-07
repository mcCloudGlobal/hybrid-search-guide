# AI agents development in ADK

**NOTE**: instruction created for Google Cloud Shell CLI.

## 1. Install ADK Python environment

 ```shell
 python -m venv .venv && source .venv/bin/activate  
 pip install google-adk
```

Reference [adk docs](https://google.github.io/adk-docs/get-started/installation/).

## 2. Create simple agent

Prepare Google Cloud project (copy project ID, e.g. from console URL), activate Vertex AI API. Full project setup instruction is [here](./big_query.md). Open Cloud Shell by naviagting to [this URL](https://shell.cloud.google.com/) or by clicking `Activate Cloud Shell` icon in the top right corner of Cloud Console and follow with commands:

Create and enter project folder:
```shell
cd ~/hybrid-search-guide
```

If folder does not exists review steps from [Hybrid search setup in BigQuery](./big_query.md).


Create simple_adk agent:
```shell
adk create simple_adk
```

**NOTE**: Only letters and underscores are allowed in agent name, otherwise **adk** fails during runtime.

**NOTE**: Run this command and follow the prompts when using Vertex AI to initiate ADC: 
```shell 
gcloud auth application-default login
```

Run adk web:
```shell
adk web --reload_agents
```

**NOTE**: Option ```--reload_agents``` will dynamically reload agents logic on *most* changes in agent code.

Open adk_web URL and verify if agent works. Test agent, for example by asking it to 

- *Count amount of countries in the world and list them in alphabetic order followed by actual population in millions. Add total people count by continent and overall.*
- *What is average Google Maps rating of restaurants with cost per person under $40 in Bydgoszcz?*
- *What is 123rd value in Fibonacci Sequence?*

Reference [adk docs](https://google.github.io/adk-docs/get-started/quickstart/#gemini---google-cloud-vertex-ai).

Press `Ctrl-C` in terminal to stop adk web when finished.

## 3. Add subagents with tools

**NOTE**: This example shows very useful built-in tools that require specific implementations due [ADK limitations](https://google.github.io/adk-docs/tools/limitations).

Create simple_adk agent:
```shell
adk create smart_adk
```

Replace ```smart_adk/agent.py``` contents with:

```python
from google.adk.tools.agent_tool import AgentTool
from google.adk.agents import Agent
from google.adk.tools import google_search
from google.adk.code_executors import BuiltInCodeExecutor
from google.adk.planners import BuiltInPlanner
from google.genai import types

search_agent = Agent(
    model='gemini-2.5-flash',
    name='SearchAgent',
    description='Web search tool',
    instruction="""
    You're a specialist in Google Search
    """,
    tools=[google_search],
)
coding_agent = Agent(
    model='gemini-2.5-flash',
    name='CodeAgent',
    description='Python code execution tool',
    instruction="""
    You're a specialist in Code Execution
    """,
    code_executor=BuiltInCodeExecutor(),
)
root_agent = Agent(
    model='gemini-2.5-flash',
    name='root_agent',
    description='A helpful assistant for user questions.',
    instruction='Answer user questions to the best of your knowledge and using tools available to you.',
    planner=BuiltInPlanner(
        thinking_config=types.ThinkingConfig(
            include_thoughts=True,
            thinking_budget=1024,
        )
    ),
    tools=[AgentTool(agent=search_agent), AgentTool(agent=coding_agent)],
)
```

Restart adk web if needed - Ctrl-C in console, then ```adk web --reload_agents```.  
Test agent using same examples as previously for comparison.

## 3. Proceed and create hybrid search agent

Navigate to [part 3](./hybrid_search.md) to create agent with hybrid search agent.
