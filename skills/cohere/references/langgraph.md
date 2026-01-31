# Cohere LangGraph Agents Reference

## Table of Contents
1. [Installation](#installation)
2. [Cohere ReAct Agent (Prebuilt)](#cohere-react-agent-prebuilt)
3. [Custom ReAct Agent](#custom-react-agent)
4. [Multi-Tool Agents](#multi-tool-agents)
5. [Agent with Memory](#agent-with-memory)
6. [Streaming Agent Responses](#streaming-agent-responses)
7. [Human-in-the-Loop](#human-in-the-loop)

## Installation

```bash
pip install langgraph langchain-cohere langchain
```

## Cohere ReAct Agent (Prebuilt)

The simplest way to create a Cohere agent using `create_cohere_react_agent`:

```python
from langchain_cohere import ChatCohere, create_cohere_react_agent
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_core.prompts import ChatPromptTemplate
from langchain.agents import AgentExecutor

# 1. Setup LLM
llm = ChatCohere(model="command-a-03-2025")

# 2. Define tools
search_tool = TavilySearchResults(max_results=3)
search_tool.name = "web_search"
search_tool.description = "Search the web for current information"

tools = [search_tool]

# 3. Create prompt
prompt = ChatPromptTemplate.from_template("{input}")

# 4. Create agent
agent = create_cohere_react_agent(llm, tools, prompt)

# 5. Create executor
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=10
)

# 6. Run
result = agent_executor.invoke({
    "input": "What are the latest developments in AI?"
})
print(result["output"])
```

### When to Use create_cohere_react_agent vs Generic create_react_agent

**Use `create_cohere_react_agent` when:**
- Complex multi-step tasks where Cohere's native planning shines
- Using Cohere-specific features like citations or connectors
- Better token efficiency on long reasoning chains
- You need Cohere's multi-hop reasoning capabilities

**Generic `create_react_agent` is fine when:**
- Simple tool-calling workflows (e.g., basic web scraping)
- You want consistent behavior across providers (easy to swap LLMs)
- You're already getting good results with the standard agent

```python
# Cohere-specific agent (optimized for Cohere)
from langchain_cohere import create_cohere_react_agent
agent = create_cohere_react_agent(llm, tools, prompt)

# Generic agent (provider-agnostic)
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(llm, tools)
```

### With Custom Tools

```python
from langchain_core.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get current weather for a location."""
    # Your implementation
    return f"Weather in {location}: 22°C, partly cloudy"

@tool
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression."""
    try:
        return str(eval(expression))
    except:
        return "Error evaluating expression"

@tool
def search_database(query: str) -> str:
    """Search the company database for information."""
    # Your implementation
    return f"Database results for '{query}': ..."

tools = [get_weather, calculate, search_database]

agent = create_cohere_react_agent(
    ChatCohere(model="command-a-03-2025"),
    tools,
    ChatPromptTemplate.from_template("{input}")
)

executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```

## Custom ReAct Agent

For more control, build a custom agent with LangGraph:

```python
from typing import Annotated, Sequence, TypedDict
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage, ToolMessage
from langchain_cohere import ChatCohere
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

# 1. Define state
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]

# 2. Setup LLM with tools
llm = ChatCohere(model="command-a-03-2025")
tools = [get_weather, calculate, search_database]
llm_with_tools = llm.bind_tools(tools)

# 3. Define agent node
def call_model(state: AgentState):
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# 4. Build graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("agent", call_model)
workflow.add_node("tools", ToolNode(tools))

# Set entry point
workflow.set_entry_point("agent")

# Add conditional edge
workflow.add_conditional_edges(
    "agent",
    tools_condition,  # Routes to "tools" if tool calls, else END
    {"tools": "tools", "__end__": END}
)

# Add edge from tools back to agent
workflow.add_edge("tools", "agent")

# Compile
app = workflow.compile()

# 5. Run
result = app.invoke({
    "messages": [HumanMessage(content="What's the weather in NYC?")]
})

print(result["messages"][-1].content)
```

## Multi-Tool Agents

### Parallel Tool Calling
Cohere models can call multiple tools in parallel:

```python
@tool
def get_stock_price(symbol: str) -> str:
    """Get current stock price."""
    return f"{symbol}: $150.00"

@tool
def get_company_info(symbol: str) -> str:
    """Get company information."""
    return f"{symbol}: Tech company, founded 1998"

@tool
def get_market_news(topic: str) -> str:
    """Get latest market news."""
    return f"News about {topic}: Market trending up"

tools = [get_stock_price, get_company_info, get_market_news]

agent = create_cohere_react_agent(
    ChatCohere(model="command-a-03-2025"),
    tools,
    ChatPromptTemplate.from_template("{input}")
)

executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# This may trigger parallel tool calls
result = executor.invoke({
    "input": "Give me a full analysis of AAPL stock"
})
```

### Multi-Step Reasoning
```python
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=15,  # Allow more steps for complex tasks
    early_stopping_method="generate"
)

result = executor.invoke({
    "input": "Compare the weather in Tokyo and London, then calculate the temperature difference"
})
```

## Agent with Memory

### Conversation Memory
```python
from langgraph.checkpoint.memory import InMemorySaver

# Build graph as before...
workflow = StateGraph(AgentState)
workflow.add_node("agent", call_model)
workflow.add_node("tools", ToolNode(tools))
workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", tools_condition)
workflow.add_edge("tools", "agent")

# Add memory
memory = InMemorySaver()
app = workflow.compile(checkpointer=memory)

# Run with thread ID for persistence
config = {"configurable": {"thread_id": "user-123"}}

# First message
result1 = app.invoke(
    {"messages": [HumanMessage(content="My name is Veer")]},
    config=config
)

# Second message (remembers context)
result2 = app.invoke(
    {"messages": [HumanMessage(content="What's my name?")]},
    config=config
)
```

### Persistent Memory (Redis/PostgreSQL)
```python
from langgraph.checkpoint.postgres import PostgresSaver

# PostgreSQL checkpointer
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
)

app = workflow.compile(checkpointer=checkpointer)
```

## Streaming Agent Responses

### Stream All Events
```python
async for event in app.astream_events(
    {"messages": [HumanMessage(content="Write a poem about AI")]},
    version="v2"
):
    kind = event["event"]
    
    if kind == "on_chat_model_stream":
        content = event["data"]["chunk"].content
        if content:
            print(content, end="", flush=True)
    
    elif kind == "on_tool_start":
        print(f"\nTool called: {event['name']}")
    
    elif kind == "on_tool_end":
        print(f"Tool result: {event['data'].get('output', '')[:100]}...")
```

### Stream Final Output Only
```python
for chunk in app.stream(
    {"messages": [HumanMessage(content="Hello")]},
    stream_mode="values"
):
    if chunk["messages"]:
        last_msg = chunk["messages"][-1]
        if hasattr(last_msg, "content") and last_msg.content:
            print(last_msg.content)
```

## Human-in-the-Loop

### Interrupt Before Tools
```python
from langgraph.prebuilt import ToolNode

# Build graph with interrupt
workflow = StateGraph(AgentState)
workflow.add_node("agent", call_model)
workflow.add_node("tools", ToolNode(tools))
workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", tools_condition)
workflow.add_edge("tools", "agent")

# Compile with interrupt
app = workflow.compile(
    checkpointer=InMemorySaver(),
    interrupt_before=["tools"]  # Pause before tool execution
)

config = {"configurable": {"thread_id": "review-session"}}

# Run until interrupt
result = app.invoke(
    {"messages": [HumanMessage(content="Search for latest AI news")]},
    config=config
)

# Check state
state = app.get_state(config)
print(f"Pending tool calls: {state.values['messages'][-1].tool_calls}")

# User approves, continue execution
final = app.invoke(None, config=config)
```

### Custom Approval Logic
```python
def should_continue(state: AgentState):
    last_message = state["messages"][-1]
    
    if not last_message.tool_calls:
        return "end"
    
    # Check if any tool needs approval
    sensitive_tools = ["delete_file", "send_email", "make_payment"]
    for tc in last_message.tool_calls:
        if tc["name"] in sensitive_tools:
            return "human_review"
    
    return "tools"

workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tools": "tools",
        "human_review": "human_review_node",
        "end": END
    }
)
```

## Complete Agent Example

```python
from typing import Annotated, Sequence, TypedDict
from langchain_core.messages import BaseMessage, HumanMessage
from langchain_core.tools import tool
from langchain_cohere import ChatCohere
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import InMemorySaver

# Tools
@tool
def web_search(query: str) -> str:
    """Search the web for information."""
    return f"Search results for '{query}': Latest AI breakthroughs..."

@tool
def calculator(expression: str) -> str:
    """Calculate mathematical expressions."""
    return str(eval(expression))

@tool
def get_current_time() -> str:
    """Get the current date and time."""
    from datetime import datetime
    return datetime.now().isoformat()

# State
class State(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]

# LLM
llm = ChatCohere(model="command-a-03-2025")
tools = [web_search, calculator, get_current_time]
llm_with_tools = llm.bind_tools(tools)

# Agent function
def agent(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

# Build graph
graph = StateGraph(State)
graph.add_node("agent", agent)
graph.add_node("tools", ToolNode(tools))
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", tools_condition)
graph.add_edge("tools", "agent")

# Compile with memory
app = graph.compile(checkpointer=InMemorySaver())

# Use
def chat(message: str, thread_id: str = "default"):
    config = {"configurable": {"thread_id": thread_id}}
    result = app.invoke(
        {"messages": [HumanMessage(content=message)]},
        config=config
    )
    return result["messages"][-1].content

# Interactive loop
while True:
    user_input = input("You: ")
    if user_input.lower() in ["quit", "exit"]:
        break
    response = chat(user_input)
    print(f"Agent: {response}")
```
