 Here is your **comprehensive, single master prompt** designed to instruct an AI agent (or development team) to build the **complete, production-grade Enterprise ReAct Agentic System** with full frontend, backend, unified memory, MCP tooling, Docker orchestration, and governance layers.

This prompt contains **zero code** but provides exhaustive architectural specifications, behavioral constraints, and integration requirements necessary to generate a fully working system.

***

# MASTER BUILD PROMPT: Enterprise ReAct Agentic System

## SYSTEM OBJECTIVE
Construct a complete, autonomous **Enterprise ReAct Agent** that functions as a non-human system actor capable of perceiving environments, decomposing complex goals into hierarchical task plans, executing actions within secure isolated sandboxes, and reflecting on outcomes to improve accuracy. The system must include a **real-time reasoning frontend**, a **stateful LangGraph backend**, a **unified PostgreSQL memory architecture**, **MCP-standardized tool registry**, **Docker-based execution sandboxing**, and **enterprise governance controls**.

---

## 1. THE COGNITIVE ARCHITECTURE: ReAct with Hierarchical Planning

**Core Identity:** You are the **Enterprise Autonomous Reasoner**. You operate in a continuous cognitive cycle of **Observe → Think → Plan → Act → Reflect → Adapt**.

### 1.1 The ReAct Loop Specification
- **Observation Phase:** Ingest user goals, environmental states, and tool outputs as ground truth. Never proceed without validating observations against previous assumptions.
- **Reasoning Phase:** Perform explicit verbal reasoning traces. Decompose ambiguous high-level goals into a **Directed Acyclic Graph (DAG)** of atomic sub-tasks with clear dependencies.
- **Action Phase:** Emit structured JSON tool calls following MCP standards. Stop immediately after action emission to await observation feedback.
- **Reflection Phase:** Critique your own outputs against defined rubrics to catch hallucinations, logical inconsistencies, or missing context before proceeding.
- **Adaptation Phase:** Dynamically replan the DAG when observations contradict expectations or reveal new constraints.
- **Termination Conditions:** Halt only upon definitive goal achievement, unrecoverable system failure, or exhaustion of iteration/token budgets (hard limit: 10 reasoning cycles).

### 1.2 Planning Agent Specifications
- **Topology Selection:** Dynamically choose execution patterns based on task structure:
  - **Sequential:** For dependent tasks requiring previous outputs
  - **Parallel Fan-Out:** For independent sub-tasks executable concurrently
  - **Hierarchical:** For complex multi-stage objectives requiring nested decomposition
- **Dependency Mapping:** Explicitly declare task dependencies in the DAG. Tasks with no dependencies execute in parallel; dependent tasks queue until prerequisites complete.
- **Checkpointing:** Persist plan state after each major phase to enable crash recovery and Human-in-the-Loop interruption.

---

## 2. FRONTEND ARCHITECTURE: The Reasoning Console

**Design Philosophy:** Build a monospace "Command Center" interface that renders the agent's internal cognitive process with complete transparency.

### 2.1 Core UI Components
- **Reasoning Trace Stream:** A real-time, scrollable feed displaying the agent's "Thought" processes, "Action" emissions, and "Observation" receipts. Use distinct visual hierarchies (color-coding, indentation) to differentiate cognitive phases.
- **Execution Graph Visualization:** An interactive, animated node graph showing the current LangGraph state machine. Highlight active nodes, completed paths, and pending transitions.
- **Memory Inspector:** A searchable, filterable panel displaying episodic history, semantic knowledge retrievals, and procedural skill invocations with timestamps and relevance scores.
- **Tool Call Inspector:** Expandable cards showing structured JSON tool requests, expected schemas, and actual responses with latency metrics.

### 2.2 Human-in-the-Loop Interface
- **Approval Gate Modal:** When high-impact actions trigger (file deletion, database writes, external API calls), render a blocking modal displaying:
  - Proposed action with full parameter visibility
  - Agent's reasoning justification
  - Risk assessment (read-only vs. destructive)
  - **Decision Controls:** Approve (proceed), Edit (modify parameters), Reject (abort with explanation), or Escalate (route to senior operator)
- **Intervention Controls:** Allow humans to inject new observations, correct agent assumptions, or force plan replanning mid-execution.

### 2.3 Streaming & Real-Time Requirements
- Implement Server-Sent Events (SSE) or WebSocket connections to stream token-by-token reasoning traces
- Support `stream_mode=['updates', 'messages']` to render both state transitions and LLM token generation
- Maintain connection persistence across browser refreshes using session resumption tokens

---

## 3. BACKEND ARCHITECTURE: LangGraph Orchestration Layer

**Framework:** Build a stateful directed graph using LangGraph with explicit node definitions and conditional edge routing.

### 3.1 StateGraph Node Definitions
- **`reasoner_node`:** The LLM cognitive core. Accepts current state, generates reasoning traces, decides next action or final answer. Implements reflexion self-critique before outputting.
- **`planner_node`:** Specialized node for hierarchical task decomposition. Takes high-level goals, outputs structured DAG with dependency mappings.
- **`executor_node`:** Tool invocation handler. Validates action schemas, routes to appropriate MCP servers, manages parallel execution for fan-out topologies.
- **`critic_node`:** Reflection and validation layer. Scores outputs against rubrics, detects hallucinations, triggers replanning if quality thresholds unmet.
- **`hitl_interrupt_node`:** Human-in-the-loop checkpoint. Persists state, emits approval request to frontend, blocks until human response received.
- **`recovery_node`:** Error handling and retry logic. Implements exponential backoff, circuit breakers, and graceful degradation strategies.
- **`memory_consolidation_node`:** Background process for compressing episodic logs, updating semantic embeddings, and refining procedural skills.

### 3.2 State Persistence & Checkpoints
- Implement **AsyncPostgresSaver** as the primary checkpointer for all graph states
- Enable long-running workflow persistence with pause/resume functionality
- Support state branching for exploring alternative reasoning paths (tree-of-thoughts)
- Implement automatic state cleanup policies for completed or stale sessions

### 3.3 API Layer Specifications
- **Framework:** FastAPI for high-performance async request handling
- **Endpoints:**
  - `POST /sessions` - Initialize new agent sessions with role configuration
  - `POST /sessions/{id}/messages` - Submit user goals, returns streaming response
  - `GET /sessions/{id}/state` - Retrieve current graph state and execution history
  - `POST /sessions/{id}/approve` - Human approval response for HITL gates
  - `GET /sessions/{id}/trace` - Retrieve complete reasoning trajectory for audit
- **Authentication:** JWT-based with role-based access control (RBAC) mapping to agent tool permissions

---

## 4. UNIFIED MEMORY ARCHITECTURE: Tiered PostgreSQL System

**Philosophy:** Consolidate all memory types into a single PostgreSQL instance with specialized extensions to eliminate multi-database latency and consistency issues.

### 4.1 Episodic Memory (Time-Series)
- **Implementation:** PostgreSQL with TimescaleDB extension (hypertables)
- **Schema:** Timestamped records of every conversation turn, tool invocation, observation receipt, and reasoning trace
- **Capabilities:** Fast temporal range queries (e.g., "what actions occurred between 2-3 PM?"), automatic partitioning by time, compression of historical data
- **Retention:** Configurable policies for hot (recent), warm (compressed), and cold (archived) storage tiers

### 4.2 Semantic Memory (Vector Search)
- **Implementation:** PostgreSQL with pgvector extension, utilizing DiskANN indexes for high-dimensional embeddings
- **Schema:** Vector embeddings of enterprise documentation, code repositories, past solutions, and external knowledge sources
- **Capabilities:** Sub-50ms similarity search, hybrid search combining vector similarity with keyword filtering, metadata-rich retrieval for context attribution
- **Ingestion Pipeline:** Automatic chunking, embedding generation, and index updates when new documents added

### 4.3 Procedural Memory (Relational)
- **Implementation:** Standard PostgreSQL relational tables with strict schemas
- **Schema:** Agent role definitions, tool registry configurations, learned behavioral preferences, skill templates, and organizational standard operating procedures
- **Capabilities:** Version-controlled skill updates, A/B testing of behavioral patterns, audit trails for policy changes

### 4.4 Context Engineering System
- **Single-Query Join:** Construct LLM context windows using unified SQL queries that join across all three memory layers
- **Compaction Strategy:** Implement intelligent summarization of older episodic logs when approaching token limits, preserving key decision points while preventing context rot
- **Relevance Scoring:** Rank memory retrievals by recency, semantic similarity, and procedural applicability to optimize attention budget

---

## 5. ACTION LAYER: MCP Tool Registry & Execution

**Standard:** All tools must adhere to **Model Context Protocol (MCP)** specifications for discoverability, schema validation, and secure invocation.

### 5.1 MCP Server Architecture
- **Server Types:** Implement both local (stdio-based) and remote (HTTP-based) MCP servers
- **Discovery:** Dynamic tool manifest exposure allowing runtime discovery without code changes
- **Lifecycle Management:** Health checks, automatic restarts for crashed servers, graceful shutdown handling

### 5.2 Core Tool Categories

**File System Tools (Sandboxed):**
- `create_file(path, content, encoding)` - Generate new files with content validation
- `read_file(path, offset, limit)` - Stream file contents with pagination for large files
- `write_file(path, content, mode)` - Atomic file writes with backup creation
- `modify_file(path, regex_pattern, replacement)` - Targeted regex-based modifications
- `delete_file(path, permanent)` - Safe deletion with recycle bin option
- `list_directory(path, recursive)` - Directory traversal with filtering
- `search_files(path, query, file_pattern)` - Content search across file trees

**Command Execution Tools (Sandboxed):**
- `execute_command(command, working_dir, timeout, env_vars)` - Shell command execution with resource limits
- `read_process_stream(process_id)` - Real-time output streaming for long-running commands
- `terminate_process(process_id)` - Graceful or forced process termination

**Database & API Tools:**
- `sql_query(connection_string, query, read_only)` - Database interaction with write protection modes
- `http_request(method, url, headers, body)` - Generic HTTP client for external APIs
- `mcp_resource_read(uri)` - Access to MCP Resources (readonly data sources)

### 5.3 Tool Schema & Validation
- **Pydantic Models:** Strict input/output schema definitions for all tools
- **Argument Validation:** Pre-execution validation against schemas with detailed error messages
- **Output Transformation:** Standardized response formats including success/failure status, execution time, and result metadata

---

## 6. EXECUTION LAYER: Docker Sandbox Integration

**Security Model:** Complete isolation of untrusted AI-generated code and commands from host systems.

### 6.1 Sandbox Architecture
- **Container Technology:** Docker with optional gVisor or Kata Containers for additional kernel isolation
- **Image Specifications:** Minimal, hardened base images (distroless or alpine) with only necessary dependencies
- **Network Policies:** Restricted egress/ingress, no access to internal networks unless explicitly whitelisted
- **Resource Limits:** CPU, memory, disk I/O, and network bandwidth quotas per session

### 6.2 Workspace Mirroring
- **Volume Mounting:** Read-only mounts of specific workspace directories, write access only to designated scratch directories
- **File System Watching:** Real-time synchronization of file changes between host and container for live collaboration
- **Secret Injection:** Secure handling of API keys and credentials via environment variables or secret mounts, never persisting in container layers

### 6.3 Blast Radius Controls
- **Immutability:** Container filesystems are ephemeral; all changes discarded on session end unless explicitly committed
- **Privilege Dropping:** Containers run as non-root users with minimal Linux capabilities
- **Audit Logging:** All file system and network operations logged for security review

---

## 7. GOVERNANCE, OBSERVABILITY & EVALUATION

### 7.1 Distributed Tracing & Monitoring
- **Tracing:** OpenTelemetry or LangSmith integration to visualize every node transition, tool invocation, and LLM call
- **Metrics:** Token usage per session, latency per reasoning cycle, tool success/failure rates, memory retrieval performance
- **Alerting:** Anomaly detection for unusual token consumption, repeated tool failures, or potential security violations

### 7.2 Evaluation Framework
- **Golden Dataset:** Curated collection of 20-50 production-representative tasks with verified optimal reasoning paths and tool call sequences
- **Evaluation Pillars:**
  - **Task Success Rate:** Percentage of goals achieved without human intervention
  - **Tool Usage Quality:** Correctness of tool selection, parameter accuracy, error recovery effectiveness
  - **Reasoning Coherence:** Logical consistency of thought traces, appropriate task decomposition
  - **Cost-Performance:** Token efficiency relative to task complexity
- **Regression Testing:** Automated evaluation runs against golden dataset before deployment of model or code updates

### 7.3 Constraint Compliance & Safety
- **Hardcoded Never Rules:** System-level prohibitions (e.g., "Never access /etc/passwd", "Never execute rm -rf /") enforced at tool registry and sandbox levels
- **Automated Judge Models:** Secondary LLM instances reviewing primary agent outputs for policy violations before HITL gates
- **Audit Trails:** Immutable logs of all decisions, tool calls, and human interventions for compliance reporting

---

## 8. DEPLOYMENT ARCHITECTURE: Docker Compose Ecosystem

**Requirement:** Provide complete container orchestration enabling single-command deployment of the entire system.

### 8.1 Service Definitions
- **`frontend`:** React/Vue-based reasoning console, served via Nginx, connects to backend WebSocket/SSE
- **`backend`:** FastAPI application with LangGraph orchestrator, horizontal scaling support
- **`postgres`:** PostgreSQL 15+ with TimescaleDB and pgvector extensions pre-configured
- **`redis`:** Session caching and message queue for asynchronous task processing
- **`mcp-servers`:** Collection of MCP tool servers (file system, command execution, database connectors)
- **`sandbox`:** Docker-in-Docker (DinD) or sibling container setup for executing untrusted code
- **`nginx`:** Reverse proxy, SSL termination, load balancing across backend instances

### 8.2 Networking & Security
- **Internal Networks:** Isolated Docker networks for service-to-service communication
- **Secrets Management:** Docker Secrets or environment file injection for credentials
- **Health Checks:** Comprehensive health check endpoints for all services with automatic restart policies

### 8.3 Persistence & Volumes
- **Named Volumes:** PostgreSQL data, Redis persistence, uploaded files, session logs
- **Backup Strategy:** Automated volume snapshots and point-in-time recovery capabilities

### 8.4 Initialization & Seeding
- **Database Migration:** Automated schema creation and extension enabling on first startup
- **Seed Data:** Default agent roles, example procedural memories, and demo golden dataset entries
- **MCP Registration:** Automatic discovery and registration of available MCP servers on system startup

---

## 9. MASTER TECHNOLOGY STACK

**Backend & Orchestration:**
- `langgraph` - Stateful graph workflows and checkpointing
- `langchain-core` - LLM abstraction and message handling
- `fastapi` - High-performance async API framework
- `uvicorn` - ASGI server with WebSocket support
- `pydantic>=2.0` - Strict schema validation and serialization

**Memory & Data:**
- `psycopg3` - Async PostgreSQL driver
- `pgvector` - Vector similarity search extension
- `timescaledb` - Time-series data optimization
- `redis` - Caching and message brokering

**Tooling & Integration:**
- `mcp` - Model Context Protocol SDK
- `docker` - Container management and sandboxing

**Frontend:**
- `react` or `vue` - Component-based UI framework
- `typescript` - Type-safe frontend development
- `tailwindcss` - Utility-first styling
- `d3.js` or `cytoscape` - Graph visualization for execution flows

**Observability:**
- `opentelemetry` - Distributed tracing and metrics
- `langsmith` - LLM-specific observability (optional)
- `prometheus` - Metrics collection
- `grafana` - Visualization dashboards

**Infrastructure:**
- `docker` & `docker-compose` - Container orchestration
- `nginx` - Reverse proxy and static file serving

---

## 10. SUCCESS CRITERIA

The built system must demonstrate:

1. **Autonomous Task Completion:** Successfully decompose and execute multi-step enterprise tasks (e.g., "Analyze Q3 sales data, identify anomalies, generate report, email stakeholders") without human intervention except at defined approval gates.

2. **Transparent Reasoning:** Provide complete, inspectable reasoning traces that non-technical stakeholders can follow and audit.

3. **Memory Continuity:** Maintain context across sessions, recall previous interactions, and apply learned procedures to new similar tasks.

4. **Secure Execution:** Isolate all file and command operations in sandboxed environments with no possibility of host system compromise.

5. **Human Governance:** Pause appropriately for high-stakes decisions, accept human corrections mid-stream, and maintain comprehensive audit trails.

6. **Operational Resilience:** Recover gracefully from crashes, handle tool failures with retry logic, and scale horizontally under load.

7. **Single-Command Deployment:** Launch entire stack with `docker-compose up` with automatic initialization of all dependencies, schemas, and seed data.

***

**Execute this prompt to generate the complete, production-ready Enterprise ReAct Agentic System.**
