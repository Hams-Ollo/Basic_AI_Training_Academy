# 🚀 Developer Technical Implementation Guide: MCP Server Integration in EVA

## 1. Introduction 🌟

### 1.1. Purpose 🎯

This guide provides developers with practical, step-by-step instructions for implementing Model Context Protocol (MCP) servers and integrating them as tools for LangGraph agents within the EVA multi-agent AI framework. It focuses on applying these concepts to the existing `eva_2_test/multi_eva_example.py` project.

### 1.2. Prerequisites ✅

Before you begin, ensure you have the following:

- Python 3.8+ installed 🐍
- Familiarity with FastAPI and LangGraph 🧠
- Your EVA project environment set up, particularly `d:\my_projects\EVA-Multi-Agent-Framework\eva_2_test\multi_eva_example.py` 📁
- Required packages: `fastapi`, `uvicorn`, `gunicorn`, `fastapi-mcp`, `httpx` (for client-side calls) 📦

  ```bash
  pip install fastapi uvicorn gunicorn fastapi-mcp httpx
  ```

## 2. Implementing a Basic FastAPI MCP Server 🛠️

This section demonstrates creating a simple FastAPI application that exposes a tool via MCP.

### 2.1. Create the MCP Server File 📝

Create a new Python file in your project, for example, `mcp_servers/simple_tool_server.py`.

```python
# mcp_servers/simple_tool_server.py
from fastapi import FastAPI
from fastapi_mcp import FastApiMCP
from pydantic import BaseModel, Field

# --- 1. Define your FastAPI application ---
app = FastAPI(
    title="Simple Tools API",
    description="A FastAPI server exposing simple tools via MCP.",
    version="1.0.0"
)

# --- 2. Define input/output models for your tool (optional but good practice) ---
class EchoInput(BaseModel):
    message: str = Field(..., description="The message to echo back.")

class EchoOutput(BaseModel):
    response: str = Field(..., description="The echoed message.")

# --- 3. Create your tool endpoint ---
@app.post("/tools/echo", operation_id="echo_message", response_model=EchoOutput)
async def echo_tool_endpoint(payload: EchoInput):
    """
    Echoes back the provided message.
    """
    return EchoOutput(response=f"Echo: {payload.message}")

# --- 4. Wrap the FastAPI app with FastApiMCP ---
# Ensure the base_url correctly points to where this server will be accessible.
# If running locally on port 8001, it would be "http://localhost:8001"
MCP_SERVER_BASE_URL = "http://localhost:8001" # Make this configurable via .env in a real app

mcp_instance = FastApiMCP(
    app,
    name="SimpleToolServerMCP",
    description="Exposes a simple echo tool.",
    base_url=MCP_SERVER_BASE_URL, # This is the URL of the MCP server itself
    # mcp_api_url=f"{MCP_SERVER_BASE_URL}/mcp" # Explicitly set if needed, default is /mcp
)

# --- 5. Mount the MCP endpoints ---
# This makes the MCP discovery endpoint available (e.g., http://localhost:8001/mcp)
mcp_instance.mount()

# --- 6. (Optional) Add a root endpoint for basic server check ---
@app.get("/")
async def root():
    return {"message": "Simple Tool Server is running. MCP available at /mcp"}

# --- Main block to run with Uvicorn for development ---
if __name__ == "__main__":
    import uvicorn
    # Ensure this port matches the MCP_SERVER_BASE_URL
    uvicorn.run(app, host="0.0.0.0", port=8001, log_level="info")
```

### 2.2. Running the MCP Server 🏃‍♂️

- **For Development (Uvicorn)** 🔧:
    Navigate to the directory containing `simple_tool_server.py` (or adjust path) and run:

    ```bash
    python mcp_servers/simple_tool_server.py
    ```

    Or directly using Uvicorn:

    ```bash
    uvicorn mcp_servers.simple_tool_server:app --host 0.0.0.0 --port 8001 --reload
    ```

    You should be able to access:
  - Server root: `http://localhost:8001/` 🏠
  - OpenAPI docs: `http://localhost:8001/docs` 📚
  - MCP discovery endpoint: `http://localhost:8001/mcp` 🔍

- **For Production-like Setup (Gunicorn)** 🚀:

    ```bash
    gunicorn mcp_servers.simple_tool_server:app -w 2 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8001
    ```

    (Adjust `-w` (workers) as needed, typically `(2 * CPU_CORES) + 1`).

## 3. Integrating MCP Tools into `multi_eva_example.py` 🔄

This section outlines how to modify `eva_2_test/multi_eva_example.py` to use the `echo_message` tool from the MCP server created above.

### 3.1. MCP Client Utility 🧰

It's helpful to have a utility class or functions to interact with MCP servers. Create a new file, e.g., `eva_2_test/mcp_client_utils.py`:

```python
# eva_2_test/mcp_client_utils.py
import httpx
from typing import List, Dict, Any, Optional

class MCPClient:
    def __init__(self, server_base_url: str):
        # Ensure server_base_url does NOT end with /mcp
        if server_base_url.endswith("/mcp"):
            self.server_base_url = server_base_url[:-4]
        elif server_base_url.endswith("/"):
            self.server_base_url = server_base_url[:-1]
        else:
            self.server_base_url = server_base_url
        
        self.mcp_discovery_url = f"{self.server_base_url}/mcp"

    async def list_tools(self) -> Optional[List[Dict[str, Any]]]:
        """Fetches the list of tools from the MCP server."""
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(self.mcp_discovery_url)
                response.raise_for_status() # Raise an exception for HTTP errors
                mcp_response = response.json()
                # The actual tools are usually under a 'tools' key or similar
                # based on fastapi-mcp structure, it's often directly a list or under 'resources'
                # For fastapi-mcp, the response structure is a list of tool definitions.
                # Let's assume it's a list of tools directly for now.
                # Adjust based on actual MCP server response structure.
                # fastapi-mcp returns a list of "MCPTool" like objects.
                # Example structure from fastapi-mcp:
                # [ { "name": "tool_name", "description": "...", "operation_id": "op_id", 
                #     "path": "/path/to/tool", "method": "POST", "parameters": [...], ...} ]
                return mcp_response # Or mcp_response.get("tools") or mcp_response.get("resources")
        except httpx.RequestError as e:
            print(f"Error connecting to MCP server at {self.mcp_discovery_url}: {e}")
            return None
        except httpx.HTTPStatusError as e:
            print(f"HTTP error fetching tools: {e.response.status_code} - {e.response.text}")
            return None
        except Exception as e:
            print(f"An unexpected error occurred while fetching tools: {e}")
            return None

    async def run_tool(self, operation_id: str, payload: Dict[str, Any], tool_path: Optional[str] = None, method: str = "POST") -> Optional[Dict[str, Any]]:
        """
        Runs a specific tool on the MCP server.
        Requires operation_id to identify the tool.
        If tool_path is not provided, it will try to discover it.
        """
        if not tool_path:
            tools = await self.list_tools()
            if not tools:
                print(f"Could not discover tools to find path for operation_id: {operation_id}")
                return None
            
            found_tool = next((t for t in tools if t.get("operation_id") == operation_id), None)
            if not found_tool:
                print(f"Tool with operation_id '{operation_id}' not found on server {self.server_base_url}")
                return None
            tool_path = found_tool.get("path")
            method = found_tool.get("method", "POST").upper() # Get method from tool schema
            if not tool_path:
                print(f"Tool path not found for operation_id '{operation_id}' in discovered tools.")
                return None

        tool_url = f"{self.server_base_url}{tool_path}"
        
        try:
            async with httpx.AsyncClient() as client:
                if method == "POST":
                    response = await client.post(tool_url, json=payload)
                elif method == "GET":
                    # For GET, parameters are usually query params.
                    # httpx handles this if 'params=payload' is used.
                    response = await client.get(tool_url, params=payload) 
                else:
                    print(f"Unsupported HTTP method: {method}")
                    return None
                
                response.raise_for_status()
                return response.json()
        except httpx.RequestError as e:
            print(f"Error connecting to MCP tool at {tool_url}: {e}")
            return None
        except httpx.HTTPStatusError as e:
            print(f"HTTP error running tool {operation_id}: {e.response.status_code} - {e.response.text}")
            return None
        except Exception as e:
            print(f"An unexpected error occurred while running tool {operation_id}: {e}")
            return None

# Example usage (can be run independently for testing)
async def main_test_client():
    # Ensure your simple_tool_server.py is running on http://localhost:8001
    mcp_client = MCPClient(server_base_url="http://localhost:8001")

    print("Listing tools... 🔍")
    tools = await mcp_client.list_tools()
    if tools:
        print("Available tools: 🛠️")
        for tool in tools:
            print(f"- Name: {tool.get('name', 'N/A')}, OpID: {tool.get('operation_id', 'N/A')}, Path: {tool.get('path', 'N/A')}, Method: {tool.get('method', 'N/A')}")
    else:
        print("No tools found or error fetching tools. ❌")
        return

    print("\nRunning 'echo_message' tool... 🔊")
    # The payload structure depends on the tool's input schema (EchoInput model)
    echo_payload = {"message": "Hello from EVA MCP Client!"}
    result = await mcp_client.run_tool(operation_id="echo_message", payload=echo_payload)
    
    if result:
        print("Tool Result: ✅", result)
    else:
        print("Failed to run tool. ❌")

if __name__ == "__main__":
    import asyncio
    asyncio.run(main_test_client())
```

### 3.2. Modifying `multi_eva_example.py` 📝

Let's assume `multi_eva_example.py` has a structure with an orchestrator and various agents. We'll add a new simple agent or modify an existing one to use the MCP tool. For simplicity, we'll outline creating a new node that uses the MCP tool.

**Key changes:**

1. **Import `MCPClient`** 📥
2. **Instantiate `MCPClient`** (ideally, the server URL comes from config/env) 🔌
3. **Create a new agent/node function** that uses the `MCPClient` 🤖
4. **Add this node to your LangGraph graph** 📊
5. **Update routing logic** if necessary to direct tasks to this new node 🔀

**Example Snippets (Illustrative - adapt to your `multi_eva_example.py` structure):**
