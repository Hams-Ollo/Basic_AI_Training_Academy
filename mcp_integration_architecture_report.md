# 🏗️ Engineering and Architecture Report: MCP Server Integration with LangGraph Agents for EVA

## 1. Introduction

### 1.1. Purpose 🎯

This document outlines the engineering and architectural considerations for integrating Model Context Protocol (MCP) servers with LangGraph agents within the EVA (Extensible Versatile Assistant) multi-agent AI framework. The goal is to establish a technically sound, dynamic, modular, and optimized approach for leveraging both official and custom self-hosted MCP servers as tools for EVA agents.

### 1.2. EVA and the Need for MCP Integration 🔄

EVA is a multi-agent AI framework designed for complex task orchestration and intelligent interaction. As EVA evolves, the ability to seamlessly integrate a diverse range of tools and capabilities becomes paramount. MCP provides a standardized mechanism for AI agents to discover and utilize tools exposed by external services, enhancing EVA's extensibility and power. This integration aligns with EVA 2.0's goal of incorporating custom MCP for 3rd party tools.

### 1.3. Model Context Protocol (MCP) 📡

MCP is an open standard that enables AI models and agents to dynamically discover and interact with tools and services. It defines a common interface for tools, making them easily consumable by various AI systems, including LangGraph agents. Key benefits include standardized tool interaction, dynamic tool discovery, and improved modularity in AI applications.

## 2. MCP Fundamentals

### 2.1. What is MCP? 🤔

MCP facilitates communication between AI agents (clients) and tool providers (servers).

- **Key Concepts**:
  - **Tools** 🛠️: Specific functionalities (e.g., web search, database query, code execution) exposed by an MCP server.
  - **Tool Discovery** 🔍: Clients can query servers to get a list of available tools and their schemas.
  - **Tool Invocation** ⚡: Clients can execute tools by sending requests to the server according to the tool's schema.
- **Benefits**:
  - Standardized interface for diverse tools.
  - Reduced boilerplate for tool integration.
  - Enhanced interoperability between AI systems and tool providers.

### 2.2. MCP Server 🖥️

An MCP server hosts one or more tools and exposes them according to the MCP specification.

- **Role**:
  - Advertise available tools and their usage schemas (inputs, outputs).
  - Handle tool execution requests from MCP clients.
  - Return tool execution results.
- **Implementation**: Can be built using various web frameworks, with FastAPI being a popular choice in the Python ecosystem, often using libraries like `fastapi-mcp`.

### 2.3. MCP Client 🤖

An MCP client is an AI agent or system component that consumes tools from an MCP server.

- **Role**:
  - Discover tools available on one or more MCP servers.
  - Format requests to invoke specific tools.
  - Process responses received from the MCP server.
- **Example**: A LangGraph agent configured to use an MCP client to access external functionalities.

### 2.4. Communication 💬

MCP typically uses HTTP-based communication.

- **Server-Sent Events (SSE)** 📲: Often used for streaming updates or asynchronous communication, allowing servers to push data to clients.
- **MCP-Proxy** 🔄: For clients that do not natively support SSE, an MCP-Proxy can be used to bridge the communication, often converting SSE to a standard HTTP long-polling or WebSocket connection.

## 3. Architectural Design for MCP Integration in EVA

### 3.1. Core Principles 🏛️

- **Modularity** 🧩: Design MCP servers to be independent and focused on specific sets of tools (e.g., a server for knowledge base tools, another for communication tools). This promotes maintainability and independent scalability.
- **Scalability** 📈: Utilize production-grade hosting solutions like Gunicorn for FastAPI-based MCP servers to handle concurrent requests efficiently.
- **Discoverability** 🔍: Enable EVA's LangGraph agents to dynamically fetch and understand the capabilities of available MCP tools at runtime.
- **Security** 🔒: Implement robust authentication and authorization mechanisms for accessing MCP tools, and manage sensitive information like API keys securely.

### 3.2. MCP Server Implementation Strategy 🛠️

- **Technology Choice**:
  - **FastAPI** ⚡: A modern, high-performance Python web framework, well-suited for building MCP servers.
  - **`fastapi-mcp` Package** 📦: Simplifies exposing FastAPI endpoints as MCP-compliant tools.
- **Deployment Options**:
    1. **Integrated MCP** 🔄: Add MCP server functionality directly into an existing FastAPI application. Suitable for tightly coupled tools.
    2. **Separate MCP Server(s)** 🧩: Host MCP servers as distinct FastAPI applications. This is the **recommended approach for EVA** as it enhances modularity, allows independent scaling of toolsets, and aligns with microservice architectures. Each server can run on its own port or host.
- **Hosting**:
  - **Gunicorn** 🦄: A mature, production-ready WSGI HTTP server for UNIX-like systems. For FastAPI (an ASGI application), Gunicorn is used with Uvicorn workers.
  - **Uvicorn Workers** 🔄: Gunicorn manages Uvicorn worker processes, which run the FastAPI/ASGI application.
  - **Worker Configuration** ⚙️: A common guideline for the number of Gunicorn workers is `(2 * number_of_cpu_cores) + 1`. This should be tuned based on application I/O characteristics and performance testing.
  - **Startup Scripts** 🚀: Use systemd services, Docker, or other process management tools to manage the Gunicorn server lifecycle (start, stop, restart).

### 3.3. LangGraph Agent Integration Strategy 🤖

- **MCP Client in Agents**:
  - EVA agents will require an MCP client component. This can be achieved by:
    - Using existing Python MCP client libraries.
    - Developing a custom MCP client wrapper tailored to LangGraph's state management and EVA's specific needs. This wrapper would handle tool discovery, invocation, and error management.
- **Tool Discovery** 🔍:
  - Agents, upon initialization or as needed, will query configured MCP server URLs (e.g., `http://localhost:8001/mcp`) to retrieve a list of available tools and their schemas (e.g., using a `GetTools` function).
- **Tool Invocation** ⚡:
  - Once a tool is selected, the agent will use the MCP client to send an execution request to the appropriate MCP server (e.g., using a `RunTool` function), providing necessary input arguments.
- **Multi-Server Support** 🌐:
  - EVA's agent configuration should allow specifying multiple MCP server endpoints.
  - A routing mechanism or a multi-server client within the agent can manage interactions with different servers, potentially based on tool names or categories.
- **Integration Pattern (Example from Research)** 📝:
  - A common pattern involves a `Router` agent that directs tasks, an `Assistant` agent that performs tasks (potentially using tools), and a generic `MCPWrapper` or `MCPSessionFunction` that abstracts the MCP communication (tool fetching and execution).

### 3.4. Data Flow 🔄

1. An EVA LangGraph agent (or its orchestrator) determines the need for an external tool to fulfill a user request or a step in a plan.
2. The agent, via its MCP client component, queries one or more configured MCP servers for available tools. This might involve filtering tools based on name or capability.
3. The agent selects the appropriate tool and prepares the input arguments according to the tool's schema.
4. The MCP client sends a tool invocation request to the specific MCP server hosting the tool.
5. The MCP server receives the request, validates it, and executes the underlying tool logic (e.g., calls a Python function, interacts with a database).
6. The MCP server returns the execution result (or error) to the agent via the MCP client.
7. The agent processes the result and continues its workflow.

### 3.5. Security Considerations 🔒

- **Authentication** 🔑:
  - MCP supports various authentication mechanisms. OAuth2 or token-based authentication (e.g., Bearer tokens) are common.
  - MCP servers should validate credentials before allowing tool discovery or execution.
- **API Key Management** 🗝️:
  - Sensitive information like API keys for underlying services used by tools should be managed securely.
  - Store keys in environment variables (`.env` files for local development, platform-specific secret management for production). MCP servers should load these at startup.
- **Communication Security** 🔐:
  - Always use HTTPS for MCP server endpoints to encrypt data in transit, especially when dealing with sensitive information or operating over public networks.
- **Access Control** 👮:
  - Implement fine-grained access control if different agents or users should have varying levels of access to specific tools or MCP servers.

## 4. Hosting MCP Servers with Gunicorn

### 4.1. Why Gunicorn for FastAPI? 🦄

Gunicorn is a robust and widely-used process manager. When used with Uvicorn workers, it provides:

- **Process Management** 🔄: Supervises multiple worker processes, restarting them if they crash.
- **Scalability** 📈: Allows running multiple worker processes to handle concurrent requests, leveraging multiple CPU cores.
- **Reliability** 💪: Battle-tested in production environments.
- **Configuration** ⚙️: Offers extensive configuration options for tuning performance and behavior.

### 4.2. Setup with Uvicorn Workers 🔄

FastAPI is an ASGI application. Gunicorn, primarily a WSGI server, can manage ASGI applications using Uvicorn workers.

- **Installation** 📦: `pip install gunicorn uvicorn`
- **Worker Class** 👷: Specify `uvicorn.workers.UvicornWorker` as the worker class for Gunicorn.

### 4.3. Example Gunicorn Command 💻

```bash
gunicorn my_mcp_server_app:app -w 4 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8001
```

- `my_mcp_server_app:app`: `my_mcp_server_app.py` is the Python file, and `app` is the FastAPI application instance within that file.
- `-w 4`: Number of worker processes (e.g., 4 workers). Adjust based on `(2 * CPU cores) + 1` and testing.
- `-k uvicorn.workers.UvicornWorker`: Specifies the Uvicorn worker class.
- `-b 0.0.0.0:8001`: Binds the server to port 8001 on all available network interfaces.

### 4.4. Configuration Best Practices ⚙️

- **Worker Class** 👷: Always use `uvicorn.workers.UvicornWorker` for FastAPI.
- **Number of Workers (`-w`)** 🔢: Start with `(2 * CPU_CORES) + 1` and adjust based on load testing.
- **Timeout (`-t`)** ⏱️: Set an appropriate timeout (e.g., `120` seconds) to prevent long-running requests from holding workers indefinitely.
- **Logging** 📝: Configure Gunicorn and FastAPI logging to capture access logs and application errors for monitoring and debugging.
- **Graceful Reloads** 🔄: Use Gunicorn's signals (e.g., `HUP`) for graceful restarts/reloads without dropping active connections, useful for deploying updates.

## 5. Example Scenario: Integrating a Custom Tool via MCP

Let's consider a simple custom tool: `get_current_weather(city: str) -> dict`. 🌦️

1. **Expose via FastAPI MCP Server** 🖥️:

    ```python
    # weather_mcp_server.py
    from fastapi import FastAPI
    from fastapi_mcp import FastApiMCP
    import httpx # Example for an external API call

    app = FastAPI(title="Weather Service API")

    @app.get("/weather/{city}", operation_id="get_current_weather")
    async def get_weather_endpoint(city: str):
        # In a real scenario, call a weather API
        # This is a mock implementation
        if city.lower() == "london":
            return {"city": city, "temperature": "15°C", "condition": "Cloudy"}
        else:
            return {"city": city, "temperature": "N/A", "condition": "Data not available"}

    # Wrap with FastApiMCP
    # Assuming this server runs at http://localhost:8002
    mcp_server = FastApiMCP(
        app,
        name="Weather Tools MCP",
        description="Provides weather information tools.",
        base_url="http://localhost:8002" # The URL where this MCP server itself is accessible
    )
    mcp_server.mount() # Mounts MCP endpoints (e.g., /mcp) on the 'app'

    # To run (e.g., with uvicorn for development):
    # uvicorn weather_mcp_server:app --port 8002 --reload
    # For production, use Gunicorn as described above.
    
