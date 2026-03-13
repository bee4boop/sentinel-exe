🚧 Project Status: Architecture & Design Phase (RFC) > This repository currently serves as a comprehensive technical specification and architectural blueprint. Implementation of the Core and SDK is pending.
# Sentinel.EXE
**Sentinel.EXE** is a high-performance, self-hosted infrastructure monitoring and automation platform. Unlike traditional uptime services, Sentinel is designed as a **modular orchestrator**, where the system core is completely decoupled from the execution logic.
## The Core Idea

The project is built on the **"Puzzle Architecture"** concept. You are not limited to standard availability checks. Powered by the **gRPC protocol**, Sentinel allows you to connect any microservices (**Addons**) written in any programming language and orchestrate their **Operations** into complex, multi-stage validation scenarios (**Pipelines**).
## Why Sentinel?

- **gRPC Core:** The binary protocol ensures minimal latency and provides strongly typed contracts between the core and executors.
    
- **Hybrid Runtime:** Use the lightweight "Standard Bundle" for basic tasks or deploy heavy, isolated workers (e.g., using Playwright or Python) for specific high-performance needs.
    
- **Contextual Intelligence:** Every validation step enriches a shared JSON context. This enables smart dependencies (e.g., "take a screenshot only if the HEAD request returns a 503 error").
    
- **Distributed Responsibility:** The core is a pure orchestrator. it doesn't store operation logs or check history. The entire system "memory" is distributed among addons. This allows storage to scale according to specific needs (e.g., storing terabytes of screenshots in a dedicated addon without bloating the core database).
    
- **Self-Hosted & Private:** Complete control over your data and monitoring infrastructure.
    
- **Role-Based Security:** Built-in authentication and authorization system. You define who has access to dashboards and pipeline configurations. All communications with addons can be secured via internal networks or mTLS.
    

---
## Core Concepts

- **Monitor** — The central object of observation (e.g., a specific domain or API endpoint). It acts as a container for validation strategies. A single Monitor can hold multiple Pipelines running in parallel at different intervals (e.g., a quick ping every 30s and a heavy screenshot check once per hour).
    
- **Pipeline** — A specific validation scenario tied to a Monitor. It has its own execution interval and defines the step-by-step logic. A Pipeline represents a complete strategy (e.g., "SSL Certificate Check" or "Auth Smoke Test").
    
- **Flow** — The internal structure of a Pipeline. It is a sequence (or graph) of operations that determines the execution order, transition logic (`on_success` / `on_failure`), and data passing.
    
- **Addon** — A physical gRPC microservice (container) acting as a functionality provider. A single addon can provide multiple distinct operations.
    
- **Operation** — An atomic functional unit within an addon (e.g., `http_head`, `screenshot`). This is the "executor" for a specific step in the Flow.
    
- **Context** — An isolated memory space (JSON) that exists during a single Pipeline run. It allows operations within a Flow to exchange data.
    
- **Params** — A set of configuration data. These can be defined at the Pipeline level (global settings) or individually for each Operation within the Flow (e.g., a specific timeout or target text on a page).

---
## Architecture: The Grid

The architecture of **Sentinel.EXE** is built on a strict **Separation of Concerns** between the **Control Plane** (Core) and the **Data Plane** (Addons). This design allows the core to remain ultra-fast and "agnostic" regarding the specifics of the data being processed.
### 1. Control Plane (The Brain & Clock)

Developed in **Go**, this is the only component aware of user identities and global configurations.

- **Orchestration:** The Core stores configurations for Monitors and Pipelines. It acts as the sole initiator of any execution.
    
- **Identity & Access (IAM):** Manages the user database and Dashboard access control. Every system interaction begins with a permission check.
    
- **API Gateway:** The Core proxies requests to Addons. The Frontend never communicates with Addons directly—only through the Core's secure abstraction layer.
    
- **Event Bus:** Serves as a central relay for events traveling between Addons and the Frontend.
### 2. Data Plane (The Execution & Memory)

A network of independent, decentralized microservices (Addons).

- **Persistence (Distributed Logs):** Unlike legacy systems, execution history is stored within the Addons themselves (e.g., in the `base-addon`). The Core does not track the granular details of an `http_head` operation; it only receives the final execution signal.
    
- **Isolation:** Each Addon is a "black box." If an operation requires long-term storage (e.g., price history or a year's worth of screenshots), the Addon manages its own database (PostgreSQL/ClickHouse/S3) independently.
    
- **Operation Runtime:** Operations within Addons perform atomic tasks without needing knowledge of the overall Pipeline structure.
### 3. The Real-time Pulse (WebSockets & Events)

To ensure high UI reactivity, Sentinel uses a cascading notification system:

1. **Operation Signal:** Upon completing a task, the Addon returns the result to the Core.
    
2. **Core Broadcast:** The Core updates the monitor status in its local database and immediately pushes an event via **WebSocket**.
    
3. **UI Refresh:** The Frontend (**Angular**), receiving the socket signal, identifies which specific widgets require updating and fetches the fresh data directly from the relevant Addon through the Core proxy.

---
## Entity Examples & Contracts

#### 1. The gRPC Contract (`sentinel.proto`)

The fundamental "law" of the system. Every addon must implement this interface to enable seamless integration with the Core.

```protobuf
syntax = "proto3";

package sentinel;

service SentinelAddon {
  // Discovery: The Core queries the addon's capabilities and manifests
  rpc GetManifest (Empty) returns (Manifest);

  // Execution: The Core triggers a specific functional operation
  rpc ExecuteOperation (ExecuteRequest) returns (ExecuteResponse);

  // Observability: The Core fetches data from the addon to render UI widgets
  rpc GetWidgetData (WidgetDataRequest) returns (WidgetDataResponse);
}

message Manifest {
  string addon_id = 1;
  string name = 2;
  repeated OperationInfo operations = 3;
}

message OperationInfo {
  string id = 1;            // Example: "http_head"
  string description = 2;
  string config_schema = 3; // UI-Schema or JSON Schema for the config form
}

message ExecuteRequest {
  string operation_id = 1;
  string context_json = 2;  // Current Pipeline Context (JSON)
  string params_json = 3;   // Specific Operation configuration
}

message ExecuteResponse {
  string update_json = 1;  // Data to be merged/patched into the Context
  string signal = 2;       // Execution signals: CONTINUE, STOP, RETRY
  string error = 3;        // Error message (if any)
}
```

#### 2. Addon Manifest (Example of `GetManifest` response)

The Manifest allows the Core to discover an addon's capabilities and determine which configuration settings to render in the UI for the user.

```json
{
  "addon_id": "network_bundle",
  "name": "Standard Network Pack",
  "operations": [
    {
      "id": "http_check",
      "description": "HTTP availability and status check",
      "config_schema": {
        "url": { "type": "string", "required": true },
        "method": { "type": "string", "enum": ["GET", "HEAD", "POST"], "default": "HEAD" },
        "timeout": { "type": "integer", "default": 5 }
      }
    }
  ]
}
```
#### 3. Pipeline Configuration (Flow Storage)

An example of how a specific Pipeline and its internal Flow are defined within the Core's database for a particular Monitor.

```json
{
  "pipeline_id": "main_check",
  "interval": "60s",
  "flow": [
    {
      "step_id": "ping_step",
      "operation_ref": "network_bundle.http_check",
      "params": { "method": "HEAD", "timeout": 10 },
      "on_failure": "notify_error"
    },
    {
      "step_id": "notify_error",
      "operation_ref": "notifiers.telegram",
      "params": { "chat_id": "12345", "template": "Site down!" }
    }
  ]
}
```

#### 4. Shared Context (Data in Motion)

An example of the object that "grows" as it passes through the Flow, carrying results from previous steps to the next ones.

```json
{
  "execution_id": "exec_88291",
  "target_url": "https://ktod.ru",
  "steps": {
    "ping_step": {
      "status_code": 200,
      "latency_ms": 142,
      "server": "nginx"
    },
    "ssl_check": {
      "is_valid": true,
      "expiry_date": "2026-12-01T00:00:00Z",
      "days_left": 263
    },
    "seo_analyzer": {
      "title": "KtoD - IT Community",
      "has_h1": true,
      "links_found": 42
    }
  }
}
```

---
## SDK Examples (Go)

Sentinel provides a lightweight SDK that abstracts away the gRPC boilerplate, allowing the developer to focus purely on business logic.
### 1. Registering a New Addon

The developer simply defines the service structure and registers the operations.

``` go
package main

import (
    "github.com/sentinel-exe/sdk-go"
    "context"
)

func main() {
    // Initialize a new addon
    addon := sdk.NewAddon("network_bundle", "Standard Network Pack")

    // Register an HTTP HEAD operation
    addon.RegisterOperation(sdk.Operation{
        ID:          "http_head",
        Description: "Fast status-code check via HEAD request",
        ConfigSchema: sdk.NewSchema().
            Number("timeout", 5, "Request timeout in seconds").
            Bool("follow_redirects", true, "Whether to follow redirects"),
        
        // The execution logic
        Handler: HandleHttpHead,
    })

    // Start the gRPC server on port 50051
    addon.Run(":50051")
}
```

### 2. Handler Implementation

A handler is a pure function that processes the incoming context and parameters.

```go
func HandleHttpHead(ctx context.Context, req sdk.ExecuteRequest) (sdk.ExecuteResponse, error) {
    // 1. Extract parameters defined by the user in the UI
    timeout := req.Params.GetInt("timeout")
    url := req.Context.GetString("target_url")

    // 2. Execute logic (e.g., performing the HTTP request)
    resp, err := myHttpClient.Head(url, timeout)
    
    if err != nil {
        // Return an error and a STOP signal to interrupt the pipeline
        return sdk.StopResponse(err.Error()), nil
    }

    // 3. Return data to update the context and a CONTINUE signal
    return sdk.ContinueResponse(map[string]interface{}{
        "status_code": resp.StatusCode,
        "latency_ms":  resp.Duration.Milliseconds(),
    }), nil
}
```

#### 3. Python Example (Cross-Language Support)

Thanks to gRPC, writing a Python addon is just as straightforward:

```python
from sentinel_sdk import Addon, ContinueResponse

addon = Addon("ml_checker", "AI Analysis Pack")

@addon.operation(id="analyze_sentiment")
def analyze(context, params):
    text = context.get("page_text")
    result = ai_model.predict(text)
    return ContinueResponse({"sentiment_score": result})

addon.run(port=50051)
```
### Why is this beneficial?

- **Strong Typing:** The SDK provides convenient methods for extracting data from JSON (`req.Params.GetInt`, `req.Context.GetString`).
    
- **High Abstraction:** Developers don't deal with gRPC boilerplate, Protobufs, or raw network sockets.
    
- **Scalability:** You can pack dozens of operations into a single `main.go` and they will all be accessible to the Core through a single port.

---
## The Handshake & Dynamic UI

**Sentinel.EXE** utilizes a **Dynamic Discovery** concept. The Core doesn't need prior knowledge of an addon's functionality—it retrieves all specifications and visualization rules in real-time via gRPC.
### 1. The Handshake Process

- **Connection:** An administrator adds the new addon's address (e.g., `grpc://network-bundle:50051`) via the Dashboard.
    
- **Handshake:** The Core invokes the `GetManifest` method.
    
- **Registration:** The Addon returns a full "passport" of its capabilities: a list of operations, their schemas, and dashboard descriptions.
    
- **On-Demand UI:** The Sentinel Frontend (**Angular**) parses these schemas to dynamically build configuration forms for pipeline steps or widgets for data visualization.
### 2. Discovery Specifications

- **A. Operation Settings (`config_schema`):** Defines the structure and appearance of the configuration form for a specific step in the pipeline.
    
- **B. Data Contract (`provides`):** A declaration of keys that the operation guarantees to write into the **Context**. This allows the Core to highlight available variables for subsequent steps in the **Flow**.
### 3. Sentinel.EXE UI: Interactive Dashboards

If an addon stores its own statistics or execution results (e.g., price history), it can define how that data should be visualized. Instead of rendering raw pixels, the addon utilizes the Core's system **UI-Kit**.

```json
{
  "dashboard_config": {
    "title": "Shop Analytics",
    "widgets": [
      {
        "id": "price_history_table",
        "type": "table", // Core system component (e.g., Ag-Grid)
        "data_func": "GetPriceHistory", // gRPC method for fetching data
        "params": {
          "columns": [
            { "header": "Product", "field": "name" },
            { "header": "Price", "field": "price", "format": "currency" }
          ]
        }
      }
    ]
  }
}
```
### 4. Dynamic Updates

- **Hot Reload:** When the addon code changes (new fields or widgets added), a simple "Refresh Manifest" click in the UI is enough.
    
- **Zero Downtime:** New capabilities appear in the interface instantly. The Core requires no rebuild or restart to support new check types or data displays.
### 5. Event Subscription

In the widget manifest, an addon can specify a list of events it subscribes to (e.g., `on_pipeline_finish`). When such an event occurs, the Core "pings" the frontend via **WebSockets**, triggering an automatic data refresh for that specific widget.

### Why This Matters

This transforms **Sentinel.EXE** from a collection of scripts into a full-fledged **ecosystem**:

1. **Full Control:** Addon developers manage how users interact with their logic using powerful, pre-built Core components.
    
2. **Stable Foundation:** The Core remains a robust foundation, providing a unified UX, authorization, and orchestration without needing to understand the specifics of business tasks (whether it's uptime monitoring or price scraping).

---
### Roadmap: The Sentinel Evolution

#### Milestone 1: The gRPC Backbone (Protocol & Skeleton)

- [ ] **Contract:** Finalization of the `.proto` interface (methods: `Execute`, `GetManifest`, `GetWidgetData`).
    
- [ ] **Core Engine (Go):** A basic orchestrator capable of invoking addon operations on a scheduled basis.
    
- [ ] **Standard Bundle:** Implementation of the first addon with atomic operations (`http_head`, `tcp_ping`) to test end-to-end connectivity.
    
- [ ] **CLI Manager:** A command-line utility for registering addons and manually triggering pipelines.
    

#### Milestone 2: Identity & Security (The Perimeter)

- [ ] **Auth System:** Implementation of user management, JWT authorization, and API protection within the Core.
    
- [ ] **Gateway Logic:** Proxy mechanism for routing requests from the future UI to addons through the Core’s authorization layer.
    
- [ ] **Persistence:** Core database for storing monitor configurations, pipelines, and user accounts.
    
- [ ] **Base Addon (Stateful):** Creation of a standard log storage service to act as the "memory" for all basic checks.
    

#### Milestone 3: Discovery & Dynamic UI (The Interface)

- [ ] **Sentinel Dashboard (Angular):** A base interface displaying the monitor list and their real-time statuses.
    
- [ ] **Handshake Engine:** Logic for automated manifest importing and operation registration.
    
- [ ] **Dynamic Form Builder:** A UI engine that generates configuration forms based on the `config_schema` provided by addons.
    
- [ ] **Discovery UI:** An addon management page (connection, manifest updates, connectivity status).
    

#### Milestone 4: Smart Flow & Context (Intelligence)

-  [ ] **Context Manager:** Implementation of JSON data passing and merging between pipeline steps.
    
-  [ ] **Logic Engine:** Support for transition conditions (`on_success` / `on_failure`) and control signals (`STOP`, `RETRY`).
    
-  [ ] **Variable Injection:** Support for using context data within operation parameters (e.g., `url: {{context.base_url}}`).
    

#### Milestone 5: Reactive Ecosystem (Visualization & Real-time)

- [ ] **WebSocket Bus:** A push-notification system from the Core to the Frontend to signal pipeline completion.
    
- [ ] **Widget Engine:** Implementation of system UI components (Table, Chart, StatCard) in Angular.
    
- [ ] **Data Stream:** Full support for `GetWidgetData` to display analytics accumulated within addons.
    
- [ ] **SDK Release:** Publication of libraries (Go/Python) to enable rapid community development of custom addons.
