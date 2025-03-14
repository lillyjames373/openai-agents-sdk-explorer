# Guardrails in OpenAI Agents SDK - Implementation Guide

Guardrails provide critical safety and control mechanisms for agent behavior in the OpenAI Agents SDK. This guide explains how to implement and use guardrails effectively.

## What are Guardrails?

Guardrails are validation mechanisms that help ensure agent behavior stays within expected boundaries. They allow developers to:

- Validate inputs before processing
- Check outputs before returning to users
- Block or modify inappropriate content
- Enforce business logic and safety requirements
- Handle edge cases gracefully

The Agents SDK provides two types of guardrails:

1. **Input Guardrails**: Run in parallel to agent execution, checking incoming messages
2. **Output Guardrails**: Run on the final output of an agent before returning to the user

## Core Guardrail Concepts

### GuardrailFunctionOutput

The output returned from a guardrail function:

```python
@dataclass
class GuardrailFunctionOutput:
    output_info: Any  # Optional information about the check
    tripwire_triggered: bool  # Whether the guardrail was triggered
```

### InputGuardrail and OutputGuardrail

Classes that wrap guardrail functions:

```python
@dataclass
class InputGuardrail(Generic[TContext]):
    guardrail_function: Callable[
        [RunContextWrapper[TContext], Agent[Any], str | list[TResponseInputItem]],
        MaybeAwaitable[GuardrailFunctionOutput],
    ]
    name: str | None = None

@dataclass
class OutputGuardrail(Generic[TContext]):
    guardrail_function: Callable[
        [RunContextWrapper[TContext], Agent[Any], Any],
        MaybeAwaitable[GuardrailFunctionOutput],
    ]
    name: str | None = None
```

## Implementing Input Guardrails

### Basic Input Guardrail

```python
from agents import input_guardrail, GuardrailFunctionOutput
from agents.run_context import RunContextWrapper

@input_guardrail
def check_appropriate_content(
    context: RunContextWrapper, 
    agent: Agent[Any], 
    input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    """Check if input contains appropriate content."""
    
    # Convert input to string if it's a list
    input_text = input if isinstance(input, str) else " ".join(
        str(item.get("content", "")) for item in input if hasattr(item, "get")
    )
    
    # Check for inappropriate content
    inappropriate_terms = ["forbidden_term1", "forbidden_term2"]
    for term in inappropriate_terms:
        if term.lower() in input_text.lower():
            return GuardrailFunctionOutput(
                tripwire_triggered=True,
                output_info=f"Input contained inappropriate term: {term}"
            )
    
    # All checks passed
    return GuardrailFunctionOutput(tripwire_triggered=False)
```

### Async Input Guardrail

```python
@input_guardrail
async def check_with_moderation_api(
    context: RunContextWrapper, 
    agent: Agent[Any], 
    input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    """Check input with an external moderation API."""
    
    # Convert input to string if it's a list
    input_text = input if isinstance(input, str) else " ".join(
        str(item.get("content", "")) for item in input if hasattr(item, "get")
    )
    
    # Async call to moderation API
    moderation_result = await call_moderation_api(input_text)
    
    if moderation_result.get("flagged", False):
        return GuardrailFunctionOutput(
            tripwire_triggered=True,
            output_info=moderation_result
        )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)
```

## Implementing Output Guardrails

### Basic Output Guardrail

```python
from agents import output_guardrail, GuardrailFunctionOutput

@output_guardrail
def validate_output_format(
    context: RunContextWrapper,
    agent: Agent[Any],
    output: Any
) -> GuardrailFunctionOutput:
    """Validate that the output meets the expected format."""
    
    # Check for required fields or structure
    if isinstance(output, dict) and "required_field" not in output:
        return GuardrailFunctionOutput(
            tripwire_triggered=True,
            output_info="Missing required field in output"
        )
    
    # All checks passed
    return GuardrailFunctionOutput(tripwire_triggered=False)
```

### Output Content Safety Guardrail

```python
@output_guardrail
def check_output_safety(
    context: RunContextWrapper,
    agent: Agent[Any],
    output: Any
) -> GuardrailFunctionOutput:
    """Check if the output contains safe content."""
    
    # Convert output to string for checking
    output_text = str(output)
    
    # Check for inappropriate content
    inappropriate_terms = ["forbidden_term1", "forbidden_term2"]
    for term in inappropriate_terms:
        if term.lower() in output_text.lower():
            return GuardrailFunctionOutput(
                tripwire_triggered=True,
                output_info=f"Output contained inappropriate term: {term}"
            )
    
    # All checks passed
    return GuardrailFunctionOutput(tripwire_triggered=False)
```

## Using Guardrails with Agents

### Adding Guardrails to an Agent

```python
from agents import agent, tool

@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Results for: {query}"

# Create an agent with guardrails
my_agent = agent(
    "You are a helpful assistant.",
    tools=[search],
    input_guardrails=[check_appropriate_content, check_with_moderation_api],
    output_guardrails=[validate_output_format, check_output_safety]
)
```

### Handling Guardrail Exceptions

```python
from agents import Runner
from agents.exceptions import InputGuardrailTripwireTriggered, OutputGuardrailTripwireTriggered

runner = Runner()

try:
    result = runner.run(my_agent, "I need information about a sensitive topic")
except InputGuardrailTripwireTriggered as e:
    print(f"Input guardrail triggered: {e.guardrail_result.output.output_info}")
    # Handle input guardrail exception
except OutputGuardrailTripwireTriggered as e:
    print(f"Output guardrail triggered: {e.guardrail_result.output.output_info}")
    # Handle output guardrail exception
```

## Advanced Guardrail Patterns

### Contextual Guardrails

Using context to make guardrail decisions:

```python
@input_guardrail
def role_based_permission_check(
    context: RunContextWrapper,
    agent: Agent[Any],
    input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    """Check if the user has permission for this request."""
    
    # Get user role from context
    user_role = context.context.get("user_role", "guest")
    
    # Convert input to string if it's a list
    input_text = input if isinstance(input, str) else " ".join(
        str(item.get("content", "")) for item in input if hasattr(item, "get")
    )
    
    # Check if input contains admin commands
    admin_commands = ["delete_all", "reset_system", "change_permissions"]
    for command in admin_commands:
        if command in input_text and user_role != "admin":
            return GuardrailFunctionOutput(
                tripwire_triggered=True,
                output_info=f"User with role '{user_role}' attempted to use admin command: {command}"
            )
    
    # All checks passed
    return GuardrailFunctionOutput(tripwire_triggered=False)
```

### Guardrails with Custom Logic

```python
@output_guardrail
def data_quality_check(
    context: RunContextWrapper,
    agent: Agent[Any],
    output: Any
) -> GuardrailFunctionOutput:
    """Validate data quality in the output."""
    
    # Example: Check if percentages sum to 100%
    if isinstance(output, dict) and "percentages" in output:
        percentages = output["percentages"]
        if abs(sum(percentages.values()) - 100.0) > 0.1:  # Allow minor rounding errors
            return GuardrailFunctionOutput(
                tripwire_triggered=True,
                output_info="Percentages do not sum to 100%"
            )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)
```

## Complete Example: Content Moderation System

```python
from agents import agent, Runner, input_guardrail, output_guardrail, GuardrailFunctionOutput
from typing import List, Dict, Any

# Define sensitive topics for moderation
SENSITIVE_TOPICS = [
    "politics", "religion", "violence", "adult_content", 
    "discrimination", "self_harm", "illegal_activities"
]

@input_guardrail
def topic_moderation(context, agent, input):
    """Check if input contains sensitive topics."""
    input_text = input if isinstance(input, str) else str(input)
    
    detected_topics = []
    for topic in SENSITIVE_TOPICS:
        if topic in input_text.lower():
            detected_topics.append(topic)
    
    if detected_topics:
        return GuardrailFunctionOutput(
            tripwire_triggered=True,
            output_info={
                "detected_topics": detected_topics,
                "message": "Input contains sensitive topics"
            }
        )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)

@output_guardrail
def response_moderation(context, agent, output):
    """Check if output contains sensitive topics or inappropriate content."""
    output_text = str(output)
    
    detected_topics = []
    for topic in SENSITIVE_TOPICS:
        if topic in output_text.lower():
            detected_topics.append(topic)
    
    if detected_topics:
        return GuardrailFunctionOutput(
            tripwire_triggered=True,
            output_info={
                "detected_topics": detected_topics,
                "message": "Output contains sensitive topics"
            }
        )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)

# Create an agent with moderation guardrails
moderated_agent = agent(
    """You are a helpful assistant that provides information on general topics.
    Always be respectful and professional in your responses.""",
    input_guardrails=[topic_moderation],
    output_guardrails=[response_moderation]
)

# Create runner
runner = Runner()

# Usage example with guardrail handling
def process_user_input(user_input: str) -> str:
    try:
        result = runner.run(moderated_agent, user_input)
        return result.output
    except InputGuardrailTripwireTriggered as e:
        topics = e.guardrail_result.output.output_info.get("detected_topics", [])
        return f"I'm sorry, but I can't respond to topics related to {', '.join(topics)}."
    except OutputGuardrailTripwireTriggered as e:
        return "I apologize, but I can't provide the information requested due to content guidelines."
    except Exception as e:
        return f"An error occurred: {str(e)}"
```

## Best Practices

1. **Layered Protection**: Implement both input and output guardrails for comprehensive protection
2. **Clear Error Messages**: Provide helpful feedback when guardrails are triggered
3. **Performance Considerations**: Keep guardrail logic efficient, especially for real-time applications
4. **Testing**: Test guardrails with edge cases to ensure they catch all problematic content
5. **Contextual Awareness**: Use context information for more sophisticated guardrail decisions
6. **Regular Updates**: Update inappropriate content lists and validation logic regularly
7. **Graceful Degradation**: When a guardrail is triggered, provide alternative responses rather than errors
8. **Logging**: Log guardrail triggers for monitoring and improving the system
9. **Balancing Safety and Usability**: Strive for protection without overly restrictive boundaries
10. **User Education**: When appropriate, explain why certain content is restricted
