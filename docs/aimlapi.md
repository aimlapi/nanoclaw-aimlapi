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

```env
ANTHROPIC_BASE_URL=https://api.aimlapi.com
```

Register the key — same pattern as [`add-opencode`](../.claude/skills/add-opencode/SKILL.md)'s examples, `Authorization: Bearer {value}`:

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
```

Grant the agent access to all secret ids you created (`set-secrets` replaces the list, not appends):

```bash
onecli agents set-secrets --id <agent-id> --secret-ids <api-key-secret-id>,<source-secret-id>
```

## Model Selection

Set `"model"` in the container's `~/.claude/settings.json` (bind-mounted from `data/v2-sessions/<agent-group-id>/.claude-shared/settings.json`) to an AI/ML API Claude model id, e.g. `anthropic/claude-opus-5`. Use the exact id from the [model catalog](https://docs.aimlapi.com) — ids are prefixed by vendor, not bare model names.

## Other model families

AI/ML API also serves GPT, Gemini, DeepSeek, and others via its OpenAI-compatible endpoint — route those through [`/add-opencode`](../.claude/skills/add-opencode/SKILL.md)'s `openai` provider example instead of this doc, which covers the Claude-compatible path only.
