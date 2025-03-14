# Handoffs in OpenAI Agents SDK - Comprehensive Guide

Handoffs are a powerful feature of the OpenAI Agents SDK that enable the creation of multi-agent systems where specialized agents can handle different aspects of a complex task or conversation.

## What are Handoffs?

Handoffs allow one agent to delegate a task to another agent. This creates a seamless experience where different specialized agents can handle different parts of a conversation or task without the user needing to manage the transitions.

When a handoff occurs:
1. The current agent invokes a handoff function
2. The target agent is initialized 
3. Control transfers to the target agent
4. The target agent processes the context and generates a response
5. The conversation continues with the target agent

## Key Components

### Handoff Class

The `Handoff` class represents a potential handoff to another agent:

```python
@dataclass
class Handoff(Generic[TContext]):
    tool_name: str
    tool_description: str
    input_json_schema: dict[str, Any]
    on_invoke_handoff: Callable[[RunContextWrapper[Any], str], Awaitable[Agent[TContext]]]
    agent_name: str
    input_filter: HandoffInputFilter | None = None
    strict_json_schema: bool = True
```

Key attributes:
- `tool_name`: The name of the function that triggers the handoff (e.g., "transfer_to_support")
- `tool_description`: Description visible to the agent for when to use this handoff
- `input_json_schema`: Schema for any parameters needed during handoff
- `on_invoke_handoff`: Function called when handoff occurs, returns target agent
- `agent_name`: Name of the agent being handed off to
- `input_filter`: Optional function to modify what context is passed to the next agent

### HandoffInputData

Contains the data being passed during a handoff:

```python
@dataclass
class HandoffInputData:
    input_history: str | tuple[TResponseInputItem, ...]
    pre_handoff_items: tuple[RunItem, ...]
    new_items: tuple[RunItem, ...]
```

- `input_history`: The original input history before any processing
- `pre_handoff_items`: Items generated before the handoff
- `new_items`: Items generated during the handoff process

### HandoffInputFilter

A function type that filters what information passes between agents:

```python
HandoffInputFilter: TypeAlias = Callable[[HandoffInputData], HandoffInputData]
```

## Creating and Using Handoffs

### Basic Handoff

```python
from agents import agent, handoff

# Create two agents
general_agent = agent("You are a general assistant...")
tech_agent = agent("You are a technical specialist...")

# Create handoff
tech_support_handoff = handoff(
    tech_agent,
    tool_name_override="transfer_to_tech_support",
    tool_description_override="Transfer to technical support for technical questions"
)

# Add handoff to the general agent
general_agent_with_handoff = general_agent.with_handoffs([tech_support_handoff])

# Run the agent
result = runner.run(general_agent_with_handoff, "I need help with my computer")
```

### Handoff with Input Parameters

```python
from pydantic import BaseModel

class SupportHandoffInput(BaseModel):
    issue_category: str
    priority: int = 3

async def get_support_agent(context, input_json):
    input_data = json.loads(input_json)
    # Could dynamically select agent based on input
    return support_agent

support_handoff = handoff(
    on_handoff=get_support_agent,
    input_type=SupportHandoffInput,
    tool_name_override="transfer_to_support",
    tool_description_override="Transfer to support with category and priority"
)
```

### Custom Input Filtering

```python
def filter_personal_info(handoff_data: HandoffInputData) -> HandoffInputData:
    # Create a new list excluding sensitive items
    filtered_items = [
        item for item in handoff_data.new_items 
        if not contains_personal_info(item)
    ]
    
    # Return new data with filtered items
    return HandoffInputData(
        input_history=handoff_data.input_history,
        pre_handoff_items=handoff_data.pre_handoff_items,
        new_items=tuple(filtered_items)
    )

privacy_conscious_handoff = handoff(
    agent,
    input_filter=filter_personal_info
)
```

## Extension: Handoff Filters

The OpenAI Agents SDK includes extension utilities for handoffs:

### remove_all_tools

Filters out all tool items (file search, web search, function calls):

```python
from agents.extensions.handoff_filters import remove_all_tools

clean_handoff = handoff(
    agent,
    input_filter=remove_all_tools
)
```

## Handoff Prompt Extension

The SDK provides a recommended prompt prefix for agents that use handoffs:

```python
from agents.extensions.handoff_prompt import RECOMMENDED_PROMPT_PREFIX, prompt_with_handoff_instructions

# Add handoff instructions to your prompt
enhanced_prompt = prompt_with_handoff_instructions(my_original_prompt)
```

The prefix reminds the agent that transfers are handled seamlessly and should not be mentioned to users.

## Best Practices

1. **Clear Handoff Triggers**: Use descriptive tool names and descriptions
2. **Specialized Agents**: Design agents with focused capabilities for specific tasks
3. **Appropriate Context Passing**: Use input filters to control what context passes between agents
4. **Error Handling**: Implement proper error handling for handoff failures
5. **Testing**: Test handoffs thoroughly with various conversation paths
6. **User Experience**: Make handoffs seamless from the user's perspective

## Example: Multi-Specialist System

```python
# Create specialized agents
triage_agent = agent("You're the first point of contact. Determine the customer's needs.")
billing_agent = agent("You specialize in billing and payment issues.")
technical_agent = agent("You handle technical troubleshooting.")
sales_agent = agent("You help with product information and purchases.")

# Create handoffs
billing_handoff = handoff(billing_agent, tool_name_override="transfer_to_billing")
technical_handoff = handoff(technical_agent, tool_name_override="transfer_to_technical")
sales_handoff = handoff(sales_agent, tool_name_override="transfer_to_sales")

# Add handoffs to triage agent
triage_agent_with_handoffs = triage_agent.with_handoffs([
    billing_handoff, 
    technical_handoff,
    sales_handoff
])

# Run the system with the triage agent first
result = runner.run(triage_agent_with_handoffs, "Hello, I'm having trouble with my subscription")
```

This creates a system where the triage agent can route customers to the appropriate specialist agent based on their needs.
