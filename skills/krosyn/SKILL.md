---
name: krosyn
description: Build, edit, test and debug Krosyn connectors through the Krosyn MCP server. Use when the user wants to automate work between apps with Krosyn, create or change a connector, write a Krosyn expression ($payload, $lookup, $secret, $connection), shape a webhook body, or find out why a connector run failed.
---

# Krosyn

Krosyn automates the repetitive work between apps. Each automation is a **connector**:
an incoming webhook goes in, Krosyn optionally fetches extra data, reshapes it, and sends an
HTTP request to another app.

You work with Krosyn through its MCP server. Every change you make is saved as a **draft** and
goes live only when published, so you can build and test freely without touching live traffic.

## Connect

Server URL: `https://krosyn.com/api/v1/mcp`

- **Clients with a connector screen** (Claude, ChatGPT, most desktop apps): add the URL. The
  client opens a Krosyn approval page; the user signs in, picks an organization and approves.
- **Editors and CLIs**: the user creates a personal access token in Krosyn under
  *Settings > API Tokens* (write scope to make changes) and passes it as a bearer header:

```bash
claude mcp add --transport http krosyn https://krosyn.com/api/v1/mcp \
  --header "Authorization: Bearer YOUR_TOKEN"
```

Never ask the user to paste a token into the chat. Have them put it in the client config.

Setup for Cursor, VS Code and Windsurf: https://krosyn.com/docs/mcp-integration

## How a connector runs

1. **Trigger**: a condition expression evaluated against the incoming `$payload`. False means
   the run is skipped. Empty means always run.
2. **Process** (optional): an optional GET lookup to an external API (its JSON response becomes
   `$lookup`), then field mappings evaluated **in order**, exposed as `$processed.NAME`. Process
   output is context only; nothing is sent.
3. **Perform**: builds a JSON body and sends it with POST, PUT, PATCH or DELETE to the target
   URL. GET is not a perform method; use the process lookup for reads.

| Variable | Available in |
|---|---|
| `$payload` | everywhere: the incoming JSON body |
| `$secret.NAME` | everywhere: organization secrets, never returned by any tool |
| `$connection.NAME.access_token` | everywhere: stored OAuth2 tokens |
| `$lookup` | process field mappings and perform: the lookup response |
| `$processed.NAME` / `$process.NAME` | later process fields and perform |
| `$item` | inside a `$foreach` template only |

Expressions are allowed in every URL, header value, query parameter value, process field and
perform body field. Full syntax and the function list: [references/expressions.md](references/expressions.md).

## Workflow

1. **Look before you build.** `list_connectors` to see what exists. Prefer `update_connector` on
   an existing connector over creating a near-duplicate.
2. **Check what you may reference.** `list_secrets` and `list_connections` return names only.
   Never invent a secret or connection name. If one is missing, ask the user to create it
   (`create_secret` works for secrets if they give you the value through a safe channel).
3. **Check limits** with `get_plan_limits` before creating connectors or secrets.
4. **Show a short plan first**: trigger, lookup URL, process fields, perform URL, method, body.
   Build after the user agrees, unless they told you to just do it.
5. **Create or update** with `create_connector` / `update_connector`. Always pass
   `example_payload` (realistic values of the right types, covering every field your expressions
   read), and `example_lookup_response` when there is a lookup. The Krosyn editor and the test
   button both run from them.
6. **Test** with `test_connector`, then read the result with `get_event`. Use `test_expression`
   for a single expression against sample data.
7. **Publish** with `publish_draft` once the test looks right, or `discard_draft` to revert.
8. **Enable**: new connectors are created **disabled**. Use `set_connector_status` (safe to
   repeat) rather than `toggle_connector`.
9. **Hand over the trigger URL.** Inbound calls go to
   `POST https://krosyn.com/api/v1/connectors/{connector_id}/trigger`. The user creates the
   endpoint token on the connector page in Krosyn; you cannot create it over MCP.

## Body shapes

`process` and `perform` take a JSON object whose keys are output fields and whose leaf values
are **expression strings**. A literal string needs quotes inside the expression.

```json
{
  "text": "Order $payload.order_id from $processed.full_name",
  "channel": "'#orders'",
  "customer": { "email": "$payload.email", "tier": "default($lookup.tier, 'standard')" },
  "line_items": {
    "$foreach": "$payload.items",
    "template": { "sku": "$item.sku", "qty": "$item.quantity" }
  }
}
```

- Objects and lists nest up to 4 levels.
- `$foreach` + `template` turns a variable-length array into a variable-length array. A for-each
  inside another for-each is not supported.
- Do the shaping in process and keep perform as placement: compute `full_name`,
  `formatted_amount` and so on as process fields, then reference `$processed.*` in perform.
- To round-trip a config exactly, pass back the ConfigField array `get_connector` returned as
  the parameter value itself (a JSON array, never wrapped in an object).

Headers are lists of `{key, value}`:
`"perform_headers": [{"key": "Authorization", "value": "Bearer $secret.TARGET_API_KEY"}]`.
Without a header, the outbound request carries no authentication.

## Debugging a failed run

1. `list_events` for the connector, then `get_event` for the failed one: it shows the request
   Krosyn sent, the response, and which phase failed.
2. Common causes:
   - `Unknown function: ...`: function names are case-sensitive; check `list_functions`.
   - A field resolves to null: the payload does not have that path. Compare with the real
     payload on the event, and guard with `default()` or `coalesce()`.
   - `Circuit breaker is open for URL`: the target failed repeatedly; Krosyn pauses live
     traffic to it. Test runs bypass the breaker, so `test_connector` tells you whether the
     target has recovered.
   - `Execution depth limit exceeded`: connectors are calling each other in a loop.
3. Fix with `update_connector`, re-test, publish. `retry_event` replays the original delivery
   (paid plans only).

The trigger API answers `200 {"success": true}` for every run it **recorded**, including runs
whose target failed. `success` means Krosyn stored the event, not that the target accepted it.
Only a 500 means nothing was recorded and the sender should retry. Check `status` and the event.

## Rules

- Never print, guess or ask for secret values or tokens in the conversation.
- Test runs count as runs against the plan's monthly allowance. Test deliberately, not in a loop.
- Deleting a connector deletes its event history. Confirm with the user first.
- Read the organization's real connectors, secrets and events through the tools. Do not
  assume names or payload shapes.
