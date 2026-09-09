# Perform MCP

Connect Claude — or any MCP client — to your Perform studio, so it can answer
things like *"who hasn't trained in two weeks, and what did they say in their
last check-in?"* and then act on the answer.

The server is hosted. There is nothing to install and nothing to run.

```
https://perform-api.otherwise.co.il/mcp
```

---

## 1. Create an API key

In Perform, open **Business settings → Integrations** and create an API key.

It looks like `pf_live_…` and **is shown once** — copy it before closing the
dialog.

One key means one studio. Every tool is scoped to it, and there is no way to
reach another studio's data with it.

> Treat the key like a password. Anyone holding it can read and change
> everything in your studio. Do not paste it into a chat, a screenshot, or a
> shared document, and give each person their own rather than passing one
> around.

## 2. Point your client at it

### Claude Code

```bash
claude mcp add --transport http perform https://perform-api.otherwise.co.il/mcp \
  --header "Authorization: Bearer pf_live_..."
```

The word `Bearer` is required — the key on its own is rejected.

### Claude Desktop

Claude Desktop speaks stdio, not HTTP, so it needs a bridge. Add this to
`claude_desktop_config.json` (**Settings → Developer → Edit Config**):

```json
{
  "mcpServers": {
    "perform": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://perform-api.otherwise.co.il/mcp",
        "--header",
        "Authorization: Bearer pf_live_..."
      ]
    }
  }
}
```

Restart Claude Desktop afterwards.

### Cursor, VS Code, and anything else that takes a URL plus headers

```json
{
  "mcpServers": {
    "perform": {
      "url": "https://perform-api.otherwise.co.il/mcp",
      "headers": { "Authorization": "Bearer pf_live_..." }
    }
  }
}
```

### Check it worked

Ask your client to list the tools. You should see 21. A `401` means the header
is missing the `Bearer` prefix or the key has been revoked.

---

## What it can do

**Read (13)** — `search_trainees`, `find_trainee_by_phone`, `get_trainee`,
`list_trainee_programs`, `list_trainee_form_responses`, `list_trainee_weights`,
`list_trainee_subscriptions`, `list_plans`, `list_program_templates`,
`list_forms`, `list_coaches`, `list_tags`, `list_tasks`

**Write (8)** — `create_trainee`, `update_trainee`, `assign_plan`,
`assign_program`, `send_form`, `create_task`, `update_task`, `log_weight`

Phone numbers are accepted in any format: `0501234567`, `+972-50-123-4567` and
`972501234567` all resolve to the same trainee.

## What it deliberately cannot do

A model acts without a human checking each step, so anything that destroys data
or reaches a real person is left out on purpose:

| Not available | Why |
|---|---|
| Delete a trainee | Erases their whole history. Irreversible. |
| Send a message | Pushes to a real trainee's phone. |
| Freeze a plan | Fires a notification that cannot be suppressed. |
| Cancel a plan | Ends someone's paid subscription. |

The tools that *could* notify (`assign_plan`, `assign_program`) always pass
`notify: false`, and `create_trainee` never sends a WhatsApp invite. Deciding to
contact a trainee stays with the coach, in the app.

For those actions, use the coach app.

---

## Notes

- **Idempotency.** `create_trainee` takes an `externalId`. Models retry; with
  that set, a repeat call returns the trainee that already exists instead of
  creating a duplicate.
- **Errors are written to be read by a model.** *"No trainee with id X exists in
  this studio. Use Search trainees or Find trainee by phone to get a valid id."*
  That sentence is what lets a model correct itself, so the server passes it
  through verbatim rather than reducing it to a code.
- **Transport.** Stateless Streamable HTTP. No session to keep alive.
- **claude.ai Connectors are not supported yet.** That surface requires OAuth
  2.1; this server authenticates with a bearer token.

## Trouble

| Symptom | Cause |
|---|---|
| `401` | Missing `Bearer ` prefix, or the key was revoked. |
| No tools listed | Client did not restart, or the config file has a syntax error. |
| `command not found: claude` | The Claude Code CLI is not installed — the desktop app alone does not provide it. |
| Tool call returns a studio you did not expect | The key belongs to that studio. Keys are per studio. |

To revoke a key, delete it in **Business settings → Integrations**. It stops
working immediately.
