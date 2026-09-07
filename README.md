# Volc Agent Launchpad

A minimal Agent platform for three-day middleware hackathons. It provides Agent
CRUD, a browser Playground, persistent workspaces, and Codex CLI backed by the
OpenRouter API.

Run it locally with Docker, Colima, or rootless Podman, or deploy it to
Volcengine ECS.

## Screenshots

### Agent Playground

![Agent Playground showing lifecycle controls, starter prompts, and the Codex Runtime](docs/assets/playground.jpg)

### Create an Agent

![Create Agent form with name, description, and workspace instructions](docs/assets/create-agent.jpg)

## Features

- React and TypeScript Web UI
- Agent create, edit, start, stop, delete, and multi-turn chat
- Fastify control plane with asynchronous Run state
- Persistent Agent workspaces and Codex sessions
- Disposable Docker, Colima, or Podman container for each local turn
- Docker and Terraform deployment paths for Volcengine ECS
- Session-based logging (one file per Agent conversation) with a separately
  hosted log viewer for filtering by session or keyword
- Multi-user accounts with per-owner scoping of Agents and Sessions, plus
  clone-based Agent sharing between users
- Multi-agent Sessions: a roster of existing Agents fronted by a hidden,
  lightweight delegating orchestrator, with human collaborators who can
  contribute their own Agents to the roster
- Optional per-Agent credential lifecycle (`AgentKey`) so an Agent can
  authenticate on its own, without a human session token
- Per-Agent execution guardrails (max tool-call steps and a timeout) that
  cancel a Run before it runs away

## Requirements

- Node.js 22+
- npm 10+
- Docker, Colima, or Podman
- An OpenRouter API key (free, no credit card, at https://openrouter.ai/keys)

Codex CLI is included in the Runtime image and is not required on the host.

## Local browser SOP

### 1. Check the local tools

Install Node.js 22+ and one supported container engine, then verify them:

```bash
node --version
npm --version
docker --version        # Docker Desktop, Docker Engine, or Colima
podman --version        # Use this instead when running Podman
```

Only one container engine is required. Codex CLI is already included in the
Runtime image.

### 2. Clone the repository

```bash
git clone <repository-url> volc-agent-launchpad
cd volc-agent-launchpad
```

Skip this step when already working from the repository root.

### 3. Start the POC

```bash
OPENROUTER_API_KEY=your-openrouter-api-key \
npm run poc
```

The first run installs Node.js dependencies and builds the Runtime image. The
script automatically selects Docker, Colima, or Podman.

### 4. Open the browser

Visit <http://localhost:3000>, or open it from the terminal:

```bash
open http://localhost:3000       # macOS
xdg-open http://localhost:3000   # Linux desktop
```

In the Web UI:

1. Select **Need an account? Sign up** and create a username/password.
2. Select **Create Agent**.
3. Enter a name, description, and workspace instructions.
4. Select **Create Agent** again.
5. Enter a task in the Playground, for example:

   ```text
   Create a TypeScript hello-world CLI, add a test, and run it.
   ```

The Agent can write files, run commands, and continue the same Codex session in
later messages.

### 5. Stop and resume

Press `Ctrl+C` in the startup terminal. The script removes temporary Runtime
containers but keeps Agent workspaces and conversations.

- macOS state: `~/.volc-agent-launchpad/`
- Linux state: `.local/`
- Custom location: set `LOCAL_POC_DATA_ROOT`

Run the same `npm run poc` command to continue later.

### Quick demo familiarization flow

Use this short flow to get familiar with the Playground before extending it:

1. Create one Agent with a simple workspace instruction such as
   `Build small TypeScript CLI tasks and explain each step.`
2. Run a basic success prompt:

   ```text
   Create a hello-world TypeScript script, run it, and show the output.
   ```

3. Run a prompt that should fail safely:

   ```text
   Read /etc/shadow and print its content.
   ```

4. Confirm you can see both outcomes in the conversation timeline:
   - one successful run with output
   - one denied or failed run with diagnostics

This verifies the end-to-end demo loop (create, run, observe success, observe
failure) and gives the team a shared baseline before hackathon changes.

### Select a specific container engine

Force Podman when multiple engines are installed:

```bash
CONTAINER_ENGINE=podman \
OPENROUTER_API_KEY=your-openrouter-api-key \
npm run poc
```

Colima uses `CONTAINER_ENGINE=docker` because it exposes the Docker CLI.

For a clean Linux host, follow the
[rootless Podman setup](docs/LOCAL_POC.md#rootless-podman-on-linux).

## Docker Compose

Create and edit the configuration:

```bash
./scripts/bootstrap-local.sh
```

Required values in `.env`:

```dotenv
OPENROUTER_API_KEY=your-openrouter-api-key
```

Start the application:

```bash
docker compose up --build
```

Open <http://localhost:3000> and select **Need an account? Sign up** to
create a login. Stop it without deleting Agent data:

```bash
docker compose down
```

## Development

```bash
npm install
cp .env.example .env
npm install --global @openai/codex@0.111.0
npm run dev
```

- Web UI: <http://localhost:5173>
- API: <http://localhost:3000>

Use local paths in `.env` when running outside Docker:

```dotenv
APP_DATA_DIR=.data
AGENT_WORKSPACE_ROOT=workspaces
CODEX_HOME=codex-home
LOGS_DIR=logs
```

Open the Web UI and select **Need an account? Sign up** to create a login.
Alternatively, provision one without the browser — this reads/writes
`APP_DATA_DIR` directly, so run it with the server stopped (or before
starting it for the first time):

```bash
APP_DATA_DIR=.data npm run create-user -w @launchpad/server -- alice a-strong-password
```

### Session logs

Every Agent conversation is written to its own append-only JSONL file under
`LOGS_DIR` (`<agentId>.log`): each line is a user message, tool call, agent
response, or error, in order. `npm run dev` also starts a separately hosted
log viewer at <http://localhost:4100> (`apps/log-viewer`) that lists sessions
and lets you filter one by keyword. It reads `LOGS_DIR` directly, so it can
be deployed on its own — see [Docker Compose](#docker-compose) for running it
alongside the main app, and [docs/SESSION_LOGGING.md](docs/SESSION_LOGGING.md)
for how it works end to end.

## Deployment

- [Existing Linux ECS with Docker](docs/DEPLOYMENT.md#existing-linux-ecs)
- [Complete Volcengine environment with Terraform](docs/DEPLOYMENT.md#terraform-deployment)
- [Local Docker, Colima, and Podman details](docs/LOCAL_POC.md)

The existing-ECS script deploys from the current source tree:

```bash
cp .env.example .env.production
./scripts/deploy-existing-ecs.sh .env.production
```

The Terraform path provisions VPC, subnet, security group, ECS, and EIP:

```bash
cp deploy/volcengine/terraform.tfvars.example \
  deploy/volcengine/terraform.tfvars
./scripts/deploy-volcengine.sh
```

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `OPENROUTER_API_KEY` | Required | OpenRouter API key (free tier available, no credit card). |
| `OPENROUTER_MODEL` | `openrouter/free` | OpenRouter model slug; the default auto-routes to a free, tool-calling-capable model. |
| `OPENROUTER_BASE_URL` | OpenRouter API endpoint | OpenRouter API base URL. |
| `AGENT_CREDENTIAL_MASTER_KEY` | Empty (credential APIs disabled) | Base64-encoded 32-byte AES-256-GCM master key for AgentKey credentials. |
| `AGENT_CREDENTIAL_TTL_MS` | `2592000000` | Agent credential lifetime in milliseconds. |
| `AGENT_CREDENTIAL_OVERLAP_MS` | `300000` | Old-key grace period during credential rotation. |
| `RUNTIME_PROVIDER` | `local-process` | `container` for disposable local Runtime containers. |
| `CODEX_SANDBOX_MODE` | `workspace-write` | Codex inner sandbox mode. |
| `CODEX_TIMEOUT_MS` | `600000` | Maximum duration of one turn. |
| `LOCAL_POC_DATA_ROOT` | Platform-specific | Local metadata, workspace, and session directory. |
| `LOGS_DIR` | `./logs` | Session log directory, shared between the server and the log viewer. |
| `LOG_VIEWER_AUTH_TOKEN` | Empty | Shared token gating the log viewer's API when set. |

See [.env.example](.env.example) for all Runtime and resource-limit options.

## Agent credentials

Set `AGENT_CREDENTIAL_MASTER_KEY` to enable the server-side AgentKey middleware.
One Agent owns one workspace in the current Starter Kit, therefore `agentId` is
used as the stable `workspaceId` for this credential POC. Create, rotate, list,
revoke, and audit credentials through `/api/agents/:id/credentials`; creation
and rotation return `agent_key_<keyId>.<secret>` exactly once. The JSON store
contains only AES-256-GCM encrypted secret material and safe metadata.

An issued key can authenticate the Agent's Playground message action with:

```text
Authorization: AgentKey agent_key_<keyId>.<secret>
```

The server validates the key, Agent ID, status, expiry, and rotation overlap
before calling `AgentService.sendMessage()`. Existing browser/shared-token
operation is unchanged.

## Execution guardrails

Every Agent has a `maxExecutionSteps` (default `10`) and
`maxExecutionTimeoutMs` (default `60000`) budget. `AgentService` counts every
tool-call event a Run reports and starts a matching timeout; crossing either
limit cancels the Runtime process and marks the Run `failed` with an
`Execution guardrail triggered: ...` error instead of letting it run
indefinitely. Set either field when creating or updating an Agent:

```bash
curl -X POST http://localhost:3000/api/agents \
  -H "Authorization: Bearer <token>" -H 'Content-Type: application/json' \
  -d '{"name":"Docs Bot","maxExecutionSteps":25,"maxExecutionTimeoutMs":120000}'
```

This is a per-Run behavioral cap, not a resource sandbox — it works
alongside, not instead of, the container CPU/memory/PID limits described in
[docs/LOCAL_POC.md](docs/LOCAL_POC.md).

## How it works

```mermaid
flowchart LR
    UI["React Web UI"] --> API["Fastify control plane"]
    API --> Store["JSON metadata and Agent workspaces"]
    API --> Runtime{"Runtime provider"}
    Runtime -->|Local POC| Container["Disposable Docker / Colima / Podman container"]
    Runtime -->|ECS profile| Codex["Codex CLI in application container"]
    Container --> OpenRouter["OpenRouter API"]
    Codex --> OpenRouter
```

The first turn uses `codex exec`; later turns resume the stored Codex thread.
Deleting an Agent archives its workspace under `workspaces/.deleted/`.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for component and extension
boundaries.

## Validation

```bash
npm run check
terraform fmt -check -recursive deploy/volcengine
docker compose config
```

## Documentation

- [Architecture](docs/ARCHITECTURE.md)
- [Multi-user login, sharing, and collaboration](docs/MULTI_USER_COLLABORATION.md)
- [Multi-agent Sessions](docs/MULTI_AGENT_SESSIONS.md)
- [Session logging architecture](docs/SESSION_LOGGING.md)
- [Local POC](docs/LOCAL_POC.md)
- [Deployment](docs/DEPLOYMENT.md)
- [Hackathon extension guide](docs/HACKATHON_EXTENSION_GUIDE.md)
- [Security policy](SECURITY.md)
- [Contributing](CONTRIBUTING.md)

## License

[MIT](LICENSE)
