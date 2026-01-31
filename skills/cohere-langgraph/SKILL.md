---
name: cohere-langgraph
description: Cohere LangGraph agents reference for building ReAct agents, multi-tool workflows, agents with memory, and human-in-the-loop patterns. Covers both prebuilt and custom agent architectures.
---

# Cohere LangGraph Agents Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Installation

```bash
pip install langgraph langchain-cohere langchain
```

## Cohere ReAct Agent (Prebuilt)

```python
from langchain_cohere import ChatCohere, create_cohere_react_agent
from langchain_core.prompts import ChatPromptTemplate
from langchain.agents import AgentExecutor
from langchain_core.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get current weather for a location."""
    return f"Weather in {location}: 22°C, partly cloudy"

@tool
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression safely."""
    # In production, use a safe math parser like numexpr or sympy
    import ast
    try:
        # Only allow simple math operations
        tree = ast.parse(expression, mode='eval')
        return str(compile(tree, '<string>', 'eval'))
    except:
        return "Error: invalid expression"

llm = ChatCohere(model="command-a-03-2025")
tools = [get_weather, calculate]
prompt = ChatPromptTemplate.from_template("{input}")

agent = create_cohere_react_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True, max_iterations=10)

result = executor.invoke({"input": "What's the weather in Tokyo?"})
print(result["output"])
```

### When to Use Which Agent

**Use `create_cohere_react_agent` when:**
- Complex multi-step tasks
- Using Cohere-specific features (citations, connectors)
- Better token efficiency on long reasoning chains

**Generic `create_react_agent` is fine when:**
- Simple tool-calling workflows
- You want consistent behavior across providers

```python
# Cohere-specific
from langchain_cohere import create_cohere_react_agent
agent = create_cohere_react_agent(llm, tools, prompt)

# Generic (provider-agnostic)
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(llm, tools)
```

## Custom ReAct Agent with LangGraph

```python
from typing import Annotated, Sequence, TypedDict
from langchain_core.messages import BaseMessage, HumanMessage
from langchain_cohere import ChatCohere
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]

llm = ChatCohere(model="command-a-03-2025")
tools = [get_weather, calculate]
llm_with_tools = llm.bind_tools(tools)

def call_model(state: AgentState):
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

workflow = StateGraph(AgentState)
workflow.add_node("agent", call_model)
workflow.add_node("tools", ToolNode(tools))
workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", tools_condition, {"tools": "tools", "__end__": END})
workflow.add_edge("tools", "agent")

app = workflow.compile()

result = app.invoke({"messages": [HumanMessage(content="What's the weather in NYC?")]})
print(result["messages"][-1].content)
```

## Agent with Memory

```python
from langgraph.checkpoint.memory import InMemorySaver

workflow = StateGraph(AgentState)
workflow.add_node("agent", call_model)
workflow.add_node("tools", ToolNode(tools))
workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", tools_condition)
workflow.add_edge("tools", "agent")

memory = InMemorySaver()
app = workflow.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "user-123"}}

result1 = app.invoke(
    {"messages": [HumanMessage(content="My name is Veer")]},
    config=config
)

result2 = app.invoke(
    {"messages": [HumanMessage(content="What's my name?")]},
    config=config
)
# Agent remembers: "Your name is Veer"
```

## Streaming Agent Responses

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

## Human-in-the-Loop

```python
app = workflow.compile(
    checkpointer=InMemorySaver(),
    interrupt_before=["tools"]  # Pause before tool execution
)

config = {"configurable": {"thread_id": "review-session"}}

result = app.invoke(
    {"messages": [HumanMessage(content="Search for latest AI news")]},
    config=config
)

state = app.get_state(config)
print(f"Pending tool calls: {state.values['messages'][-1].tool_calls}")

# User approves, continue execution
final = app.invoke(None, config=config)
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

@tool
def web_search(query: str) -> str:
    """Search the web for information."""
    return f"Search results for '{query}': Latest AI breakthroughs..."

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

class State(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]

llm = ChatCohere(model="command-a-03-2025")
tools = [web_search, multiply]
llm_with_tools = llm.bind_tools(tools)

def agent(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph = StateGraph(State)
graph.add_node("agent", agent)
graph.add_node("tools", ToolNode(tools))
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", tools_condition)
graph.add_edge("tools", "agent")

app = graph.compile(checkpointer=InMemorySaver())

def chat(message: str, thread_id: str = "default"):
    config = {"configurable": {"thread_id": thread_id}}
    result = app.invoke({"messages": [HumanMessage(content=message)]}, config=config)
    return result["messages"][-1].content

print(chat("What is 15 times 23?"))
```
