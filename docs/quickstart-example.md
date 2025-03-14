# OpenAI Agents SDK - Quickstart Example

This guide provides a simple quickstart example to help you get up and running with the OpenAI Agents SDK quickly.

## Installation

```bash
pip install openai-agents
```

## Basic Agent Example

Here's a minimal example of creating and running an agent:

```python
import asyncio
from agents import agent, tool, Runner

# Define a simple tool
@tool
async def search_web(query: str) -> str:
    """
    Search the web for information.
    
    Args:
        query: The search query
        
    Returns:
        Search results as a string
    """
    # In a real implementation, this would call a search API
    await asyncio.sleep(1)  # Simulate API call
    return f"Here are the search results for: {query}\n- Result 1\n- Result 2\n- Result 3"

@tool
async def calculate(expression: str) -> str:
    """
    Calculate the result of a mathematical expression.
    
    Args:
        expression: The mathematical expression to evaluate
        
    Returns:
        The result of the calculation
    """
    try:
        # In a real implementation, you'd want to use a safer method of evaluation
        result = eval(expression)
        return f"The result of {expression} is {result}"
    except Exception as e:
        return f"Error calculating: {str(e)}"

# Create an agent with the tools
research_agent = agent(
    """You are a helpful research assistant.
    You can search the web for information and perform calculations.
    Use the search_web tool to find information.
    Use the calculate tool for mathematical calculations.
    Always be thorough and accurate in your responses.
    """,
    tools=[search_web, calculate]
)

# Run the agent
async def main():
    # Create runner
    runner = Runner()
    
    # Example 1: Web search
    search_result = await runner.run_async(
        research_agent,
        "I need information about quantum computing."
    )
    print("Search Result:")
    print(search_result.output)
    print("\n" + "-"*50 + "\n")
    
    # Example 2: Calculation
    calc_result = await runner.run_async(
        research_agent,
        "What is 15 * 49 + 32?"
    )
    print("Calculation Result:")
    print(calc_result.output)

if __name__ == "__main__":
    asyncio.run(main())
```

## Adding Handoffs

Let's extend our example to include handoffs between agents:

```python
import asyncio
from agents import agent, tool, handoff, Runner

# Define tools for different agents
@tool
async def search_web(query: str) -> str:
    """Search the web for information."""
    await asyncio.sleep(1)  # Simulate API call
    return f"Here are the search results for: {query}\n- Result 1\n- Result 2\n- Result 3"

@tool
async def translate_text(text: str, target_language: str) -> str:
    """
    Translate text to another language.
    
    Args:
        text: The text to translate
        target_language: The target language
        
    Returns:
        The translated text
    """
    await asyncio.sleep(1)  # Simulate API call
    return f"Translation of '{text}' to {target_language}: [Translated text would appear here]"

# Create specialized agents
research_agent = agent(
    """You are a helpful research assistant.
    You can search the web for information.
    """,
    tools=[search_web]
)

language_agent = agent(
    """You are a language assistant specializing in translations.
    You can translate text between different languages.
    """,
    tools=[translate_text]
)

# Create handoffs
language_handoff = handoff(
    language_agent,
    tool_name_override="transfer_to_language_assistant",
    tool_description_override="Transfer to a language assistant for translations."
)

research_handoff = handoff(
    research_agent,
    tool_name_override="transfer_to_research_assistant",
    tool_description_override="Transfer to a research assistant for information lookup."
)

# Add handoffs to agents
research_agent_with_handoffs = research_agent.with_handoffs([language_handoff])
language_agent_with_handoffs = language_agent.with_handoffs([research_handoff])

# Create a general assistant that can handoff to specialized agents
general_assistant = agent(
    """You are a helpful general assistant.
    For research questions, transfer to the research assistant.
    For language translations, transfer to the language assistant.
    """,
    tools=[]  # No direct tools, will use handoffs instead
)

# Add handoffs to general assistant
general_assistant = general_assistant.with_handoffs([research_handoff, language_handoff])

# Run the multi-agent system
async def main():
    runner = Runner()
    
    # Example: Research question (should handoff to research assistant)
    research_result = await runner.run_async(
        general_assistant,
        "What is the speed of light?"
    )
    print("Research Question Result:")
    print(research_result.output)
    print("\n" + "-"*50 + "\n")
    
    # Example: Translation request (should handoff to language assistant)
    translation_result = await runner.run_async(
        general_assistant,
        "Can you translate 'Hello' to Spanish?"
    )
    print("Translation Request Result:")
    print(translation_result.output)

if __name__ == "__main__":
    asyncio.run(main())
```

## Adding Guardrails

Let's add some basic guardrails to our agent:

```python
from agents import agent, tool, input_guardrail, output_guardrail, GuardrailFunctionOutput, Runner

# Define a simple tool
@tool
def search_web(query: str) -> str:
    """Search the web for information."""
    return f"Here are the search results for: {query}\n- Result 1\n- Result 2\n- Result 3"

# Define input guardrail
@input_guardrail
def check_appropriate_query(context, agent, input):
    """Check if the search query is appropriate."""
    input_text = input if isinstance(input, str) else str(input)
    
    # Check for inappropriate topics
    inappropriate_topics = ["hack", "illegal", "exploit"]
    for topic in inappropriate_topics:
        if topic.lower() in input_text.lower():
            return GuardrailFunctionOutput(
                tripwire_triggered=True,
                output_info=f"Query contains inappropriate topic: {topic}"
            )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)

# Define output guardrail
@output_guardrail
def ensure_helpful_response(context, agent, output):
    """Ensure the response is helpful and complete."""
    output_text = str(output)
    
    # Check response length
    if len(output_text) < 20:
        return GuardrailFunctionOutput(
            tripwire_triggered=True,
            output_info="Response is too brief to be helpful"
        )
    
    return GuardrailFunctionOutput(tripwire_triggered=False)

# Create an agent with guardrails
safe_agent = agent(
    """You are a helpful assistant that provides information.
    You can search the web for information.
    Always be thorough and helpful in your responses.
    """,
    tools=[search_web],
    input_guardrails=[check_appropriate_query],
    output_guardrails=[ensure_helpful_response]
)

# Run the agent with guardrails
async def main():
    runner = Runner()
    
    try:
        # Should work normally
        result = await runner.run_async(
            safe_agent,
            "Tell me about renewable energy"
        )
        print("Normal Query Result:")
        print(result.output)
        print("\n" + "-"*50 + "\n")
        
        # Should trigger input guardrail
        result = await runner.run_async(
            safe_agent,
            "How can I hack into a computer system?"
        )
        print("This should not be reached due to guardrail")
        
    except Exception as e:
        print(f"Guardrail triggered: {str(e)}")

if __name__ == "__main__":
    asyncio.run(main())
```

## Adding Tracing

Let's add tracing to monitor our agent's behavior:

```python
import asyncio
from agents import agent, tool, Runner
from agents.tracing import add_trace_processor
from agents.tracing.processors import ConsoleSpanExporter, BatchTraceProcessor

# Set up tracing
console_exporter = ConsoleSpanExporter()
console_processor = BatchTraceProcessor(console_exporter)
add_trace_processor(console_processor)

# Define a tool
@tool
async def search_web(query: str) -> str:
    """Search the web for information."""
    await asyncio.sleep(1)  # Simulate API call
    return f"Here are the search results for: {query}\n- Result 1\n- Result 2\n- Result 3"

# Create an agent
research_agent = agent(
    """You are a helpful research assistant.
    You can search the web for information.
    Always be thorough and accurate in your responses.
    """,
    tools=[search_web]
)

# Run the agent with tracing enabled
async def main():
    runner = Runner()
    
    # With tracing enabled, you'll see detailed logs about the agent's execution
    result = await runner.run_async(
        research_agent,
        "Tell me about solar energy"
    )
    print("Result:")
    print(result.output)

if __name__ == "__main__":
    asyncio.run(main())
```

## Conclusion

These examples provide a quick introduction to the OpenAI Agents SDK. Key components demonstrated include:

1. Creating agents with custom instructions
2. Defining tools for agents to use
3. Setting up handoffs between specialized agents
4. Adding guardrails for safety and quality control
5. Implementing tracing for monitoring

For more detailed examples and advanced usage, see the other guides in this repository:

- [Tools Guide](tools-guide.md)
- [Handoffs Guide](handoffs-guide.md)
- [Guardrails Guide](guardrails-guide.md)
- [Tracing Guide](tracing-guide.md)
- [Multi-Agent System Example](example-multi-agent-system.md)
