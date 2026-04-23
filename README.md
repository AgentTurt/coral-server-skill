# Coral Server

Coral Server is the open-source runtime for CoralOS — "Kubernetes for AI agents." It provides agent registry, session management, thread-based inter-agent communication, MCP protocol integration, and blockchain payment settlement.

- **Repo:** `https://github.com/Coral-Protocol/coral-server`
- **Docs:** `https://docs.coralos.ai`
- **API spec:** `https://docs.coralos.ai/api_v1.json`
- **Base URL (cloud):** `https://api.coralcloud.ai/`
- **Default local port:** `5555`
- **Stack:** Kotlin, Ktor 3.3.3, MCP Kotlin SDK 0.9.0, Docker Java 3.7.0, Hoplite (TOML config)

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Namespace** | Logical group of sessions. Can auto-delete when the last session exits. |
| **Session** | A single, self-contained context for a multi-agent interaction. Contains threads and agents. |
| **Thread** | A messaging channel within a session. Has a participant list and ordered message history. |
| **Agent** | A runtime process connected to the session via MCP. Defined by `coral-agent.toml`. |
| **Registry** | Catalog of available agents (local files, marketplace, linked servers). |
| **Puppet API** | Server-side control of agents — create threads, send messages, terminate agents, all impersonating a named agent. |

---

## Server Setup

### config.toml

Required file — pass path via `CONFIG_FILE_PATH` env var.

```toml
[auth]
apiKey    = "your-api-key"
secretKey = "your-secret-key"

[network]
host = "0.0.0.0"
port = 5555
# tlsConfig = { ... }  # Optional TLS

[docker]
host       = "unix:///var/run/docker.sock"
socketPath = "/var/run/docker.sock"
imagePrefix = ""

[registry]
paths     = ["./agents"]          # Dirs to scan for coral-agent.toml files
patterns  = ["**/coral-agent.toml"]
hotReload = true

[logging]
level = "INFO"                    # DEBUG | INFO | WARN | ERROR
# webhooks = ["https://..."]      # Optional event notification endpoints

[blockchain]
rpcUrl          = "https://mainnet-rpc-url"
contractAddress = "0x..."
chainId         = 1
```

### Running the server

```bash
# Gradle (dev)
./gradlew run

# Docker (production)
docker run -p 5555:5555 \
  -e CONFIG_FILE_PATH=/config/config.toml \
  -v /path/to/config.toml:/config/config.toml \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/coral-protocol/coral-server

# CLI overrides (dot notation)
./gradlew run --args="--auth.apiKey=key1 --network.port=8080 --registry.hotReload=true"
```

> **Note:** Docker deployment requires mounting the host Docker socket. The Docker image does not include native executables — agents must use the Docker runtime.

### Authentication

All management endpoints require an `Authorization` header with the API key.
Agent RPC endpoints use an `agentSecret` instead.

```
Authorization: Bearer <apiKey>
```

---

## coral-agent.toml

Declarative agent definition file. Every agent the server can run must have one.

```toml
edition = 4   # Required. Supported: 3 or 4.

[agent]
name        = "my-agent"       # 1-32 chars, pattern: ^[a-z0-9_-]*$
version     = "1.0.0"          # Semver
description = "..."            # 1-1024 chars. Shown to LLMs.
summary     = "..."            # 1-256 chars. No markdown.
readme      = "..."            # 1-4096 chars. Markdown OK.
# capabilities = ["resources", "tool_refreshing"]   # optional
# links = { github = "https://github.com/...", docs = "https://..." }  # max 16

# ── DOCKER RUNTIME ──────────────────────────────────────────────────
[[runtimes.docker]]
image   = "my-org/my-agent:latest"
command = []         # Override entrypoint args (optional)
env     = {}         # Extra env vars

# ── EXECUTABLE RUNTIME ──────────────────────────────────────────────
[[runtimes.executable]]
command = ["python", "agent.py"]
args    = []
env     = {}
workingDir = "."

# ── PROTOTYPE RUNTIME (LLM-based, Koog framework) ───────────────────
[[runtimes.prototype]]
model    = "claude-sonnet-4-6"
provider = "anthropic"
prompt   = "You are a helpful agent..."
# mcpServers = [...]   # Additional MCP tool servers

# ── OPTIONS ─────────────────────────────────────────────────────────
[[options]]
name    = "api_key"
type    = "string"
default = ""
# type options: bool | i8 | i16 | i32 | i64 | u8 | u16 | u32 | u64 | f32 | f64 | string | blob

[[options]]
name    = "max_retries"
type    = "i32"
default = 3

# ── MARKETPLACE ──────────────────────────────────────────────────────
# [marketplace]
# price = { ... }    # ERC-8004 pricing settings
```

> String fields (`description`, `readme`, etc.) support inline text, file references (`file:./README.md`), or URL references (`url:https://...`).

---

## REST API Reference

Base: `https://api.coralcloud.ai/api/v1` (or `http://localhost:5555/api/v1` locally)

All management endpoints: `Authorization: Bearer <apiKey>`

### Namespaces

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/local/namespace` | List all namespaces |
| `POST` | `/local/namespace` | Create empty namespace |
| `GET` | `/local/namespace/{ns}` | List sessions in namespace |
| `DELETE` | `/local/namespace/{ns}` | Delete namespace + close sessions |
| `GET` | `/local/namespace/{ns}/extended` | Extended session states |
| `GET` | `/local/namespace/extended` | Extended namespace states |

**Create namespace:**
```json
POST /local/namespace
{ "name": "my-namespace", "deleteOnLastSessionExit": true }
```

### Sessions

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/local/session` | Create (and optionally start) a session |
| `GET` | `/local/session/{ns}/{id}` | Get session state |
| `POST` | `/local/session/{ns}/{id}` | Execute a deferred session |
| `DELETE` | `/local/session/{ns}/{id}` | Close a session |
| `GET` | `/local/session/{ns}/{id}/extended` | Extended session state |

**Create session (full example):**
```json
POST /local/session
{
  "namespace": "my-namespace",
  "agents": [
    {
      "agentName": "orchestrator",
      "registrySource": "local",
      "agentRegistryName": "my-orchestrator",
      "agentRegistryVersion": "1.0.0",
      "options": { "api_key": "sk-..." }
    },
    {
      "agentName": "worker",
      "registrySource": "local",
      "agentRegistryName": "my-worker",
      "agentRegistryVersion": "1.0.0"
    }
  ],
  "settings": {
    "deferred": false,
    "persistenceMode": "None"
  }
}
```

**Session lifecycle states:**
- `PendingExecution` — Created with `deferred: true`, awaiting explicit start
- `Running` — Agents launched and active
- `Closing` — Shutting down, still in memory

**Persistence modes:**
- `None` — Session removed immediately on exit
- `HoldAfterExit` — Session retained in memory after completion
- `MinimumTime` — Retained for a minimum duration

**Execute a deferred session:**
```json
POST /local/session/{ns}/{id}
{ "runtimeSettings": {} }
```

### Registry

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/registry` | List all agents (all sources) |
| `GET` | `/registry/local/{name}/{version}` | Inspect local agent |
| `GET` | `/registry/marketplace/{name}/{version}` | Inspect marketplace agent |
| `GET` | `/registry/linked/{server}/{name}/{version}` | Inspect linked-server agent |

### Puppet API (server-side agent control)

Impersonate any agent in a running session — create threads, send messages, manage participants. Useful for orchestration scripts and testing.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/puppet/{ns}/{sid}/{agent}/thread` | Create thread as agent |
| `DELETE` | `/puppet/{ns}/{sid}/{agent}/thread` | Close thread |
| `POST` | `/puppet/{ns}/{sid}/{agent}/thread/message` | Send message |
| `POST` | `/puppet/{ns}/{sid}/{agent}/thread/participant` | Add participant |
| `DELETE` | `/puppet/{ns}/{sid}/{agent}/thread/participant` | Remove participant |
| `DELETE` | `/puppet/{ns}/{sid}/{agent}` | Terminate agent runtime |

**Create thread:**
```json
POST /puppet/{ns}/{sid}/{agent}/thread
{
  "threadName": "task-discussion",
  "participantNames": ["orchestrator", "worker-1", "worker-2"]
}
```

**Send message:**
```json
POST /puppet/{ns}/{sid}/{agent}/thread/message
{
  "threadId": "thread-uuid",
  "content": "Please analyse the dataset.",
  "mentions": ["worker-1"]
}
```

**Add participant:**
```json
POST /puppet/{ns}/{sid}/{agent}/thread/participant
{ "threadId": "thread-uuid", "participantName": "worker-3" }
```

### Agent RPC (agent → server, uses agentSecret)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/agent-rpc/rental-claim` | Submit rental agent payment claim |
| `POST` | `/agent-rpc/x402` | Request x402 proxy payment |

### Agent Rental (public, no auth)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/agent-rental/catalog` | List available rental agents |
| `GET` | `/agent-rental/wallet` | Get wallet address for payments |
| `POST` | `/agent-rental/reserve` | Reserve rental agents → returns session ID |

---

## MCP Tools (what agents call)

Each agent receives an MCP server from Coral. These are the built-in tools available inside every agent:

| Tool | Parameters | Description |
|------|-----------|-------------|
| `coral_create_thread` | `threadName: string`, `participantNames: string[]` | Creates a new thread and adds listed agents as participants. Returns thread object. |
| `coral_send_message` | `threadId: string`, `content: string`, `mentions?: string[]` | Posts a message. Mentioned agents are notified. Self-mentions are blocked. |
| `coral_wait_for_message` | `threadId: string` | **Blocks** until a message arrives on this thread. 60-second timeout. Returns message. |
| `coral_add_participant` | `threadId: string`, `agentName: string` | Adds an agent to a thread. They receive full message history on join. |
| `coral_remove_participant` | `threadId: string`, `agentName: string` | Removes an agent from a thread. |
| `coral_close_thread` | `threadId: string` | Closes the thread. Returns summary. Closed threads reject further messages. |
| `coral_close_session` | `sessionId: string` | Terminates all agents in the session. |

### Message filtering

When an agent calls `coral_wait_for_message`, it receives messages matching any of:
- **Mentions** — message contains the agent's name in `mentions`
- **Thread** — any message on the specified thread
- **From** — message sent by a specific agent

### 60-second wait timeout

`coral_wait_for_message` has a hard 60-second timeout. Agents must be designed to re-call this tool in a loop if they expect long waits.

---

## Thread & Session Model

### Thread lifecycle

```
Created (Open) → [messages flow] → Closed
```

- Participants added after creation receive **full message history**
- Closed threads throw `ThreadClosedException` on new messages
- Thread state transitions: `Open` → `Closed`

### Session event stream

Events emitted during a session (available via WebSocket / webhooks):

| Event | Fired when |
|-------|-----------|
| `ThreadCreated` | A new thread is opened |
| `ThreadMessageSent` | A message is dispatched to a thread |
| `AgentWaitStart` | An agent begins blocking on `coral_wait_for_message` |
| `AgentWaitStop` | An agent unblocks (message received or timeout) |
| `AgentConnected` | An agent MCP connection is established |

### Link-based blocking

Agents can be configured as "blocking" — they hold their MCP connection open until a dependency agent is `AgentConnected`. This prevents race conditions in orchestration startup.

---

## Common Patterns

### Pattern 1: Orchestrator spawns workers

```
Orchestrator agent starts
  → calls coral_create_thread("task", ["orchestrator", "worker-1", "worker-2"])
  → calls coral_send_message(threadId, "Analyse X", mentions=["worker-1"])
  → calls coral_wait_for_message(threadId)   # blocks
Worker-1 receives mention
  → does work
  → calls coral_send_message(threadId, "Done: result", mentions=["orchestrator"])
Orchestrator unblocks, processes result
```

### Pattern 2: Dynamic team expansion

```
Orchestrator realizes React knowledge needed
  → Server-side: POST /puppet/{ns}/{sid}/orchestrator/thread/participant
    { agentName: "claude-code" }
  → claude-code joins, receives full thread history
  → Orchestrator mentions claude-code in next message
```

### Pattern 3: Deferred session (start when ready)

```
POST /local/session  { settings: { deferred: true }, agents: [...] }
→ Returns sessionId, state: PendingExecution
→ (prepare resources...)
POST /local/session/{ns}/{sessionId}   # trigger execution
→ State transitions to Running
```

### Pattern 4: Puppet API for testing

```python
# Inject a message as a test agent without running a real agent process
requests.post(
    f"{base}/puppet/{ns}/{sid}/test-orchestrator/thread/message",
    headers={"Authorization": f"Bearer {api_key}"},
    json={"threadId": tid, "content": "Run the analysis", "mentions": ["worker"]}
)
```

---

## Key Data Structures

### SessionRequest

```typescript
{
  namespace: string
  agents: AgentConfig[]
  settings?: {
    deferred?: boolean
    persistenceMode?: "None" | "HoldAfterExit" | "MinimumTime"
  }
}

AgentConfig {
  agentName: string                // name within this session
  registrySource: "local" | "marketplace" | "linked"
  agentRegistryName: string        // name in the registry
  agentRegistryVersion: string     // semver
  options?: Record<string, any>    // matches [[options]] in toml
}
```

### SessionStateBase

```typescript
{
  sessionId: string
  namespace: string
  status: "PendingExecution" | "Running" | "Closing"
  createdAt: string                // ISO timestamp
  agents: AgentStateBase[]
}
```

### SessionThread message

```typescript
{
  messageId: string
  threadId: string
  senderName: string
  content: string
  mentions: string[]               // agent names mentioned
  sentAt: string                   // ISO timestamp
  telemetry?: object               // LLM call data if available
}
```

### AgentRegistrySource

```typescript
{
  name: string
  version: string
  source: "local" | "marketplace" | "linked"
  description: string
  summary: string
  capabilities: string[]
}
```

---

## Payment & x402 Flow

Coral supports autonomous agent payments via two mechanisms:

### Rental agents

Agents available for pay-per-use. Clients don't need an API key.

```
GET  /agent-rental/catalog        → list available agents + pricing
GET  /agent-rental/wallet         → get payment wallet address
POST /agent-rental/reserve        → pay + reserve → returns session ID
```

### x402 proxy

Agents can make HTTP calls to external paid APIs through the Coral x402 proxy. The server handles the HTTP 402 payment flow transparently.

```
POST /agent-rpc/x402
Authorization: AgentSecret <agentSecret>
{
  "targetUrl": "https://paid-api.example.com/endpoint",
  "method": "POST",
  "body": { ... }
}
```

### Rental claim

After completing paid work, agents submit a claim:

```
POST /agent-rpc/rental-claim
Authorization: AgentSecret <agentSecret>
{
  "sessionId": "...",
  "completionProof": "..."
}
→ Returns AgentRemainingBudget
```

---

## Error Responses

All errors return `RouteException`:

```typescript
{
  code: number        // HTTP status code
  message: string     // human-readable description
  details?: object    // additional context
}
```

Common codes: `400` invalid request, `403` unauthorized, `404` resource not found.

---

## Quick Reference: Endpoint Summary

```
# Sessions & Namespaces
GET    /api/v1/local/namespace
POST   /api/v1/local/namespace
DELETE /api/v1/local/namespace/{ns}
GET    /api/v1/local/namespace/{ns}
GET    /api/v1/local/namespace/{ns}/extended
POST   /api/v1/local/session
GET    /api/v1/local/session/{ns}/{id}
POST   /api/v1/local/session/{ns}/{id}       # execute deferred
DELETE /api/v1/local/session/{ns}/{id}
GET    /api/v1/local/session/{ns}/{id}/extended

# Registry
GET    /api/v1/registry
GET    /api/v1/registry/local/{name}/{ver}
GET    /api/v1/registry/marketplace/{name}/{ver}
GET    /api/v1/registry/linked/{server}/{name}/{ver}

# Puppet (impersonate agents)
POST   /api/v1/puppet/{ns}/{sid}/{agent}/thread
DELETE /api/v1/puppet/{ns}/{sid}/{agent}/thread
POST   /api/v1/puppet/{ns}/{sid}/{agent}/thread/message
POST   /api/v1/puppet/{ns}/{sid}/{agent}/thread/participant
DELETE /api/v1/puppet/{ns}/{sid}/{agent}/thread/participant
DELETE /api/v1/puppet/{ns}/{sid}/{agent}

# Agent RPC (agentSecret auth)
POST   /api/v1/agent-rpc/rental-claim
POST   /api/v1/agent-rpc/x402

# Rental (no auth)
GET    /api/v1/agent-rental/catalog
GET    /api/v1/agent-rental/wallet
POST   /api/v1/agent-rental/reserve
```

---

## Gotchas

- `coral_wait_for_message` has a **hard 60-second timeout**. Loop agents must handle this and re-call.
- Agents cannot mention themselves (`self-mention prevention`).
- Participants added to a thread after creation receive the **full message history** — design thread creation order accordingly.
- The Docker image does **not** support executable runtimes — use Docker runtime or run Gradle locally.
- `hotReload = true` in registry config enables live re-discovery of `coral-agent.toml` files without restart.
- Optional parameters in MCP tool input schemas can cause issues with structured output in some LLMs — validate tool call shapes before deploying.
- Agent `name` in `coral-agent.toml` must match pattern `^[a-z0-9_-]*$` — no uppercase, no spaces.
- `edition` must be `3` or `4` — any other value causes registry load failure.

---

## Coral Console

Coral Console (formerly Coral Studio) is the official web UI for Coral Server. Use it to create sessions, inspect threads, manage agents, view telemetry, and interact with the payment layer — all without writing API calls directly.

- **Repo:** `https://github.com/Coral-Protocol/console`
- **Stack:** SvelteKit + TypeScript
- **Connects to:** Coral Server REST API, SSE event stream, WebSocket

### Pages / features

| Route | What it does |
|-------|-------------|
| **Workbench** | Main session canvas — create sessions, watch agents run in real time |
| **Thread view** | Inspect message history, participants, and mention flow for any thread |
| **Agent** | Browse registered agents, inspect `coral-agent.toml` details |
| **Server** | Manage server connection, view namespace/session state |
| **Finance** | Wallet address, rental catalog, payment history |
| **Templates / Create** | Save and reuse session templates (pre-configured agent graphs) |
| **Tools / User Input** | Inject messages into a live session as a human participant |
| **Help** | In-app guidance |

---

### Option A — Coral Cloud (managed, no server needed)

The quickest way to start. No self-hosted server required.

1. Sign up at `https://coralcloud.ai`
2. Get your API key from the dashboard
3. The Console is hosted — point it at `https://api.coralcloud.ai` via `PUBLIC_API_PATH`
4. All REST, SSE, and WebSocket traffic routes through the cloud base URL

**Environment variable for cloud:**
```
PUBLIC_API_PATH=https://api.coralcloud.ai
PUBLIC_LOGIN_BEHAVIOUR=token
```

---

### Option B — Local (self-hosted coral-server + console)

Run everything locally. Good for development, air-gapped, or on-prem deployments.

#### Step 1 — Start coral-server

```bash
# Option: Gradle
./gradlew run

# Option: Docker
docker run -p 5555:5555 \
  -e CONFIG_FILE_PATH=/config/config.toml \
  -v /path/to/config.toml:/config/config.toml \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/coral-protocol/coral-server
```

Coral Server binds to `http://localhost:5555` by default.

#### Step 2 — Start Coral Console

```bash
git clone https://github.com/Coral-Protocol/console
cd console
yarn install

# Development (hot-reload)
yarn dev
# → http://localhost:5173

# Production build
yarn build && yarn preview
# → http://localhost:4173
```

#### Step 3 — Console connects automatically

In dev mode, Vite proxies all traffic to `localhost:5555`:

| Console path | Proxied to |
|-------------|-----------|
| `/api/*` | `http://localhost:5555/api/*` |
| `/sse/*` | `http://localhost:5555/sse/*` |
| `/ws/*` | `ws://localhost:5555/ws/*` (WebSocket) |

No extra config needed for local development — it works out of the box.

#### Step 4 — Log in

Enter your `apiKey` (from `config.toml` `[auth]` section) in the login screen.

---

### Environment variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PUBLIC_API_PATH` | string | `/` | Base path for all API calls. Set to `https://api.coralcloud.ai` for cloud, leave as `/` for local proxy. |
| `PUBLIC_LOGIN_BEHAVIOUR` | `"token"` \| `"reload"` | `"token"` | How the console handles auth sessions. `"token"` stores API key in memory; `"reload"` re-authenticates on page load. |

Set these in a `.env` file at the project root:
```
PUBLIC_API_PATH=/
PUBLIC_LOGIN_BEHAVIOUR=token
```

---

### Local vs Cloud decision guide

| | Local | Coral Cloud |
|--|-------|------------|
| **Server to run** | `coral-server` yourself | None — managed |
| **Cost** | Free (self-hosted) | Usage-based |
| **Agent runtimes** | Docker or executable on your machine | Cloud-hosted containers |
| **Data** | Stays on your infra | Processed by CoralCloud |
| **Setup time** | ~5 min | ~1 min |
| **Best for** | Dev, on-prem, enterprise | Prototyping, demos, managed scale |

---

### Full local stack (quick copy-paste)

```bash
# Terminal 1 — coral-server
docker run -p 5555:5555 \
  -e CONFIG_FILE_PATH=/config/config.toml \
  -v $(pwd)/config.toml:/config/config.toml \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/coral-protocol/coral-server

# Terminal 2 — Coral Console
cd console && yarn dev
# Open http://localhost:5173, log in with apiKey from config.toml
```

Minimum `config.toml` to get started:
```toml
[auth]
apiKey    = "dev-key"
secretKey = "dev-secret"

[network]
port = 5555

[registry]
paths    = ["./agents"]
hotReload = true
```

---

## Interactive Onboarding Wizard

When a user wants to get started with CoralOS or set up an agent, run through this wizard. Ask one question at a time, then branch based on the answer. Don't dump all steps at once.

### Step 1 — Deployment target

Ask:
> "Do you want to use **Coral Cloud** (managed, no server to run) or **Local** (self-hosted, full control)?"

**→ Cloud:** Jump to [Cloud onboarding](#cloud-onboarding) below.  
**→ Local:** Continue to Step 2.

---

### Step 2 — Prerequisites check (Local only)

Check or ask the user to confirm:

| Requirement | Check command | Min version |
|-------------|--------------|-------------|
| Docker | `docker --version` | 24+ |
| Node.js | `node --version` | 18+ |
| Git | `git --version` | any |

If Docker is missing → suggest `brew install --cask docker` (macOS), `winget install Docker.DockerDesktop` (Windows), or the official install guide.

If Node is missing → suggest `https://nodejs.org` or `nvm install --lts`.

Once confirmed, continue to Step 3.

---

### Step 3 — Start the server (Local)

Two options. Ask:
> "Use the **CLI quickstart** (recommended for first-time setup) or the **Docker image** (production/on-prem)?"

**CLI quickstart (npx):**
```bash
npx coralos-dev@latest server start -- --auth.keys=dev
```
This starts the server at `http://localhost:5555` with a dev API key of `dev`.  
Console is available at `http://localhost:5555/ui/console`.  
No `config.toml` needed — flags override everything.

**Docker:**
First create a minimal `config.toml` (shown in the [Full local stack](#full-local-stack-quick-copy-paste) section above), then:
```bash
docker run -p 5555:5555 \
  -e CONFIG_FILE_PATH=/config/config.toml \
  -v $(pwd)/config.toml:/config/config.toml \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/coral-protocol/coral-server
```

Wait for the line: `Coral Server started on port 5555`.

---

### Step 4 — Open the Console

Navigate to `http://localhost:5555/ui/console`.  
Log in with your API key (`dev` if you used `--auth.keys=dev`).

You should see the Workbench with an empty session list. The server is running.

---

### Step 5 — Add an agent

Ask:
> "Which language / framework are you using for your agent?"
> - Kotlin (Koog framework — first-party, recommended)
> - Python
> - Other / I have an existing agent

**→ Kotlin/Koog:** Continue to [Koog agent setup](#koog-agent-setup) below.  
**→ Python:** Continue to [Python agent setup](#python-agent-setup) below.  
**→ Existing agent:** Jump to [Linking an existing agent](#linking-an-existing-agent) below.

---

### Cloud Onboarding

1. Sign up at `https://coralcloud.ai`
2. Copy your API key from the dashboard
3. Open the hosted Console at `https://console.coralcloud.ai`
4. Log in with your API key
5. Continue to Step 5 above to add an agent

For agent setup the steps are the same as local — the only difference is agents are launched as cloud containers via Docker runtime definitions in `coral-agent.toml`.

---

## Koog Agent Setup

The Koog framework (JVM/Kotlin) is the first-party agent SDK. Use `npm create koog` to scaffold a new agent.

### Prerequisites

| Requirement | Version |
|-------------|---------|
| JDK | **24** (exact — Koog uses Project Loom virtual threads) |
| Git | any |
| Node.js | 18+ (for `npm create koog` scaffolding only) |

Check JDK version: `java --version`. If wrong version, use [SDKMAN](https://sdkman.io): `sdk install java 24-open`.

### Scaffold

```bash
npm create koog my-agent-name
cd my-agent-name
```

This generates a Gradle project with:
- `coral-agent.toml` — agent definition (this is what Coral reads)
- `build.gradle.kts` — Gradle build
- `src/main/kotlin/Main.kt` — entry point

### Configure the agent

Open `coral-agent.toml`. The key fields to set are the `[[options]]` values — these become environment variables at runtime:

```toml
edition = 4

[agent]
name        = "my-agent"          # lowercase, hyphens ok
description = "..."               # shown to other LLMs
summary     = "..."               # one-liner

[[runtimes.executable]]
path      = "./gradlew"
arguments = ["-q", "run"]

[[options]]
name    = "MODEL_PROVIDER"
type    = "string"
default = "OPENAI"          # OPENAI | OPENROUTER | ANTHROPIC

[[options]]
name    = "MODEL_API_KEY"
type    = "string"
default = ""                # REQUIRED — user must supply at session launch

[[options]]
name    = "MODEL_ID"
type    = "string"
default = "gpt-4o-mini"     # any model ID valid for the chosen provider

[[options]]
name    = "SYSTEM_PROMPT"
type    = "string"
default = "You are a helpful assistant..."

[[options]]
name    = "MAX_ITERATIONS"
type    = "i32"
default = 60

[[options]]
name    = "MAX_TOKENS"
type    = "i64"
default = 150000
```

**Provider values:**
| Value | API key expected | Notes |
|-------|-----------------|-------|
| `OPENAI` | OpenAI key `sk-...` | Default |
| `ANTHROPIC` | Anthropic key `sk-ant-...` | Claude models |
| `OPENROUTER` | OpenRouter key | Routes to many models |

### Link the agent to the registry

From inside the agent directory:

```bash
npx @coral-protocol/coralizer@latest link .
```

This symlinks `coral-agent.toml` into the server's registry paths (default `./agents/`). The server hot-reloads it immediately — you should see the agent appear in the Console under the **Agents** tab.

### Run a test session

In the Console → Workbench → **Create Session**:
1. Add your agent from the registry
2. Fill in `MODEL_API_KEY` in the options panel
3. Click **Run**

Watch the agent connect in the Thread view.

### Build and run locally (without Console)

```bash
./gradlew -q run
```

The agent will connect to the Coral Server automatically using the `CORAL_SERVER_URL` environment variable (defaults to `http://localhost:5555`).

---

## Python Agent Setup

Python agents use the `coral-sdk` package (MCP-based).

### Install

```bash
pip install coral-sdk
# or
uv add coral-sdk
```

### Minimal agent

```python
from coral_sdk import CoralAgent, CoralMCP

async def main():
    async with CoralAgent(server_url="http://localhost:5555", api_key="dev") as agent:
        mcp = CoralMCP(agent)
        thread = await mcp.create_thread("main", participants=["my-python-agent"])
        # ... agent loop here
```

### coral-agent.toml for Python

```toml
edition = 4

[agent]
name        = "my-python-agent"
description = "Python-based Coral agent"
summary     = "Handles X tasks"

[[runtimes.executable]]
command    = ["python"]
args       = ["agent.py"]
workingDir = "."

[[options]]
name    = "OPENAI_API_KEY"
type    = "string"
default = ""
```

Link it the same way: `npx @coral-protocol/coralizer@latest link .`

---

## Linking an Existing Agent

If you already have an agent with a `coral-agent.toml`:

```bash
# From the agent directory
npx @coral-protocol/coralizer@latest link .

# Or manually: copy/symlink the toml into the server's registry path
cp coral-agent.toml /path/to/server/agents/my-agent/coral-agent.toml
```

If `hotReload = true` in the server config, the agent appears immediately. Otherwise restart the server.

Verify: `GET http://localhost:5555/api/v1/registry` — your agent should appear in the list.

---

## Common Setup Issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Agent doesn't appear in registry | `hotReload = false` or wrong `paths` in config | Restart server, or check `registry.paths` includes the agent dir |
| `coral_wait_for_message` times out immediately | Session not started, or agent joined wrong thread | Verify sessionId and threadId match |
| Koog `./gradlew run` fails | Wrong JDK version | `java --version` must show 24. Use SDKMAN to install |
| Console can't connect | Server not running, or wrong API path | Check `http://localhost:5555/api/v1/local/namespace` returns JSON |
| Docker socket permission denied | Docker not running, or socket path wrong | `docker ps` to verify Docker is up |
| `npx coralizer link` not found | npm/Node not installed | `node --version` first, then retry |
