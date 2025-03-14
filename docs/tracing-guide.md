# Tracing in OpenAI Agents SDK - Comprehensive Guide

The OpenAI Agents SDK provides a robust tracing system that enables developers to observe, monitor, and debug agent behavior. This guide covers the key concepts and usage patterns for tracing.

## Key Concepts

### Traces

A trace represents a complete workflow or conversation. It is the root-level object in the tracing system and typically corresponds to a single agent run or conversation thread.

```python
class Trace:
    """A trace is the root level object that tracing creates. It represents a logical "workflow"."""
    
    @property
    def trace_id(self) -> str:
        """The trace ID."""
        ...
        
    @property
    def name(self) -> str:
        """The name of the workflow being traced."""
        ...
        
    def start(self, mark_as_current: bool = False):
        """Start the trace."""
        ...
        
    def finish(self, reset_current: bool = False):
        """Finish the trace."""
        ...
        
    def export(self) -> dict[str, Any] | None:
        """Export the trace as a dictionary."""
        ...
```

### Spans

Spans represent specific operations or events within a trace. Different types of spans capture various aspects of agent behavior:

- **Agent Spans**: Track agent activity
- **Function Spans**: Record tool function calls
- **Generation Spans**: Capture model generation details
- **Response Spans**: Track model responses
- **Handoff Spans**: Monitor transfers between agents
- **Custom Spans**: Allow for user-defined tracking
- **Guardrail Spans**: Record guardrail evaluations

```python
class Span(ABC, Generic[TSpanData]):
    def start(self, mark_as_current: bool = False):
        """Start the span."""
        ...
        
    def finish(self, reset_current: bool = False) -> None:
        """Finish the span."""
        ...
```

### Processors

Tracing processors handle the processing and exporting of trace data:

- **TracingProcessor**: Base interface for processing spans
- **TracingExporter**: Interface for exporting traces and spans
- **ConsoleSpanExporter**: Prints traces to the console
- **BackendSpanExporter**: Sends traces to a backend service
- **BatchTraceProcessor**: Processes traces in batches for efficiency

## Setting Up Tracing

### Basic Setup

```python
from agents.tracing import (
    add_trace_processor, 
    set_tracing_disabled, 
    set_tracing_export_api_key
)
from agents.tracing.processors import ConsoleSpanExporter, BatchTraceProcessor

# Create a console exporter
console_exporter = ConsoleSpanExporter()
console_processor = BatchTraceProcessor(console_exporter)

# Add the processor
add_trace_processor(console_processor)

# Set API key if using backend exporter
set_tracing_export_api_key("your-openai-api-key")

# Optionally disable tracing globally
# set_tracing_disabled(True)
```

### Creating and Using Traces

```python
from agents.tracing import trace, agent_span, function_span, custom_span

# Using a trace as a context manager
with trace("my_workflow", metadata={"user_id": "user123"}) as my_trace:
    # Do work within the trace
    pass

# Manual trace management
my_trace = trace("my_workflow")
my_trace.start()
try:
    # Do work within the trace
    pass
finally:
    my_trace.finish()
```

### Creating and Using Spans

```python
# Using agent span as a context manager
with trace("my_workflow") as my_trace:
    with agent_span("my_agent", tools=["search", "calculator"]) as agent_activity:
        # Agent activity happens here
        pass

# Function span for tool calls
with trace("my_workflow") as my_trace:
    with function_span("search", input="query", output="results") as func_span:
        # Function execution happens here
        pass

# Custom span for user-defined events
with trace("my_workflow") as my_trace:
    with custom_span("data_processing", data={"records": 100}) as custom_activity:
        # Custom activity happens here
        pass
```

## Specialized Span Types

### Generation Span

Captures details of a model generation:

```python
with generation_span(
    input=[{"role": "user", "content": "Hello"}],
    output=[{"role": "assistant", "content": "Hi there!"}],
    model="gpt-4o",
    model_config={"temperature": 0.7},
    usage={"prompt_tokens": 10, "completion_tokens": 5}
) as gen_span:
    # Model generation happens here
    pass
```

### Response Span

Captures a model response:

```python
with response_span(response=model_response) as resp_span:
    # Process model response
    pass
```

### Handoff Span

Tracks transfers between agents:

```python
with handoff_span(from_agent="triage", to_agent="support") as handoff_activity:
    # Handoff execution happens here
    pass
```

### Guardrail Span

Records guardrail evaluations:

```python
with guardrail_span("content_safety", triggered=False) as guardrail_check:
    # Guardrail evaluation happens here
    pass
```

## Advanced Features

### Trace and Span IDs

Generate IDs for traces and spans:

```python
from agents.tracing.util import gen_trace_id, gen_span_id

my_trace_id = gen_trace_id()
my_span_id = gen_span_id()

with trace("my_workflow", trace_id=my_trace_id) as my_trace:
    with agent_span("my_agent", span_id=my_span_id) as agent_activity:
        pass
```

### Getting Current Trace and Span

Access the currently active trace or span:

```python
from agents.tracing import get_current_trace, get_current_span

current_trace = get_current_trace()
current_span = get_current_span()
```

### Group Traces

Group related traces with a group ID:

```python
with trace("conversation", group_id="chat_thread_123") as trace1:
    # First part of conversation
    pass

with trace("conversation", group_id="chat_thread_123") as trace2:
    # Later part of same conversation
    pass
```

## Exporting and Processing

### Custom Processor

Create a custom trace processor:

```python
from agents.tracing.processor_interface import TracingProcessor
from agents.tracing.traces import Trace
from agents.tracing.spans import Span
from typing import Any

class MyCustomProcessor(TracingProcessor):
    def on_trace_start(self, trace: Trace) -> None:
        print(f"Trace started: {trace.name} ({trace.trace_id})")
        
    def on_trace_end(self, trace: Trace) -> None:
        print(f"Trace ended: {trace.name} ({trace.trace_id})")
        
    def on_span_start(self, span: Span[Any]) -> None:
        print(f"Span started: {span.__class__.__name__}")
        
    def on_span_end(self, span: Span[Any]) -> None:
        print(f"Span ended: {span.__class__.__name__}")
        
    def shutdown(self) -> None:
        print("Processor shutting down")
        
    def force_flush(self) -> None:
        print("Forcing flush")

# Add the custom processor
add_trace_processor(MyCustomProcessor())
```

### Backend Export

Export traces to OpenAI's backend:

```python
from agents.tracing.processors import BackendSpanExporter, BatchTraceProcessor

# Create a backend exporter
backend_exporter = BackendSpanExporter(
    api_key="your-openai-api-key",
    organization="your-org-id",
    project="your-project-id",
)

# Create a batch processor
batch_processor = BatchTraceProcessor(
    exporter=backend_exporter,
    max_batch_size=100,
    schedule_delay=5.0
)

# Add the processor
add_trace_processor(batch_processor)
```

## Integration with Agents SDK

The tracing system is integrated with the Agents SDK, providing automatic tracing for agent runs:

```python
from agents import agent, tool, Runner

@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Results for: {query}"

my_agent = agent(
    "You are a helpful assistant.",
    tools=[search]
)

runner = Runner()
# Tracing happens automatically in the background
result = runner.run(my_agent, "I need information about astronomy")
```

## Best Practices

1. **Use Descriptive Names**: Provide meaningful names for traces and spans to make debugging easier.
2. **Appropriate Metadata**: Include relevant metadata that will help with analysis.
3. **Proper Hierarchy**: Structure spans in a logical hierarchy that reflects the workflow.
4. **Error Handling**: Always ensure spans are finished, especially in error cases (use context managers).
5. **Performance Considerations**: Be mindful of the performance impact of extensive tracing.
6. **Data Privacy**: Be careful about what data is included in traces, especially when using backend exporters.
7. **Batch Processing**: Use BatchTraceProcessor for better performance.
8. **Grouping**: Use group IDs to correlate related traces.

## Conclusion

The tracing system in the OpenAI Agents SDK provides comprehensive observability into agent behavior. By leveraging traces, spans, and processors, developers can gain insights into agent execution, monitor performance, and debug issues effectively.
