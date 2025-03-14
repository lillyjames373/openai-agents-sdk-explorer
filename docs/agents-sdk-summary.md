# OpenAI Agents SDK - Summary

This document provides a comprehensive overview of the key components in the OpenAI Agents SDK.

## Core Components

### Agents

Agents are the central abstraction in the SDK. An agent:
- Has specific instructions (system prompt)
- Can use tools (functions, capabilities)
- Optionally defines an output schema
- Can handoff to other agents

Agents are created using the `agent()` function and can be run via a `Runner`.

### Runner

The execution engine that:
- Manages agent lifecycle
- Handles tool execution
- Processes handoffs between agents
- Returns results to the caller
- Supports streaming outputs

### Tools

Functions that agents can call to perform actions:
- Created using the `@tool` decorator
- Include schema information derived from type hints and docstrings
- Can be synchronous or asynchronous
- Can receive the run context

### Handoffs

Mechanisms for transferring control between agents:
- Created using the `handoff()` function
- Allow specialized agents to handle specific parts of a task
- Support optional filtering of context passed between agents
- Enable multi-agent systems with different specializations

## Monitoring & Observability

### Tracing

Comprehensive tracing capabilities:
- Traces - Represent complete workflows
- Spans - Track specific operations (agent activity, function calls, etc.)
- Processors - Handle exporting trace data
- Support for console output or backend service integration

### Results & Streaming

Managing outputs from agent runs:
- `RunResult` - Contains all items generated during a run
- Streaming support via async iterators
- Various output item types (messages, tool calls, handoffs)
- Usage statistics tracking

## Advanced Features

### Guardrails

Safety and control mechanisms:
- Input Guardrails - Check incoming content before processing
- Output Guardrails - Validate agent outputs
- Tripwires - Can halt execution when triggered
- Support for custom validation logic

### Context Management

Managing state across agent interactions:
- `RunContextWrapper` - Passes context to tools and guardrails
- Custom context types for domain-specific needs
- Usage statistics tracking
- Support for both synchronous and asynchronous patterns

### Model Settings

Configuration for underlying LLM behavior:
- Temperature control
- Top-p sampling
- Frequency and presence penalties
- Token limit settings
- Tool choice configuration

## Integration Components

### Model Interface

Abstraction over different model APIs:
- OpenAI Chat Completions model
- OpenAI Responses model
- Extensible for other model providers
- Tracing integration

## Structure and Error Handling

### Function Schema

Automatically derives JSON schema from Python functions:
- Parameter validation
- Type conversion
- Documentation extraction
- Support for Python type hints

### Exceptions

Structured error handling:
- `AgentsException` - Base class for all exceptions
- `ModelBehaviorError` - For unexpected model behavior
- `MaxTurnsExceeded` - For run limits
- `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered` - For guardrail violations
