# Ralph Wiggum for Cursor

An implementation of [Geoffrey Huntley's Ralph Wiggum technique](https://ghuntley.com/ralph/) for Cursor, enabling autonomous AI development with deliberate context management.

> "That's the beauty of Ralph - the technique is deterministically bad in an undeterministic world."

## What is Ralph?

Ralph is a technique for autonomous AI development. In its purest form, it's a loop:

```bash
while :; do cat PROMPT.md | agent ; done
```

The same prompt is fed repeatedly to an AI agent. Progress persists in **files and git**, not in the LLM's context window. When context fills up, you get a fresh agent with fresh context.

## Two Modes

### 🌩️ Cloud Loop (Recommended)

**Fully autonomous.** Spawns Cloud Agents, watches them, chains new ones until task is complete.

```bash
./scripts/ralph-loop.sh
```

Best for: Fire-and-forget, overnight runs, "true Ralph"

### 💻 Local + Handoff

Work in Cursor normally. When context fills up, hooks automatically spawn a Cloud Agent to continue.

Best for: Interactive work where you want hands-on control initially

---

## Quick Start (Cloud Loop)

### 1. Install

```bash
cd your-project
curl -fsSL https://raw.githubusercontent.com/agrimsingh/ralph-wiggum-cursor/main/install.sh | bash
```

### 2. Configure API Key

Get your key from [cursor.com/dashboard](https://cursor.com/dashboard?tab=integrations)

```bash
# Option A: Environment variable
export CURSOR_API_KEY='your-key'

# Option B: Config file (persists)
echo '{"cursor_api_key": "your-key"}' > ~/.cursor/ralph-config.json
```

### 3. Define Your Task

Edit `RALPH_TASK.md`:

```markdown
---
task: Build a REST API
test_command: "npm test"
---

# Task: REST API

## Success Criteria

1. [ ] GET /health returns 200
2. [ ] POST /users creates a user
3. [ ] Tests pass
```

### 4. Start the Loop

```bash
./scripts/ralph-loop.sh
```

Ralph will:
1. Spawn a Cloud Agent
2. Watch it work
3. When it finishes, check if task is complete
4. If not, spawn another agent
5. Repeat until all `[ ]` are `[x]`

---

## Quick Start (Local + Handoff)

### 1. Install

```bash
cd your-project
curl -fsSL https://raw.githubusercontent.com/agrimsingh/ralph-wiggum-cursor/main/install.sh | bash
```

### 2. Configure (Optional for Local-Only)

For automatic Cloud handoff, configure your API key (see above).

### 3. Work in Cursor

1. Open the project in Cursor
2. **Restart Cursor** (to load hooks)
3. Start a conversation: *"Work on the Ralph task in RALPH_TASK.md"*

### 4. Context Limit Handoff

When context fills up (~60k tokens):
- Hooks block further prompts
- Stop hook commits your work
- Cloud Agent is spawned automatically
- You can watch it: `./scripts/watch-cloud-agent.sh <agent-id>`

---

## How It Works

### The malloc/free Problem

> "When data is `malloc()`'ed into the LLM's context window, it cannot be `free()`'d unless you create a brand new context window."

LLM context is like memory:
- Reading files, tool outputs, conversation = `malloc()`
- **There is no `free()`**
- Only way to free: start a new conversation/agent

### Architecture

```
External State (tamper-proof)          Workspace
~/.cursor/ralph/<hash>/                
├── state.md                           RALPH_TASK.md (your task)
├── context-log.md                     .ralph/
├── progress.md          ◄── synced ── ├── progress.md
├── guardrails.md        ◄── synced ── └── guardrails.md
└── .terminated                        
```

State is stored **outside the workspace** so agents can't tamper with tracking.

### Cloud Loop Flow

```
ralph-loop.sh
     │
     ├─► Spawn Cloud Agent 1
     │        │
     │        ▼
     │   Agent works on task
     │        │
     │        ▼
     │   Agent finishes
     │        │
     │   ┌────┴────┐
     │   │         │
     │   ▼         ▼
     │ Complete?  Incomplete?
     │   │         │
     │   ▼         ▼
     │  Done!    Spawn Agent 2
     │             │
     └─────────────┘ (repeat up to 10x)
```

### Local Handoff Flow

```
You in Cursor ──► work ──► context fills ──► hook blocks
                                                  │
                                                  ▼
                                          spawn Cloud Agent
                                                  │
                                                  ▼
                                          (optionally watch)
```

---

## Commands

| Command | Description |
|---------|-------------|
| `./scripts/ralph-loop.sh` | Start autonomous cloud loop |
| `./scripts/watch-cloud-agent.sh <id>` | Watch and chain a specific agent |
| `./scripts/spawn-cloud-agent.sh` | Manually spawn a cloud agent |
| `./scripts/test-cloud-api.sh` | Test API connectivity |

---

## Configuration

### `~/.cursor/ralph-config.json`

```json
{
  "cursor_api_key": "key_xxx",
  "github_token": "ghp_xxx"
}
```

### `RALPH_TASK.md` Format

```markdown
---
task: Short description
test_command: "npm test"           # Optional: verify completion
max_iterations: 20                 # Optional: safety limit
---

# Task Title

## Success Criteria

1. [ ] First thing to complete
2. [ ] Second thing to complete
3. [ ] Third thing to complete
```

**Important:** Use `[ ]` checkboxes. Ralph tracks completion by counting unchecked boxes.

---

## Guardrails (Signs)

When Ralph makes mistakes, add "signs" to `.ralph/guardrails.md`:

```markdown
### Sign: Validate Input
- **Trigger**: When accepting user input
- **Instruction**: Always validate and sanitize
- **Added after**: Iteration 3 - SQL injection
```

Signs are injected into agent context to prevent repeated mistakes.

---

## Monitoring

### Check Agent Status

```bash
curl -s "https://api.cursor.com/v0/agents/<id>" \
  -u "$CURSOR_API_KEY:" | jq .
```

### View Agent Conversation

```bash
curl -s "https://api.cursor.com/v0/agents/<id>/conversation" \
  -u "$CURSOR_API_KEY:" | jq '.messages[-3:]'
```

### List Your Agents

```bash
curl -s "https://api.cursor.com/v0/agents" \
  -u "$CURSOR_API_KEY:" | jq '.agents[] | {id, status, name}'
```

---

## Troubleshooting

### "No RALPH_TASK.md found"

Create a task file in your project root.

### "No API key configured"

```bash
export CURSOR_API_KEY='your-key'
# or
echo '{"cursor_api_key": "key"}' > ~/.cursor/ralph-config.json
```

### Agent stuck on same issue

Add a guardrail to `.ralph/guardrails.md` explaining what to do differently.

### Hooks not firing

Restart Cursor after installing. Check `.cursor/hooks.json` exists.

---

## Learn More

- [Original Ralph technique](https://ghuntley.com/ralph/) - Geoffrey Huntley
- [Context as memory](https://ghuntley.com/allocations/) - The malloc/free metaphor
- [Cursor Hooks docs](https://cursor.com/docs/agent/hooks)
- [Cloud Agents API](https://cursor.com/docs/cloud-agent/api/endpoints)

## License

MIT
