# Tools in OpenAI Agents SDK - Implementation Guide

Tools are a core component of the OpenAI Agents SDK, enabling agents to perform actions and access external systems. This guide covers how to implement, customize, and use tools effectively.

## What are Tools?

Tools in the Agents SDK are functions that an agent can call to:
- Access external systems or APIs
- Perform computations
- Retrieve or store data
- Execute actions in the real world

The SDK automatically handles the conversion between the agent's natural language interface and the structured function calls required by tools.

## Basic Tool Implementation

### Using the `@tool` Decorator

The simplest way to create a tool is with the `@tool` decorator:

```python
from agents import tool

@tool
def search_database(query: str) -> str:
    """
    Search for information in the database.
    
    Args:
        query: The search query string
        
    Returns:
        Search results as a string
    """
    # Implementation...
    return f"Results for query: {query}"
```

### Adding Tools to an Agent

Tools are provided when creating an agent:

```python
from agents import agent

my_agent = agent(
    "You are a helpful assistant with access to a database.",
    tools=[search_database]
)
```

## Tool Parameters and Types

Tools use Python type hints and docstrings to generate a schema that the agent uses to understand how to call the function.

### Simple Types

```python
@tool
def calculate_area(length: float, width: float) -> float:
    """
    Calculate the area of a rectangle.
    
    Args:
        length: The length of the rectangle in meters
        width: The width of the rectangle in meters
        
    Returns:
        The area in square meters
    """
    return length * width
```

### Complex Types with Pydantic

For more complex parameters, use Pydantic models:

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from agents import tool

class SearchFilter(BaseModel):
    keywords: List[str] = Field(description="List of keywords to search for")
    max_results: int = Field(default=10, description="Maximum number of results to return")
    category: Optional[str] = Field(default=None, description="Category to filter by")

@tool
def advanced_search(filters: SearchFilter) -> str:
    """
    Perform an advanced search with filters.
    
    Args:
        filters: The search filter parameters
        
    Returns:
        Search results as a string
    """
    # Implementation...
    return f"Found {filters.max_results} results for keywords: {', '.join(filters.keywords)}"
```

## Asynchronous Tools

Tools can be asynchronous for non-blocking operations:

```python
@tool
async def fetch_data(url: str) -> str:
    """
    Fetch data from a URL.
    
    Args:
        url: The URL to fetch data from
        
    Returns:
        The fetched data as a string
    """
    # Asynchronous implementation...
    await asyncio.sleep(1)  # Simulating network request
    return f"Data from {url}: [...]"
```

## Context-Aware Tools

Tools can access the run context to get state information:

```python
from agents import tool
from agents.run_context import RunContextWrapper

@tool
def get_user_preferences(context: RunContextWrapper, category: str) -> str:
    """
    Get the user's preferences for a category.
    
    Args:
        category: The preference category to retrieve
        
    Returns:
        The user's preferences for the category
    """
    # The context parameter is automatically injected
    user_id = context.context.get("user_id")
    # Implementation to fetch preferences...
    return f"Preferences for user {user_id} in category {category}: [...]"
```

## Tool Customization

### Name Override

You can override the tool name:

```python
@tool(name="search")
def search_database(query: str) -> str:
    """Search the database."""
    return f"Results for: {query}"
```

### Description Override

You can override the tool description:

```python
@tool(description="Search for information in the database with enhanced accuracy")
def search_database(query: str) -> str:
    """Search the database."""
    return f"Results for: {query}"
```

### Custom Docstring Parsing

Control how docstrings are parsed:

```python
from agents import tool, DocstringStyle

@tool(docstring_style=DocstringStyle.GOOGLE)
def search_database(query: str) -> str:
    """
    Search the database.
    
    Args:
        query: The search query
        
    Returns:
        Search results
    """
    return f"Results for: {query}"
```

## Tool Function Schema

The `function_schema` function extracts schema information from Python functions:

```python
from agents.function_schema import function_schema

# Get schema information from a function
schema = function_schema(
    search_database,
    name_override="search",
    description_override="Custom description",
    docstring_style=DocstringStyle.GOOGLE,
    strict_json_schema=True
)

# Access schema details
print(schema.name)  # "search"
print(schema.description)  # "Custom description"
print(schema.params_json_schema)  # JSON schema for parameters
```

## Tool Collections

Group related tools together for organization:

```python
# Define tool functions
@tool
def search_web(query: str) -> str:
    """Search the web."""
    return f"Web results for: {query}"

@tool
def search_images(query: str) -> str:
    """Search for images."""
    return f"Image results for: {query}"

@tool
def search_news(query: str) -> str:
    """Search for news."""
    return f"News results for: {query}"

# Create an agent with multiple tools
search_agent = agent(
    "You are a search assistant.",
    tools=[search_web, search_images, search_news]
)
```

## Error Handling in Tools

Implement proper error handling in tools:

```python
@tool
def divide(a: float, b: float) -> float:
    """
    Divide a by b.
    
    Args:
        a: The numerator
        b: The denominator
        
    Returns:
        The result of a divided by b
    """
    try:
        if b == 0:
            return "Error: Division by zero is not allowed"
        return a / b
    except Exception as e:
        return f"Error: {str(e)}"
```

## Tool Usage Patterns

### Chaining Tools

Agents can use multiple tools in sequence:

```python
# Agent might generate code like:
# 1. search_database("quantum physics")
# 2. summarize_text(results)
```

### Tool Output Processing

Agents can process tool outputs and make decisions:

```python
# Agent might generate code like:
# results = search_database("quantum physics")
# if "uncertainty principle" in results:
#     get_more_details("uncertainty principle")
# else:
#     search_database("uncertainty principle")
```

## Best Practices

1. **Clear Documentation**: Provide detailed docstrings with parameter descriptions
2. **Type Hints**: Use accurate type hints for all parameters and return values
3. **Error Handling**: Implement robust error handling in tools
4. **Atomic Tools**: Design tools to do one thing well
5. **Descriptive Names**: Use clear, descriptive function and parameter names
6. **Context Awareness**: Use context parameters when shared state is needed
7. **Performance**: Consider asynchronous implementation for I/O-bound operations
8. **Input Validation**: Validate inputs to prevent security issues
9. **Logging**: Include appropriate logging for debugging
10. **Testing**: Write tests for all tool functions

## Example: Building a Weather Tool Set

```python
from pydantic import BaseModel, Field
from agents import tool, agent
from typing import List, Optional
import asyncio

class Location(BaseModel):
    city: str = Field(description="The city name")
    country: Optional[str] = Field(default=None, description="The country name")

@tool
async def get_current_weather(location: Location) -> str:
    """
    Get the current weather for a location.
    
    Args:
        location: The location to get weather for
        
    Returns:
        Current weather information
    """
    # Simulating API call
    await asyncio.sleep(1)
    return f"Weather in {location.city}, {location.country or 'Unknown'}: 22°C, Partly Cloudy"

@tool
async def get_forecast(location: Location, days: int = 5) -> str:
    """
    Get a weather forecast for a location.
    
    Args:
        location: The location to get forecast for
        days: Number of days for the forecast (default: 5)
        
    Returns:
        Weather forecast information
    """
    # Simulating API call
    await asyncio.sleep(1)
    forecast = "Forecast:\n"
    for i in range(days):
        forecast += f"Day {i+1}: 24°C, Sunny\n"
    return forecast

@tool
def convert_temperature(celsius: float, to_unit: str = "fahrenheit") -> str:
    """
    Convert temperature between units.
    
    Args:
        celsius: Temperature in Celsius
        to_unit: Target unit (fahrenheit or kelvin)
        
    Returns:
        Converted temperature
    """
    if to_unit.lower() == "fahrenheit":
        return f"{celsius}°C = {celsius * 9/5 + 32}°F"
    elif to_unit.lower() == "kelvin":
        return f"{celsius}°C = {celsius + 273.15}K"
    else:
        return f"Unknown unit: {to_unit}. Supported units are 'fahrenheit' and 'kelvin'."

# Create a weather assistant agent
weather_assistant = agent(
    "You are a helpful weather assistant. You can provide current weather, " 
    "forecasts, and convert temperatures between units.",
    tools=[get_current_weather, get_forecast, convert_temperature]
)
```

This example creates a weather assistant with three tools: getting current weather, retrieving a forecast, and converting temperatures between units.
