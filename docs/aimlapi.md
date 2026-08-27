# Running Agents on AI/ML API

NanoClaw agents can be routed to [AI/ML API](https://aimlapi.com) instead of the Anthropic API — one key, 300+ models (Claude, GPT, Gemini, DeepSeek, and more), with automatic fallback between providers per model.

## How It Works

AI/ML API exposes an Anthropic-compatible `/v1/messages` endpoint alongside its OpenAI-compatible one. The Claude Code CLI (which runs inside agent containers) uses the Anthropic SDK, which reads `ANTHROPIC_BASE_URL` to find the API host. Pointing that variable at AI/ML API is all that's needed — no new provider code, no OneCLI bypass, no blocked hosts. This is the same trick as [Ollama](ollama.md), minus the local-proxy complications: AI/ML API is a normal remote host, so OneCLI's usual credential-proxy path (`ANTHROPIC_AUTH_TOKEN=placeholder` + a proxy-injected `Authorization` header) works unmodified — it's exactly the flow `src/providers/claude.ts` already implements for the real Anthropic API.

```
┌─────────────────────────────┐
│  Agent container            │
│                             │
│  Claude Code CLI            │
│    ↓ ANTHROPIC_BASE_URL     │      ┌──────────────────────┐
│    https://api.aimlapi.com ─┼─────▶│  AI/ML API            │
│    (via OneCLI proxy)       │      │  anthropic/claude-*   │
└─────────────────────────────┘      └──────────────────────┘
```

Verified directly against the live API: both auth styles the Claude Agent SDK can send work —
`x-api-key: <key>` and `Authorization: Bearer <key>` — so the standard OneCLI generic-secret
setup (which injects the latter) needs no changes.

## Setup

`ANTHROPIC_BASE_URL` in `.env` is necessary but **not sufficient** —
`src/providers/claude.ts` (the file that reads it) is only loaded when
`import './claude.js';` is present in `src/providers/index.ts`; standard
installs hitting `api.anthropic.com` never load it. Setup adds that import
for you when it runs the custom-endpoint flow — pick one:

**Fresh install or re-running setup:**

```bash
NANOCLAW_ANTHROPIC_BASE_URL=https://api.aimlapi.com \
NANOCLAW_ANTHROPIC_AUTH_TOKEN=YOUR_KEY \
  pnpm run setup:auto
```

**Existing install, just adding this later** — the same flow via the
standalone re-entry point, no need to redo the rest of setup:

```bash
NANOCLAW_ANTHROPIC_BASE_URL=https://api.aimlapi.com \
NANOCLAW_ANTHROPIC_AUTH_TOKEN=YOUR_KEY \
  pnpm exec tsx setup/index.ts --step provider-auth claude
```

Either creates the OneCLI secret, writes `ANTHROPIC_BASE_URL` to `.env`, and
appends the `claude.js` import automatically — nothing manual to do beyond
that. (Doing it by hand instead: run the `onecli secrets create` command
below yourself, add `ANTHROPIC_BASE_URL=https://api.aimlapi.com` to `.env`,
**and** append `import './claude.js';` to the end of
`src/providers/index.ts` — skipping that last line is the whole bug this
section exists to head off.)

```bash
onecli secrets create --name "AI/ML API" --type generic \
  --value YOUR_KEY --host-pattern "api.aimlapi.com" \
  --header-name "Authorization" --value-format "Bearer {value}"
```

Optionally, attribute this traffic as coming from NanoClaw — a second, non-secret "credential" injecting a static header alongside the real key (the gateway applies every matching rule, not just one):

```bash
onecli secrets create --name "AI/ML API Source" --type generic \
  --value "agent/nanoclaw" --host-pattern "api.aimlapi.com" \
  --header-name "X-AIMLAPI-Source" --value-format "{value}"

onecli secrets create --name "AI/ML API Partner" --type generic \
  --value "part_JVb2uUP2fkNTsmCLGT8ABaZy" --host-pattern "api.aimlapi.com" \
  --header-name "X-AIMLAPI-Partner-ID" --value-format "{value}"
```

Grant the agent access to all secret ids you created. `set-secrets` **replaces** the entire list, so read the current one first and merge in the new ids rather than overwriting it:

```bash
AGENT_ID=$(onecli agents list | jq -r '.data[] | select(.identifier=="<agentGroupId>") | .id')
CURRENT=$(onecli agents secrets --id "$AGENT_ID" | jq -r '[.data[]] | join(",")')
MERGED=$(printf '%s' "$CURRENT,<api-key-secret-id>,<source-secret-id>,<partner-secret-id>" | tr ',' '\n' | sort -u | paste -sd ',' -)
onecli agents set-secrets --id "$AGENT_ID" --secret-ids "$MERGED"
onecli agents secrets --id "$AGENT_ID"
```

## Model Selection

Set `"model"` in the container's `~/.claude/settings.json` (bind-mounted from `data/v2-sessions/<agent-group-id>/.claude-shared/settings.json`) to an AI/ML API Claude model id, e.g. `anthropic/claude-opus-5`. Use the exact id from the [model catalog](https://docs.aimlapi.com) — ids are prefixed by vendor, not bare model names.

## Other model families

AI/ML API also serves GPT, Gemini, DeepSeek, and others via its OpenAI-compatible endpoint — route those through [`/add-opencode`](../.claude/skills/add-opencode/SKILL.md)'s `openai` provider example instead of this doc, which covers the Claude-compatible path only.
