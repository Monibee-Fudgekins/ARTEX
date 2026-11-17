# subscription-bridge

Drive ARTEX with your **Claude Code subscription** instead of a pay-per-token API key.

ARTEX normally needs an `ANTHROPIC_API_KEY` (or OpenAI key) and talks to the model
over the Messages API. This bridge lets ARTEX use the model through the official
`claude` CLI — the same subscription you use interactively — with no API key.

## How it works

```
ARTEX (container)  --Anthropic /v1/messages (non-streaming)-->  bridge.py (host)
                                                                    |
                                                          runs:  claude -p  (your subscription)
                                                                    |
bridge returns a normal Anthropic Messages response  <--  structured_output (content blocks)
```

- The bridge exposes a minimal **non-streaming** Anthropic `/v1/messages` endpoint.
- For each call it renders ARTEX's system prompt, tool list and transcript into one
  prompt, runs `claude -p --output-format json --json-schema ... --allowedTools ""`
  as a **pure reasoner**, and maps the structured result back into a Messages
  response (text / `tool_use` blocks, `stop_reason`, token usage).
- **ARTEX keeps its own tool loop.** The bridge never executes tools. The `claude`
  CLI runs with its own tools disabled, so it only *decides* the next turn; ARTEX
  executes every tool itself, under its own scope rules, approval gates and budgets.

## Why this is legitimate (and what it is not)

- It uses the official `claude` CLI, i.e. your Claude Code subscription the way the
  CLI is meant to be driven programmatically.
- It does **not** read, extract, or replay any credential or OAuth token, and does
  **not** call `api.anthropic.com` directly. There is no key handling in the bridge.
- It is subject to your Claude Code plan's usage/rate limits, and to Anthropic's
  terms for your account. An autonomous multi-agent pentest can burn through a lot
  of model usage — watch your limits.

## Run it

1. **Start the bridge on the host** (where you are signed in to Claude Code):
   ```bash
   ./subscription-bridge/run-bridge.sh        # listens on 0.0.0.0:8745
   ```
   It runs a one-turn self-test, then serves until you Ctrl-C it. Keep it running
   while ARTEX works.

2. **`.env` is already wired** for the bridge:
   ```
   ARTEX_LLM_PROVIDER=anthropic
   ARTEX_LLM_BASE_URL=http://host.docker.internal:8745
   ARTEX_LLM_STREAM=false
   ANTHROPIC_API_KEY=subscription     # any non-empty value; the bridge ignores it
   ```
   `docker-compose.override.yml` adds `host.docker.internal:host-gateway` so the
   container can reach the bridge on the host.

3. **Start ARTEX:**
   ```bash
   docker compose up -d --build
   docker compose logs artex | grep "Setup token"
   ```
   Open http://localhost:8787.

### Running ARTEX natively (no container)
If you run the Go binary directly on the host, set `ARTEX_LLM_BASE_URL=http://127.0.0.1:8745`
instead (no `host.docker.internal` needed).

## Environment knobs

| Var | Default | Meaning |
| --- | --- | --- |
| `BRIDGE_ADDR` | `0.0.0.0` | bind address |
| `BRIDGE_PORT` | `8745` | bind port |
| `CLAUDE_BIN` | `claude` | path to the Claude Code CLI |
| `BRIDGE_CALL_TIMEOUT` | `600` | seconds per model turn before 504 |

## Model choice (BRIDGE_MODEL) and the safety classifier

The `claude` CLI routes calls through Claude Code's safety classifier. With the
**default model**, ARTEX's security-testing planner prompts can be refused — the
model returns *"a safety classifier withheld my response"* and ARTEX logs *"goal
decomposition produced nothing."* Setting **`BRIDGE_MODEL=opus`** (the launcher's
default) clears that on authorized security-testing prompts: the planner then
emits the expected `set_goals` / `add_task_scope` / `add_intent` tool calls.

Use the bridge only for authorized testing of systems you own or are permitted to
test — the same rule that applies to ARTEX itself.

## Known gap: non-streaming tool dispatch

The bridge runs ARTEX in non-streaming mode (`ARTEX_LLM_STREAM=false`), and the
decomposition/planner calls return correct tool_use blocks, but ARTEX's
non-streaming path does not reliably persist those tool calls (no intents/scope get
created, so no workers spawn). ARTEX is built around streaming; a streaming (SSE)
bridge is the likely fix. Until then the planner runs but the autonomous loop does
not fully execute. (For the full offensive loop today, a real Anthropic/OpenAI API
key with streaming is the supported path.)

## Limits / caveats

- **Non-streaming only.** `ARTEX_LLM_STREAM=false` is required (set in `.env`); the
  bridge does not implement SSE. You lose live token counts during a run.
- **Stateless per turn.** Each call re-renders the whole transcript into a fresh
  `claude -p` process, so prompt-cache reuse and thinking-block signatures from the
  native API are not carried across turns. Correct, just less token-efficient.
- **Latency.** Each turn spawns a `claude` process; expect a few seconds of overhead
  per turn on top of model time.
- The model id ARTEX reports (`claude-subscription`) is only a label; the actual
  model is whatever your Claude Code session uses.
