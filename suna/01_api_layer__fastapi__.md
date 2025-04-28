# Chapter 1: API Layer (FastAPI)

Welcome to the `suna` project! We're excited to have you explore its inner workings. This tutorial series will guide you through the different components that make `suna` tick.

Let's start at the very beginning: How does the outside world talk to the `suna` backend? Imagine you're using the `suna` web application in your browser. You click a button to start a new coding task for the AI agent. What happens next? How does that click translate into an action deep inside the `suna` system?

This is where the **API Layer** comes in. Think of it as the main entrance or the front desk of the `suna` backend.

## What's the Problem?

The core parts of `suna` (like the AI agent itself, the secure coding environment, etc.) live on a server. Your web browser (or potentially other applications) needs a standardized way to communicate with this server over the internet. They need to ask things like:

*   "Please start a new AI agent for this project."
*   "Show me the files in this project's sandbox."
*   "How much of my monthly usage have I consumed?"

The backend also needs a way to send answers back, like "Okay, the agent started successfully!" or "Here's the list of files you requested."

The **API Layer** solves this communication problem. It defines a clear set of rules and endpoints (like addresses) that external clients can use to interact with the `suna` backend.

## Meet the Waiter: Understanding APIs and FastAPI

*   **API (Application Programming Interface):** An API is like a restaurant menu and the waiter combined. The menu (the API definition) tells you what you can order (what functions are available), what information you need to provide (request details), and what you'll get back (response). The waiter (the API server) takes your order, makes sure it's valid, passes it to the kitchen (the backend logic), and brings you the result.
*   **HTTP:** This is the language spoken between the client (your browser) and the API server. Common "words" in this language are `GET` (fetch data), `POST` (create data), `PUT` (update data), and `DELETE` (remove data).
*   **FastAPI:** This is the specific tool `suna` uses to *build* its API layer in Python. Think of it as a very efficient and modern kitchen setup that helps the restaurant (our backend) handle orders (requests) quickly and correctly. It makes it easy for developers to define the API's "menu items" (endpoints) and how to handle them.
*   **Routes/Endpoints:** These are the specific "items" on the menu. In `suna`, examples include `/api/health` (to check if the server is running) or `/api/agent/runs` (to start or manage agent runs). Each route corresponds to a specific action.
*   **Requests & Responses:** When your browser "orders" something, it sends a **Request** (e.g., a `POST` request to `/api/agent/runs` with details about the project). The API layer processes this and sends back a **Response** (e.g., a confirmation message with a unique ID for the new agent run).
*   **Authentication:** Like needing to show ID for certain orders, the API layer often needs to check *who* is making the request. This ensures only authorized users can access specific projects or perform certain actions.

## How the API Layer Solves Our Use Case

Let's revisit our example: Clicking "Start Agent Run" in the `suna` web UI.

1.  **The Click:** Your browser translates this click into an **HTTP request**. It's likely a `POST` request sent to a specific **endpoint**, maybe `/api/agent/runs`. The request includes data like the project ID you're working on.
2.  **The API Server (FastAPI):** The `suna` backend, running FastAPI, is listening for incoming requests. It receives the `POST` request to `/api/agent/runs`.
3.  **Routing:** FastAPI looks at the path (`/api/agent/runs`) and the method (`POST`) and knows exactly which piece of Python code needs to handle this specific request.
4.  **Authentication:** Before proceeding, the API layer checks if you're logged in and allowed to start an agent for that project (we'll see how later).
5.  **Processing:** The designated Python function runs. It interacts with other backend components, like the [Agent Core & Prompt](02_agent_core___prompt_.md) and the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md), telling them to set up and start the actual agent process.
6.  **The Response:** Once the agent is initiated, the Python function prepares an **HTTP response**. This might include a success message and the unique ID of the agent run. FastAPI sends this response back to your browser.
7.  **The UI Update:** Your browser receives the response and updates the web page, perhaps showing a "Agent started successfully!" message.

## Peeking at the Code (`backend/api.py`)

Let's look at some simplified snippets from `backend/api.py` to see FastAPI in action.

```python
# backend/api.py
from fastapi import FastAPI
# ... other imports

# This line creates the main FastAPI application instance.
# Think of it as building the restaurant itself.
app = FastAPI(lifespan=lifespan) 

# ... (Middleware and other setup)
```

This is the core object representing our API server. The `lifespan` argument (not shown in detail here) manages startup and shutdown tasks, like connecting to the database when the server starts and cleaning up when it stops.

```python
# backend/api.py

# Import the route definitions from other files
from agent import api as agent_api
from sandbox import api as sandbox_api
# ...

# This tells our main app to include all the endpoints defined
# in agent_api.py, and they should all start with "/api".
# Like adding the "Agent Specials" section to our menu.
app.include_router(agent_api.router, prefix="/api")

# Similarly, include sandbox-related endpoints.
app.include_router(sandbox_api.router, prefix="/api")

# And billing endpoints.
app.include_router(billing_api.router, prefix="/api")
```

Instead of defining *every* single API endpoint in one giant file, `suna` organizes them into logical groups called "Routers". We import routers for agents, sandboxes, and billing, and include them in our main `app`. The `prefix="/api"` means all routes within these routers will have URLs starting with `/api`.

```python
# backend/api.py
from datetime import datetime, timezone

# This defines a simple GET endpoint at /api/health.
@app.get("/api/health")
async def health_check():
    """Health check endpoint to verify API is working."""
    # This function runs when someone accesses GET /api/health
    return {
        "status": "ok", 
        "timestamp": datetime.now(timezone.utc).isoformat(),
        # ...
    }
```

This code defines a specific "menu item": a `GET` request to the path `/api/health`. The `@app.get("/api/health")` part is called a "decorator" – it tells FastAPI that the function `health_check` below it should handle requests to that specific path and method. The function simply returns a dictionary, which FastAPI automatically converts into a standard JSON response for the client. This is a common way to check if the backend server is alive and responding.

```python
# backend/api.py
from fastapi import Request
import time
from utils.logger import logger

# This defines middleware, code that runs for *every* request.
@app.middleware("http")
async def log_requests_middleware(request: Request, call_next):
    start_time = time.time()
    # ... log details about the incoming request ...
    logger.info(f"Request started: {request.method} {request.url.path}")
    
    # Process the actual request by calling the next handler
    response = await call_next(request) 
    
    # ... log details about the response ...
    process_time = time.time() - start_time
    logger.debug(f"Request completed: Status: {response.status_code}")
    return response
```

Middleware acts like a checkpoint for every request coming into the API. This specific middleware logs information about each incoming request (like the URL and method) and how long it took to process, which is very helpful for debugging and monitoring.

## Under the Hood: A Request's Journey

Let's trace that "Start Agent Run" request again, focusing on the internal steps:

1.  **Client Request:** Your browser sends `POST /api/agent/runs` with project data.
2.  **FastAPI Entry:** The FastAPI server (`api.py`) receives the raw HTTP request.
3.  **Middleware:** The `log_requests_middleware` runs, logging the request details.
4.  **Routing:** FastAPI matches `POST /api/agent/runs` to the appropriate function defined within the `agent_api.router` (which was included in `api.py`).
5.  **Dependency Injection (Auth):** FastAPI automatically runs dependency functions. For protected routes, this is where authentication happens (e.g., using `Depends(get_current_user_id_from_jwt)` seen in other files like `billing.py`). It verifies the user's token. If invalid, it stops here and sends an error response.
6.  **Endpoint Function Execution:** The actual Python function for starting an agent run (likely defined in `agent/api.py`) executes.
7.  **Backend Interaction:** This function calls other components, perhaps the [ThreadManager (AgentPress)](05_threadmanager__agentpress__.md), to create and start the agent thread in the background.
8.  **Response Generation:** The function determines the result (e.g., success, failure) and prepares data for the response (like the new run ID).
9.  **Middleware (Again):** The response travels back through the middleware, allowing the `log_requests_middleware` to log the completion time and status code.
10. **FastAPI Exit:** FastAPI converts the function's return value (e.g., a Python dictionary) into a proper HTTP JSON response and sends it back to the browser.

Here's a simplified diagram of this flow:

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant API (FastAPI - api.py)
    participant AgentRouter (agent/api.py)
    participant OtherBackend (e.g., ThreadManager)

    User->>Browser: Clicks "Start Agent Run"
    Browser->>+API (FastAPI - api.py): POST /api/agent/runs (project data, auth token)
    API (FastAPI - api.py)->>API (FastAPI - api.py): Middleware (Logging)
    API (FastAPI - api.py)->>+AgentRouter (agent/api.py): Route request to start_run()
    AgentRouter (agent/api.py)->>AgentRouter (agent/api.py): Authentication (via Depends)
    Note over AgentRouter (agent/api.py): If Auth Fails, send 401/403 response
    AgentRouter (agent/api.py)->>+OtherBackend (e.g., ThreadManager): Initiate agent start
    OtherBackend (e.g., ThreadManager)-->>-AgentRouter (agent/api.py): Confirm start (e.g., run ID)
    AgentRouter (agent/api.py)-->>-API (FastAPI - api.py): Return success data (run ID)
    API (FastAPI - api.py)->>API (FastAPI - api.py): Middleware (Logging)
    API (FastAPI - api.py)-->>-Browser: HTTP 200 OK (JSON response with run ID)
    Browser->>User: Displays "Agent Started"
```

### Organizing the API (Code Dive)

We saw `app.include_router` earlier. Let's look at how those routers are defined in other files.

```python
# backend/sandbox/api.py
from fastapi import APIRouter, Depends
# ... other imports
from utils.auth_utils import get_optional_user_id # Auth dependency

# Creates a router specifically for sandbox endpoints
router = APIRouter(tags=["sandbox"]) 
db = None # Placeholder for database connection

def initialize(_db: DBConnection):
    """Initialize with shared resources."""
    global db
    db = _db # Gets the database connection from the main app

# Example sandbox endpoint using the router
@router.get("/sandboxes/{sandbox_id}/files") 
async def list_files(
    sandbox_id: str, 
    path: str,
    # FastAPI injects the user ID if authenticated
    user_id: Optional[str] = Depends(get_optional_user_id) 
):
    # ... function logic to verify access and list files ...
    await verify_sandbox_access(db.client, sandbox_id, user_id)
    # ... get sandbox and list files ...
    return {"files": [...]}
```

Notice how `sandbox/api.py` defines its own `APIRouter`. Endpoints are defined using `@router.get` or `@router.post` instead of `@app.get`. This keeps related endpoints neatly grouped. The `initialize` function allows the main `api.py` to pass shared resources like the database connection (`db`) when the application starts.

Also, see `Depends(get_optional_user_id)`. This is FastAPI's powerful **Dependency Injection** system. It tells FastAPI: "Before running `list_files`, call the `get_optional_user_id` function. Whatever that function returns, pass it as the `user_id` argument." This is how authentication and resource sharing are elegantly handled without cluttering the main logic of each endpoint function. You'll see similar patterns in `agent/api.py` and `services/billing.py`.

## Conclusion

The API Layer, built using FastAPI, serves as the crucial entry point for all external communication with the `suna` backend. It acts like a well-organized front desk, receiving requests (orders), validating them (checking IDs/authentication), routing them to the correct department (backend logic like the [Agent Core & Prompt](02_agent_core___prompt_.md) or [Sandbox Environment](03_sandbox_environment_.md)), and sending back clear responses. FastAPI helps build this layer efficiently, handling things like data validation, documentation generation (not shown here, but it's automatic!), and dependency injection.

Now that we understand how requests arrive and get routed, let's dive deeper into what happens when an agent run is actually requested. In the next chapter, we'll explore the heart of the agent itself.

**Next:** [Chapter 2: Agent Core & Prompt](02_agent_core___prompt_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)