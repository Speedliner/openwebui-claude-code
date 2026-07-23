# OpenWebUI Claude Code Pipe

Run [Claude Code](https://docs.claude.com/en/docs/claude-code/overview)'s agent loop from inside [Open WebUI](https://github.com/open-webui/open-webui) chats, via the [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python).

This is an Open WebUI **Pipe** that exposes Claude Code as a selectable model. Each chat gets its own isolated workspace directory; agent turns within the same chat resume the same Claude Code session, so context (files, prior tool calls) carries forward.

## Features

- **Full Claude Code agent loop** — Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch (configurable allowlist)
- **Per-chat workspaces** — each `chat_id` gets a sandboxed working directory that persists across turns
- **Dual auth** — bring your own Anthropic **API key** (pay-per-token) *or* a **Claude Pro/Max OAuth token** (bills against your subscription)
- **Streaming UI** — tool calls render inline with previews; generated images/PDFs/CSVs surface as artifacts in the chat
- **Configurable valves** — model, permission mode, tool allowlist, max turns, workspace root

## Requirements

- Open WebUI (any recent version with the Pipes/Functions framework)
- Python deps (auto-installed by Open WebUI from the file header):
  - `claude-agent-sdk>=0.1.60`
  - `anthropic>=0.40.0`
- The `claude` CLI must be available on the host running Open WebUI's Python backend (the SDK shells out to it). Install via `npm install -g @anthropic-ai/claude-code`.

## Installation

1. In Open WebUI, go to **Workspace → Functions → +** (or **Admin Panel → Functions**).
2. Paste the contents of [`claude_agent_pipe.py`](https://github.com/tfriedel/openwebui-claude-code/blob/main/claude_agent_pipe.py) into the editor.
3. Save and enable the function.
4. Open the function's **Valves** and configure auth (one of):
   - `ANTHROPIC_API_KEY` — standard pay-per-token billing
   - `CLAUDE_CODE_OAUTH_TOKEN` — generate via `claude setup-token`; bills against your Pro/Max/Team subscription. **Known issue:** `setup-token` has occasionally issued tokens with a restricted scope (see [anthropics/claude-code#23703](https://github.com/anthropics/claude-code/issues/23703)), causing `401 Invalid bearer token` on every request even though the token looks valid. If this happens, log in interactively instead (run `claude` with no arguments, complete the full browser OAuth flow) and persist the resulting credentials — see [Docker / persistence notes](#docker--persistence-notes) below.
5. A new model named **Claude Code** will appear in the model picker.

## Configuration (Valves)

| Valve                     | Default                                             | Description                                                                        |
| ------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `ANTHROPIC_API_KEY`       | *(env)*                                              | Anthropic API key. Falls back to the backend's env var.                             |
| `CLAUDE_CODE_OAUTH_TOKEN` | *(empty)*                                            | Claude subscription OAuth token. Takes priority over the API key when set.          |
| `MODEL`                   | `claude-haiku-4-5`                                   | Claude model ID (e.g. `claude-haiku-4-5`, `claude-sonnet-4-6`, `claude-opus-4-7`).  |
| `PERMISSION_MODE`         | `bypassPermissions`                                  | `default`, `acceptEdits`, `bypassPermissions`, `plan`, or `dontAsk`.                |
| `ALLOWED_TOOLS`           | `Read,Write,Edit,Bash,Glob,Grep,WebSearch,WebFetch`  | Comma-separated tools auto-approved without prompting.                              |
| `WORKDIR_ROOT`            | `/tmp/claude-agent-pipe`                             | Root directory for per-chat workspaces.                                             |
| `MAX_TURNS`                | `30`                                                 | Max agent turns per user message. `0` disables the cap.                             |

## Auth notes

When both auth methods are present, the OAuth token wins and the API key is unset before invoking the SDK so it can't override.

Per Anthropic's terms: a Claude subscription is for personal use — **don't re-offer subscription auth to other end users** through a shared Open WebUI deployment. For multi-user setups, use API keys.

## Docker / persistence notes

If Open WebUI's backend runs in a container without a volume for `~/.claude` (and `~/.claude.json`), credentials from an interactive `claude` login are lost the next time the container is recreated (rebuild, `docker compose down`, etc.) — even though the `CLAUDE_CODE_OAUTH_TOKEN` **Valve** itself survives fine, since Valves live in Open WebUI's own database, not the container filesystem.

To make an interactive login durable, mount both paths:

```yaml
services:
  openwebui:
    volumes:
      - open-webui:/app/backend/data
      - claude-code-config:/root/.claude
      - ./claude-code-auth/.claude.json:/root/.claude.json   # pre-create as an empty file on the host

volumes:
  open-webui:
  claude-code-config:
```

Then log in once inside the running container:

```bash
docker exec -it <container> /usr/local/lib/python3.11/site-packages/claude_agent_sdk/_bundled/claude
```

(path depends on your Python/SDK install — check with `pip show claude-agent-sdk` if it's elsewhere)

With credentials persisted this way, leave the `CLAUDE_CODE_OAUTH_TOKEN` valve empty — the pipe falls back to whatever the backend environment / `~/.claude` already provides.

## License

MIT
