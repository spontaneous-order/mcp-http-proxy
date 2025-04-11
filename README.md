# mcp-http-proxy
An HTTP/SSE proxy server for Model Context Protocol (MCP) applications using stdio. Supports raw JSON-RPC commands via HTTP and implements direct stdio communication without an MCP SDK.

# MCP RPC Proxy Worker

## Overview

This Node.js script (`rpc-proxy-worker.js`) acts as an intermediary proxy server between HTTP clients and an underlying MCP (Model Context Protocol) server process. It simplifies interaction with an MCP server by:

1.  **Managing the MCP Process:** It spawns and manages the lifecycle of a configured MCP server running as a child process.
2.  **Providing HTTP Endpoints:** It exposes HTTP endpoints that allow clients to interact with the MCP server's tools and resources without needing to handle the MCP protocol's stdio communication directly.
3.  **Offering Real-time Events:** It provides a Server-Sent Events (SSE) endpoint for clients to receive asynchronous notifications and responses from the MCP server.
4.  **Providing a Web Interface:** It includes a basic web dashboard for viewing available tools and simple interfaces for debugging and sending commands.

## How it Works

The system operates with two main processes:

1.  **The Proxy Worker (This Script):**
    *   Runs as a Node.js process.
    *   Starts an HTTP server (using Express) to listen for client requests.
    *   Spawns the *actual* MCP server as a child process based on the configuration (`MCP_CONFIG`).
    *   Communicates with the child MCP process using its standard input (`stdin`), standard output (`stdout`), and standard error (`stderr`).
    *   Handles translating some HTTP requests (like URL parameters) into JSON-RPC commands for the MCP process.
    *   Forwards raw JSON-RPC commands received via specific HTTP endpoints to the MCP process.
    *   Reads responses and events from the MCP process's `stdout`/`stderr`.
    *   Sends responses back to HTTP clients.
    *   Pushes events and asynchronous responses to connected SSE clients.

2.  **The MCP Server (Child Process):**
    *   Launched by the proxy worker.
    *   Listens for JSON-RPC commands on its `stdin`.
    *   Executes MCP operations (like listing/calling tools).
    *   Writes JSON-RPC responses and protocol messages to its `stdout`.
    *   Writes logs or errors to its `stderr`.

## Launching the Server

1.  **Prerequisites:** Ensure you have Node.js installed.
2.  **Navigate:** Open your terminal and change the directory to where `rpc-proxy-worker.js` is located.
3.  **Run:** Execute the command:
    ```bash
    node rpc-proxy-worker.js
    ```
4.  **Output:** You should see output indicating the server is running, typically including:
    ```
    Web interface running on http://localhost:3005
    SSE endpoint available at http://localhost:3005/sse
    ```
    The server listens on port 3005 by default.

## Communication Mechanisms

### 1. Stdio (Proxy <-> MCP Process) - *Internal*

This is the **internal communication channel** between the proxy worker script and the child MCP server process it manages.

*   **Proxy -> MCP:** The proxy sends validated JSON-RPC command strings to the MCP process's `stdin`.
*   **MCP -> Proxy:**
    *   The MCP process sends JSON-RPC response strings (results, errors, protocol messages) to its `stdout`.
    *   The MCP process sends log messages or fatal errors to its `stderr`.
*   **User Interaction:** Users **do not** directly interact with the *proxy's* stdio to send commands. All interaction happens via the HTTP endpoints.

### 2. Server-Sent Events (SSE) (Proxy -> Clients) - *External*

This is an **external, primarily unidirectional communication channel** allowing the proxy to push events to connected HTTP clients in real-time.

*   **Connection:** Clients establish an SSE connection by making an HTTP `GET` request to the `/sse` endpoint. The proxy keeps this connection open.
*   **Pushing Events:** When the proxy receives data from the MCP process's `stdout` (responses) or `stderr` (logs/errors), it formats this data according to the SSE protocol (`data: <json_string>\n\n`) and pushes it down the open connection to *all* currently connected SSE clients.
*   **Sending Commands:** Clients **cannot** send commands *back* to the proxy over the same SSE connection. To execute a command, an SSE client must make a separate standard HTTP request (e.g., `POST` to `/rpc/raw/command` or `GET` to `/tool/:toolName`). The result of that command will then typically be pushed back to the client via the `/sse` stream it's listening on.

## Interacting with the Proxy (HTTP Endpoints)

The proxy exposes several HTTP endpoints:

*   **`GET /`**
    *   Displays the main HTML dashboard, listing available MCP tools discovered from the child process. Provides forms to execute tools via the `/tool/:toolName` endpoint.

*   **`GET /tools`**
    *   Returns a JSON array listing all available tools provided by the MCP server, including their names, descriptions, and input schemas.

*   **`GET /tool/:toolName`**
    *   Executes a specific tool.
    *   Parameters for the tool are provided as **URL query parameters** (e.g., `/tool/my_tool?param1=valueA&param2=123`).
    *   The proxy **translates** these URL parameters into a valid JSON-RPC `tools/call` request object, validating them against the tool's schema.
    *   Sends the command to the MCP process via stdio.
    *   Returns an **HTML page** displaying the RPC command sent and the response received from the MCP process.

*   **`POST /rpc/raw/command`**
    *   Executes a raw JSON-RPC command.
    *   Expects the **full JSON-RPC 2.0 request object** in the `POST` request body with `Content-Type: application/json`.
    *   The proxy **validates** the structure of the incoming request body against the basic RPC schema.
    *   It **forwards** the validated, raw command directly to the MCP process's `stdin`.
    *   Returns the **raw JSON-RPC response** received from the MCP process's `stdout` directly in the HTTP response body as JSON. *This is the primary endpoint for programmatic interaction where you construct the full RPC call yourself.*

*   **`GET /sse`**
    *   Establishes a Server-Sent Events stream. Clients connect here to receive real-time events (MCP responses, logs) pushed from the server.

*   **`GET /sse-client`**
    *   Provides a simple HTML page that connects to the `/sse` endpoint and displays the received events. Useful for debugging the SSE stream.

*   **`GET /help`**
    *   Displays an HTML page providing documentation for the available tools and how to use the `/tool/:toolName` endpoint.

*   **Debugging Endpoints:**
    *   `GET /rpc/command`: Displays an HTML interface for manually constructing and sending RPC commands via a web form (uses `/rpc/raw/command` internally).
    *   `GET /rpc/raw`: Shows a JSON history of raw messages exchanged between the proxy and the MCP process.
    *   `POST /rpc/raw/clear`: Clears the raw message history.
    *   `GET /debug`: Displays a more comprehensive HTML debug interface showing RPC, process, and SSE events.
    *   `GET /debug/logs`: Returns the current debug logs as JSON.
    *   `GET /debug/sse`: SSE stream specifically for debug log events.

*   **(Optional) MCP Management Endpoints:** (`/mcp/*`)
    *   Endpoints like `/mcp/install`, `/mcp/start`, `/mcp/stop`, `/mcp/list` are present for dynamically managing different MCP server installations if the `mcp-manager.js` is used.

## Sending Commands (Examples)

### Using `POST /rpc/raw/command` (Recommended for programmatic use)

Send the complete JSON-RPC request in the body.

**curl (Bash/zsh/WSL):**

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' \
  http://localhost:3005/rpc/raw/command
```

**curl (Windows cmd - careful with quoting):**

```cmd
curl -X POST -H "Content-Type: application/json" -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/list\",\"id\":1}" http://localhost:3005/rpc/raw/command
```

**PowerShell:**

```powershell
$rpcBody = @{
    jsonrpc = "2.0"
    method = "tools/call"
    id = 2
    params = @{
        name = "your_tool_name"
        arguments = @{
            param1 = "value1"
            count = 10
        }
    }
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri http://localhost:3005/rpc/raw/command -Method Post -ContentType 'application/json' -Body $rpcBody
```

### Using `GET /tool/:toolName` (Convenient for simple GET requests)

Provide parameters in the URL query string.

**curl:**

```bash
# Assuming a tool 'worker_get' exists that takes a 'name' parameter
curl "http://localhost:3005/tool/worker_get?name=my-worker"
```

## Error Handling

The proxy attempts to handle errors gracefully:

*   **HTTP Errors:** Returns standard HTTP status codes (e.g., 404 for unknown tools/endpoints, 400 for invalid parameters on `/tool/:toolName`).
*   **Parameter Validation:** The `/tool/:toolName` endpoint validates query parameters against the tool's schema before sending the command. Errors are returned in the HTML response. The `/rpc/raw/command` endpoint performs basic JSON-RPC structure validation.
*   **MCP Errors:** Errors returned by the underlying MCP process (e.g., tool execution failures) are captured from its `stdout` (as JSON-RPC error responses) or `stderr` and are relayed back to the client either in the direct HTTP response (for `/rpc/raw/command`) or within the HTML page (for `/tool/:toolName`), and potentially pushed via SSE.

## Cleanup

The script registers handlers for `exit` and `SIGINT` (Ctrl+C). On exit, it attempts to:

1.  Kill the child MCP process (`MCPWorker.mcpProcess.kill()`).
2.  Close all active SSE client connections.
