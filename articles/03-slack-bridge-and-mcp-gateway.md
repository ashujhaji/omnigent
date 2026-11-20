# Building a Multi-Agent Platform on Omnigent — Part 3: The Slack Bridge & the MCP Gateway

> **Series**
> 1. [Platform design & setting up Omnigent](./01-platform-design-and-omnigent-setup.md)
> 2. [The Agent Kit: how every team builds agents on one platform](./02-agent-kit.md)
> 3. **The worker: Slack bridge & MCP gateway** ← you are here
> 4. [Evals: measuring and gating agent quality](./04-evals.md)

Parts 1 and 2 gave us an Omnigent server that can run agents, and an Agent Kit
that builds those agents. But nobody can talk to them yet except through the
web UI, and they can't see any of your company's data.

This part builds the two worker services that fix that:

* the **Slack bridge**, which turns a Slack thread into an Omnigent session, and
* the **MCP gateway**, which gives every agent one shared, audited way into
  your internal systems.

![Platform architecture](./platform-architecture.svg)

Both live on the **Omnigent Worker** side. Neither contains any prompts or
agent logic; that all stays in the Agent Kit.

---

# Part A — The Slack bridge

## 1. What the bridge is (and isn't)

The bridge is a small, stateless Python service (3.12, `asyncio`, `httpx`,
`asyncpg`) that sits between Slack and Omnigent.

| It does | It does not |
|---|---|
| Receive Slack events over Socket Mode | Contain prompts, skills or routing logic |
| Map one Slack thread ↔ one Omnigent session | Run agents (the Omnigent runner does) |
| De-duplicate, batch and serialize messages | Call internal systems (agents do, via MCP) |
| Pick which agent handles a message | Decide *how* to answer |
| Post the final answer back, with feedback buttons | Store conversation history (Omnigent does) |

Keep this boundary strict. The moment routing or answering logic leaks into
the bridge, you have two places to change every behaviour, and only one of
them is versioned with the agent.

## 2. Architecture

```mermaid
flowchart LR
    S([Slack]) -- Socket Mode --> IN[Ingress<br/>ack · loop guard]
    IN --> DD[Dedup]
    DD --> CL[Classifier<br/>mention or DM?]
    CL --> CO[Coalescer<br/>batch rapid messages]
    CO --> SE[Session service<br/>thread ↔ session]
    SE --> TU[Turn dispatcher<br/>one active turn per session]
    TU -- REST --> OM[(Omnigent server)]
    OM -. SSE .-> ST[Live progress]
    OM -- items --> DE[Delivery]
    DE --> S
    RC[Reconciler] -. recovers dropped turns .-> DE
    DB[(Bridge Postgres)] --- DD & SE & TU
```

The only coordinator is **Postgres**. No Redis, no queue. That means you can
run two or more bridge instances behind Slack's Socket Mode and they
coordinate through row claims and unique constraints.

The code splits cleanly into layers:

```
bridge/
├── surfaces/      # Slack in/out: ingress, parsing, slack_client, feedback buttons
├── services/      # classifier, dedup, coalescer, session, turn, delivery, registry, identity
├── engine/        # base.py (AgentEngine interface) + omnigent.py (the only Omnigent-aware file)
├── background/    # dispatcher, sweeper, reconciler, runtime watcher
├── db/            # models, queries, Alembic migrations
└── registry.yaml  # friendly agent names → Omnigent agent ids
```

Everything Omnigent-specific lives behind one `AgentEngine` interface in one
file. If you ever swap the engine, or add a second one, nothing else moves.

## 3. A message, step by step

1. **Ingress.** Socket Mode delivers the event. The bridge acknowledges within
   Slack's 3-second window, adds an instant 👀 reaction so the user knows it was
   seen, and drops anything authored by a bot (loop guard).
2. **Dedup.** Slack retries events. The bridge checks an in-memory LRU, then
   claims the event in the database with `INSERT … ON CONFLICT DO NOTHING`. A
   `UNIQUE` constraint on the turn's dedup key is the final backstop.
3. **Classify.** Deterministic and explicit: only an `@mention` or a DM becomes
   a turn. A plain reply in the thread is ignored. No LLM in this step, so it
   is predictable and free.
4. **Batch.** People often send three short messages in a row. The coalescer
   holds messages for a short quiet window and flushes them as **one** turn.
5. **Session.** Look up the session for this `(channel, thread_ts)`. If none
   exists, create it in three phases: reserve a row, `POST /v1/sessions`, then
   let the runtime warm up in the background. Otherwise resume the existing one.
6. **Context.** Fetch thread messages since the last bookmark (including image
   attachments, sent inline as base64) so the agent sees what happened in the
   thread between mentions.
7. **Dispatch.** At most one active turn per session. The message is posted
   with `response_id = turn.id`, which makes the send **idempotent**: a retry
   refers to the same task and never runs the agent twice.
8. **Stream.** SSE is used only to paint live progress in Slack. It is
   best-effort.
9. **Deliver.** The final answer is built from the **stored session items**,
   not from the stream. If a stream drops or an instance dies mid-turn, the
   reconciler finds the turn and delivers it from the items.
10. **Idle close.** A watcher stops a session's runtime after ~15 minutes of
    inactivity, freeing runner capacity. The next mention resumes it.

> **Design rule:** the stream is for humans watching; the stored items are the
> truth. Every delivery path reads items, so a lost connection never loses an
> answer.

## 4. Talking to Omnigent

The whole integration is a handful of REST calls:

| Purpose | Call |
|---|---|
| Create a session | `POST /v1/sessions` |
| Find a session by label (orphan recovery) | `GET /v1/sessions?limit=100` |
| Send a message / stop a runtime | `POST /v1/sessions/{id}/events` |
| Live progress | `GET /v1/sessions/{id}/stream` (SSE) |
| Final answer | `GET /v1/sessions/{id}/items` |
| Cost, model, runner status | `GET /v1/sessions/{id}` |
| Sub-agent tree | `GET /v1/sessions/{id}/child_sessions` |
| Share with the human who asked | `PUT /v1/sessions/{id}/permissions` |
| Hand off to another agent | `POST /v1/sessions/{id}/switch-agent` |

Creating a session:

```python
payload = {
    "agent_id": agent_id,               # from registry.yaml
    "title": title,
    "labels": {                         # lets us find the session again
        "bridge.slack_channel_id": channel_id,
        "bridge.slack_thread_ts": thread_ts,
    },
    # Optional runtime placement, forwarded verbatim:
    # "host_type": "external", "host_id": "...", "workspace": "/data/threads/<c>/<t>"
}
session = await client.post("/v1/sessions", json=payload)
```

Sending a message, with the turn id as the idempotency key:

```python
payload = {
    "type": "message",
    "data": {
        "role": "user",
        "content": [
            # images first, then text
            {"type": "input_image", "image_url": f"data:{mime};base64,{b64}"},
            {"type": "input_text", "text": text},
        ],
        "response_id": turn_id,         # retry-safe: same id = same task
    },
}
await client.post(f"/v1/sessions/{session_id}/events", json=payload)
```

**Auth.** The bridge runs as one service identity and sends
`Authorization: Bearer $OMNIGENT_BOT_TOKEN`. To let a person open the full
transcript from Slack, it maps the Slack user to their SSO email and grants
them access to the session it created.

**Which agents work?** Only agents registered as built-ins on the server (Part
1, `OMNIGENT_BUILTIN_AGENT_DIRS`). Agents created ad hoc in the web UI are
scoped to their creator, so the bot identity gets a 404 on them.

## 5. Choosing the agent: `registry.yaml`

The bridge needs to know which agent should answer. That's a small YAML map,
loaded at startup, with **no LLM call**:

```yaml
# registry.yaml
agents:
  router:
    agent_id: ag_xxxxxxxxxxxxxxxx      # from GET /v1/agents (Part 1, §3.6)

  code:
    agent_id: ag_yyyyyyyyyyyyyyyy
  coder:                               # a synonym: same agent id
    agent_id: ag_yyyyyyyyyyyyyyyy

  triage:
    agent_id: ag_zzzzzzzzzzzzzzzz
    suppress_narration: true           # deliver only the final answer
    final_marker: "<<final>>"          # the item carrying this marker is the answer
    feedback: true                     # 👍 / 👎 / 💬 buttons
    cost: false                        # hide the cost/model footer
    summary: false                     # no "session wrapped up" message
    allow_dm: false
    default_slack_channels: "C0123ABC456, C0987XYZ654"

  chart:
    agent_id: ag_wwwwwwwwwwwwwwww
    suppress_narration: true
    allow_dm: true
```

Resolution order for a message:

1. **Named dispatch.** If the first word is a registered name, use it:
   `@agent triage …`, `@agent run triage …` and `/agent triage …` all mean
   the same thing.
2. **Channel default.** Otherwise, if the channel is in some agent's
   `default_slack_channels`, use that agent. An on-call channel can talk to
   `triage` without anyone typing its name.
3. **Fallback.** Otherwise, the default router agent.

Two consequences worth knowing:

* Synonyms (`code` / `coder`) are a bridge concern. The Agent Kit only knows
  each bundle's canonical name.
* Agent ids are content-derived. After you change an agent's bundle and
  restart the server, re-check the ids with `GET /v1/agents` and update the
  registry if one changed.

### Per-agent delivery flags

| Flag | Why it exists |
|---|---|
| `suppress_narration` | Agents sometimes emit "Let me check the logs…" before the real answer. With this on, only the final item is delivered. |
| `final_marker` | The agent writes a fixed marker in its final message; the bridge delivers the last item containing it, and strips it. This stops a late "nothing more to add" remark from replacing a real answer. |
| `feedback` / `cost` / `summary` | Customer-facing threads shouldn't show internal cost footers or ops-style wrap-up messages. |
| `default_slack_channels` | Use channel IDs, not names: they're already in the event payload and survive renames. |
| `allow_dm` | Some agents are only for specific channels. |

## 6. Setting up the bridge

### 6.1 Create the Slack app

1. Create a new app at api.slack.com → **From scratch**.
2. Turn on **Socket Mode** and create an **app-level token** with
   `connections:write`. This becomes `SLACK_APP_TOKEN` (`xapp-…`).
3. Under **OAuth & Permissions**, add bot scopes. A typical set:
   `app_mentions:read`, `chat:write`, `reactions:write`,
   `channels:history`, `groups:history`, `im:history`,
   `files:read` (image attachments), `files:write` (posting chart images),
   `users:read`, `users:read.email` (identity mapping), and `commands` if you
   add a slash command.
4. Under **Event Subscriptions**, subscribe to `app_mention` and `message.im`.
5. Install the app to the workspace. The **Bot User OAuth Token** is
   `SLACK_BOT_TOKEN` (`xoxb-…`).
6. Invite the bot to the channels where it should answer.

### 6.2 Create an Omnigent service identity

With auth enabled on the server, create a bot user for the bridge and mint a
token for it. That's `OMNIGENT_BOT_TOKEN`. On a single-user local server
(`OMNIGENT_AUTH_ENABLED=0`) set `OMNIGENT_AUTH_MODE=single_user` and skip this.

### 6.3 Configure `.env`

```dotenv
# Required
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
DATABASE_URL=postgresql://bridge:<password>@postgres:5432/bridge
OMNIGENT_SERVER_URL=http://<omnigent-server-host>:8000
DEFAULT_AGENT_ID=ag_xxxxxxxxxxxxxxxx        # the router

# Multi-user Omnigent (default)
OMNIGENT_AUTH_MODE=multi_user
OMNIGENT_BOT_TOKEN=<service-identity-token>

# Optional
OMNIGENT_UI_BASE_URL=https://<public-omnigent-url>   # "View full session" links
HEALTH_PORT=8080
CONSOLE_PORT=8081
LOG_LEVEL=INFO

# Optional runtime placement, forwarded verbatim on session create
OMNIGENT_HOST_TYPE=            # managed | external
OMNIGENT_HOST_ID=              # external: which runner host
WORKSPACE_BASE_PATH=           # per-thread dir: <base>/<channel>/<thread_ts>/
```

The bridge fails loudly at startup if a required variable is missing. Never
commit `.env`.

`WORKSPACE_BASE_PATH` gives each Slack thread a durable working directory on
the runner host (local disk on one host, a shared mount such as EFS across
several). It must match the `os_env.cwd` your agents use in their
`config.yaml`.

### 6.4 Run it

```yaml
# docker-compose.yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: bridge
      POSTGRES_USER: bridge
      POSTGRES_PASSWORD: ${BRIDGE_DB_PASSWORD}
    volumes: [pg-data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U bridge -d bridge"]
      interval: 5s
      retries: 5

  bridge:
    build: .
    env_file: .env
    environment:
      DATABASE_URL: postgresql://bridge:${BRIDGE_DB_PASSWORD}@postgres:5432/bridge
    depends_on:
      postgres: { condition: service_healthy }
    ports:
      - "8080:8080"   # /health
      - "8081:8081"   # read-only console

volumes:
  pg-data:
```

```bash
docker compose up -d --build
curl -s localhost:8080/health
```

Database migrations (Alembic `upgrade head`) run in-process on every boot, so
there is no separate migration step. When several instances start at once,
Alembic's lock on the version table lets one migrate while the others wait.

The image is `python:3.12-slim`, runs as a non-root user, and has a
`HEALTHCHECK` on `/health`. Because Socket Mode is an outbound WebSocket, the
bridge needs **no inbound port from Slack**: only the health and console ports,
which you can keep private.

## 7. Hard-won lessons

* **Ack first, work later.** Slack retries anything not acked in 3 s. Ack and
  react before any database or Omnigent call.
* **Idempotency at every boundary.** Event dedup at ingress, one active turn
  per session, and `response_id = turn.id` on send. Retries become harmless.
* **Deliver from items, not the stream.** SSE drops. Items don't.
* **Deterministic routing.** First-word lookup plus channel defaults is boring,
  and that's why it's easy to debug.
* **Treat the database schema as code.** Use migrations. Hand-edited live
  tables drift from the migration history and bite you on the next deploy.
* **Use channel IDs everywhere.** Names change; IDs don't.

---

# Part B — The MCP gateway

## 8. Why one gateway?

Agents need data: warehouse queries, logs, APM metrics, tickets, docs, files.
You could give each agent its own MCP servers and credentials. Instead we run
**one HTTP MCP server** that wraps every internal system, for these reasons:

* **Credentials live in one place.** Agents only see a URL. Keys stay on the
  gateway host (env vars or a secrets manager).
* **Least privilege by construction.** The gateway exposes *only* the safe
  operations: SQL is SELECT/WITH only, object storage is read-only behind an
  allowlist.
* **One audit log.** Every tool call is logged with the calling agent and
  session.
* **Agents stay portable.** A skill says "query the warehouse"; whether that's
  Databricks or Snowflake is the gateway's business.

Rule of thumb from Part 2: **consume a vendor's official MCP directly when one
exists.** Build into the gateway only for internal APIs, or where you need a
guard the vendor MCP doesn't offer.

## 9. Structure

One process, one `FastMCP` instance, one folder per backend:

```
mcp-gateway/
├── app.py                    # the ONLY FastMCP() — loads every controller
├── config.yaml               # non-secret settings per tool
├── platform/
│   ├── warehouse/   controller.py  client.py  tests/
│   ├── logs/        controller.py  client.py  tests/
│   ├── metrics/     controller.py  client.py  tests/
│   ├── ticketing/   …
│   ├── docs/        …
│   ├── object-store/…
│   ├── memory/      …            # vector search over your docs
│   └── charts/      …            # renders a branded PNG
└── shared/
    ├── mcp_scaffold/             # load_sibling, tool_error, truncate
    ├── observability/            # log_tool_call wrapper, request context
    └── config/                   # env → config.yaml → secrets manager
```

Each `platform/<x>/` has two files with a strict split:

* **`client.py`** knows *how* to call the backend: auth, timeouts, guards,
  error shapes.
* **`controller.py`** defines the tool functions and a `register(mcp)`
  function. No module-level `FastMCP()`.

The gateway knows **how**; the Agent Kit knows **what** (which table, which
query, which playbook). Domain knowledge never lands in the gateway.

> **Naming gotcha:** don't make `platform/` a Python package (no
> `__init__.py`). It would shadow the standard library's `platform` module.
> Load controllers by file path instead, as below.

## 10. Writing a tool

A complete metrics tool:

```python
# platform/metrics/controller.py
from fastmcp import FastMCP
from shared.mcp_scaffold import load_sibling, tool_error
from shared.observability import log_tool_call

client = load_sibling(__file__, "client")   # NOT `import client` (name clash across platforms)


def metrics_run_query(query: str) -> dict:
    """Run a read-only metrics query and return {"rows": [...], "row_count": n},
    or {"error": "..."} on failure."""
    if not query or not query.strip():
        return tool_error("query must not be empty")
    result = client.run_query(query)
    return tool_error(result) if isinstance(result, str) else result


def register(mcp: FastMCP) -> None:
    mcp.tool()(log_tool_call("metrics_run_query", metrics_run_query))
```

Conventions that pay off:

* **Prefix tool names with the backend** (`warehouse_run_sql`,
  `ticketing_create_issue`). All tools share one namespace.
* **Return errors as data** (`{"error": "..."}`) instead of raising, so the
  agent can read the error and recover.
* **The docstring is the tool description** the model sees. Write it for the
  model: inputs, output shape, failure shape.
* **Wrap every tool in `log_tool_call`** for uniform logging.
* **Truncate large results.** A 5 MB log dump will blow the agent's context.

### Guards belong in the client

The warehouse client refuses anything that isn't a single read:

```python
_READ_ONLY_PREFIX = re.compile(r"\A(SELECT|WITH)\b", re.IGNORECASE)

def check_read_only(sql: str) -> bool:
    """True only if the first real token is SELECT/WITH, it is a single
    statement, and no write keyword hides behind a CTE."""
    ...
```

Note the CTE case: many SQL dialects allow
`WITH x AS (...) DELETE FROM …`, so a prefix check alone isn't enough.
Reject stacked statements and write keywords too.

## 11. Wiring it together: `app.py`

```python
# app.py
import hmac
import importlib.util
import os
from pathlib import Path

from fastmcp import FastMCP
from starlette.middleware import Middleware
from starlette.responses import JSONResponse, PlainTextResponse

# BearerAuthASGIMiddleware and RequestContextASGIMiddleware are shown below.

ROOT = Path(__file__).resolve().parent

def _load_controller(path: Path):
    spec = importlib.util.spec_from_file_location(f"{path.parent.name}_controller", path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    return module

app = FastMCP("mcp-gateway")

for name in ["warehouse", "logs", "metrics", "ticketing", "docs",
             "object-store", "memory", "charts"]:
    _load_controller(ROOT / "platform" / name / "controller.py").register(app)


@app.custom_route("/health", methods=["GET"])
async def health(request):
    return JSONResponse({"status": "ok"})


def main():
    app.run(
        transport="http",
        host="0.0.0.0",
        port=8931,
        middleware=[
            Middleware(BearerAuthASGIMiddleware, token=os.environ["MCP_GATEWAY_TOKEN"]),
            Middleware(RequestContextASGIMiddleware),
        ],
    )
```

Adding a backend is one new folder plus one entry in that list.

### Authentication

Put a **Bearer-token check in raw ASGI middleware**, ahead of MCP dispatch:

```python
class BearerAuthASGIMiddleware:
    def __init__(self, app, token):
        self.app, self.token = app, token

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http" or scope.get("path") == "/health":
            return await self.app(scope, receive, send)
        header = dict(scope.get("headers", [])).get(b"authorization", b"")
        expected = f"Bearer {self.token}".encode() if self.token else b""
        # Fail closed if no token is configured; constant-time compare.
        if not self.token or not hmac.compare_digest(header, expected):
            return await PlainTextResponse("Unauthorized", 401)(scope, receive, send)
        await self.app(scope, receive, send)
```

Why raw ASGI rather than FastMCP's own middleware? FastMCP middleware runs
*after* the HTTP layer accepts the request, so a rejection shows up as
`HTTP 200` with a JSON-RPC error instead of a real `401`.

You may be tempted to skip auth because the gateway sits in a private subnet.
Network isolation is a good second layer, but it's not a substitute: anything
else inside that network (including any agent with a shell tool) can reach
the port. Keep the token check on and serve over TLS if traffic leaves the
host.

### Knowing who called

The Agent Kit stamps `X-Agent-Id: <agent>` on every MCP declaration (Part 2,
§7.4). A second middleware binds that header, plus `X-Session-Id` when
present, into a per-request context, so every `log_tool_call` event carries
the agent and session. One query then answers "which agent ran this SQL, in
which Slack thread?"

## 12. Configuration and secrets

Settings resolve in a fixed order: **env var → `config.yaml` → secrets
manager**. Non-secret settings go in `config.yaml`; secrets never do:

```yaml
# config.yaml (non-secret only)
tools:
  memory:
    variables:
      QDRANT_URL: ""             # empty = resolve from env / secrets manager
      MEMORY_SPACE_KEY: KB
    settings:
      sync: { enabled: true, interval_minutes: 5 }
      chunking: { size: 1000, overlap: 150 }
  charts:
    settings:
      timeout_seconds: 30
```

## 13. Running it

```yaml
# docker-compose.yml
services:
  qdrant:                         # only if you use the memory tool
    image: qdrant/qdrant:v1.12.4
    volumes: [qdrant_storage:/qdrant/storage]
    restart: unless-stopped

  mcp:
    build: .
    ports: ["8931:8931"]
    env_file: .env                # backend credentials + MCP_GATEWAY_TOKEN
    environment:
      QDRANT_URL: http://qdrant:6333
    depends_on: [qdrant]
    restart: unless-stopped

volumes:
  qdrant_storage:
```

```bash
docker compose up -d --build
curl -s localhost:8931/health      # {"status":"ok"}
```

## 14. Connecting agents to the gateway

This is where Part 2 pays off. Declare the gateway **once** in the Agent Kit:

```yaml
# agent-kit/shared/mcp/internal-mcp.yaml
name: internal-mcp
transport: http
url: http://<mcp-gateway-host>:8931/mcp
headers:
  Authorization: "Bearer ${INTERNAL_MCP_TOKEN}"
```

Include it from any agent that needs it:

```yaml
# agent-kit/agents/triage/manifest.yaml
includes:
  shared:
    mcp: [internal-mcp]
```

Then:

1. Rebuild (`python build_bundle.py build --all`).
2. Make sure `INTERNAL_MCP_TOKEN` is set in the **runner's** environment.
   Omnigent resolves `${VAR}` on the runner when it loads the MCP declaration.
3. Restart the Omnigent server so it re-registers the bundle.

Including the declaration exposes **every** tool on the gateway. Narrow it per
agent with a guardrail policy in that agent's `config.yaml`. This is the
chart agent from Part 2, which may only call the chart tools:

```yaml
guardrails:
  policies:
    chart_tools_only:
      type: function
      function:
        path: omnigent.policies.builtins.cel.cel_policy
        arguments:
          expression: >-
            event.type == "tool_call" &&
            event.data.name.matches("(?i)^mcp__") &&
            !event.data.name.matches("(?i)^mcp__.*__charts_(render|list_types)$")
            ? {"result": "DENY", "reason": "Only the chart tools are allowed."}
            : {}
```

---

## 15. Putting it all together

```mermaid
sequenceDiagram
    autonumber
    actor U as Slack user
    participant B as Slack bridge
    participant O as Omnigent server
    participant R as Runner (triage agent)
    participant M as MCP gateway
    participant W as Warehouse

    U->>B: @agent triage order 123 failed at checkout
    B-->>U: 👀
    B->>O: POST /v1/sessions (agent_id = triage)
    B->>O: POST /events (response_id = turn id)
    O->>R: run turn
    R->>M: warehouse_run_sql(SELECT …)  [X-Agent-Id: triage]
    M->>M: auth · read-only guard · log
    M->>W: SELECT …
    W-->>M: rows
    M-->>R: {"rows": [...]}
    R-->>O: final answer + <<final>>
    B->>O: GET /items
    B-->>U: answer · 👍 👎 💬
```

## 16. Try it yourself

**Slack bridge**

1. `curl localhost:8080/health` returns healthy.
2. In a channel the bot is in, post `@agent hello`. You should see 👀 within a
   second, then a reply in the thread.
3. Post `@agent chart …` and confirm the chart agent answers (check the
   session in the Omnigent UI: the agent name should be `chart`).
4. Post a plain reply in the same thread **without** a mention. The bot should
   ignore it.
5. Send three quick messages that each mention the bot. You should get **one**
   batched answer, not three.
6. Mention the bot in a channel listed under `default_slack_channels` for
   `triage`, without typing an agent name. `triage` should answer.
7. Restart the bridge container mid-answer. The reply should still arrive once
   the bridge is back (the reconciler delivers it from the stored items).

**MCP gateway**

1. `curl localhost:8931/health` returns `{"status":"ok"}`.
2. Call `/mcp` without a token and confirm you get **HTTP 401**, not a 200 with
   an error body.
3. List the tools with an MCP client, for example:
   ```bash
   npx @modelcontextprotocol/inspector
   ```
   Connect to `http://localhost:8931/mcp` with your Bearer token and check every
   expected tool appears.
4. Call `warehouse_run_sql` with `DELETE FROM orders` and with
   `WITH x AS (SELECT 1) DELETE FROM orders`. Both must be rejected.
5. Ask an agent in Slack a data question, then find the matching tool-call
   log line on the gateway and confirm it carries the right agent id.
6. Ask the `chart` agent to query the warehouse. The guardrail should deny the
   call, and the agent should ask you to paste the data instead.

---

## What's next

In **Part 4** we close the loop with **evals**: collecting real sessions,
judging them with calibrated LLM judges, replaying golden cases against a
candidate bundle with recorded tool responses, and using that as a gate
before a new agent release reaches Slack.
