# Integrating Web Search with OpenAI Agents SDK

This guide explains how to create an agent that can search the web using OpenAI's built-in web search capabilities through the Agents SDK.

## Prerequisites

- OpenAI API key with access to the Responses API
- OpenAI Agents SDK installed

## OpenAI's Web Search Tool

OpenAI provides a built-in web search capability through the `web_search_preview` tool in the Responses API. This tool allows models to search the web for the latest information before generating a response.

## Implementing a Web Search Agent

Here's how to implement a web search agent using the OpenAI Agents SDK:

```python
import asyncio
from openai import OpenAI
from agents import agent, tool, Runner
from typing import Dict, Any, List

# Create a wrapper tool around OpenAI's web search
@tool
async def search_web(query: str) -> str:
    """
    Search the web for the given query using OpenAI's web search tool.
    
    Args:
        query: The search query
        
    Returns:
        Search results as a formatted string
    """
    try:
        # Initialize the OpenAI client
        client = OpenAI()
        
        # Make the search request
        response = client.responses.create(
            model="gpt-4o",
            tools=[{"type": "web_search_preview"}],
            tool_choice={"type": "web_search_preview"},  # Force using web search
            input=query
        )
        
        # Process the response - this will depend on the response format
        # Assuming response is a list of items
        web_search_call = next((item for item in response if item.get("type") == "web_search_call"), None)
        message = next((item for item in response if item.get("type") == "message"), None)
        
        if not message or not message.get("content"):
            return "No search results found."
        
        content = message["content"][0]
        text = content.get("text", "")
        annotations = content.get("annotations", [])
        
        # Format the results with citations
        result = text
        if annotations:
            result += "\n\nSources:\n"
            for i, annotation in enumerate(annotations):
                if annotation.get("type") == "url_citation":
                    url = annotation.get("url", "No URL")
                    title = annotation.get("title", "No title")
                    result += f"{i+1}. {title} - {url}\n"
        
        return result
    except Exception as e:
        return f"Error during web search: {str(e)}"

# Create an agent that can search the web
search_agent = agent(
    """You are a helpful research assistant with the ability to search the web for the latest information.
    When asked a question, use the search_web tool to find up-to-date information.
    Always cite your sources by referencing the URLs provided in the search results.
    If the search doesn't return useful results, you can try reformulating the query and searching again.
    Prioritize recent and credible information from the search results.
    """,
    tools=[search_web]
)

# Example of running the agent
async def main():
    runner = Runner()
    
    query = "What are the latest developments in renewable energy technology in 2025?"
    print(f"Searching for: {query}")
    
    result = await runner.run_async(search_agent, query)
    print("\nAgent Response:")
    print(result.output)

if __name__ == "__main__":
    asyncio.run(main())
```

## Understanding the Response Format

The response from OpenAI's web search includes two main parts:

1. A `web_search_call` output item with the ID of the search call
2. A `message` output item containing:
   - The text result in `message.content[0].text`
   - Annotations `message.content[0].annotations` for the cited URLs

Example response structure:

```json
[
  {
    "type": "web_search_call",
    "id": "ws_67c9fa0502748190b7dd390736892e100be649c1a5ff9609",
    "status": "completed"
  },
  {
    "id": "msg_67c9fa077e288190af08fdffda2e34f20be649c1a5ff9609",
    "type": "message",
    "status": "completed",
    "role": "assistant",
    "content": [
      {
        "type": "output_text",
        "text": "Search results with content...",
        "annotations": [
          {
            "type": "url_citation",
            "start_index": 100,
            "end_index": 105,
            "url": "https://example.com",
            "title": "Example Site"
          }
        ]
      }
    ]
  }
]
```

## Forcing Web Search

You can force the use of the web search tool by setting the `tool_choice` parameter to `{"type": "web_search_preview"}`. This ensures the model will always use web search when processing the query, which can help with consistency and latency.

## Displaying Citations

When displaying web search results to end users, you should make inline citations clearly visible and clickable in your user interface. The annotations in the response provide the necessary information for creating these citations, including:

- URL of the source
- Title of the source
- Location in the text where the citation applies

## Benefits of Using OpenAI's Built-in Web Search

1. **Simplicity**: No need to integrate with third-party search APIs
2. **Consistency**: Standardized response format with proper citations
3. **Reliability**: Backed by OpenAI's infrastructure
4. **Integration**: Works seamlessly with your existing OpenAI API setup
5. **Up-to-date Information**: Access to recent information beyond the model's training cutoff

## Considerations and Best Practices

1. **API Limits**: Be aware of rate limits and costs associated with the OpenAI API
2. **Error Handling**: Implement robust error handling for API failures
3. **Query Formulation**: Consider reformulating vague queries for better search results
4. **Citation Display**: Always display citations for information from web searches
5. **Caching**: Consider caching frequent searches to reduce API calls and costs
6. **User Experience**: Format search results in a readable way and highlight citations

## Alternative Approaches

If you prefer not to use OpenAI's built-in web search or need more control over the search process, you can integrate with third-party search APIs:

1. **Google Custom Search API**: Requires API key and search engine ID
2. **Bing Web Search API**: Microsoft's search API with rich results
3. **Serper Dev API**: A Google Search API wrapper with simplified access
4. **SerpAPI**: Multi-search engine results with structured data

These alternatives require more setup but offer more customization options for specific use cases.
