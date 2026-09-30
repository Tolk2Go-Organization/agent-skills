# Tolk2Go agent access

Tolk2Go connects customers with sworn and qualified interpreters for on-site, phone and video assignments. This repository lets AI agents
book and manage Tolk2Go interpreters on behalf of a customer account.

- **MCP server:** `https://api.tolk2go.com/api/v1/mcp` (Streamable HTTP)
- **REST API (OpenAPI 3):** `https://api.tolk2go.com/api/v1/spec`
- **Agent guide:** `https://www.tolk2go.com/llms.txt`
- **Agent skill:** [`skills/booking-tolk2go-interpreters`](skills/booking-tolk2go-interpreters/SKILL.md)

## How access works

Search is public. Booking actions belong to a customer account and use an **agent key**:

1. Create an account at [tolk2go.com](https://www.tolk2go.com) or via `POST /api/v1/register`.
2. Log in once to get a 5-minute token:

   ```bash
   curl -s -X POST https://api.tolk2go.com/api/v1/login \
     -H 'Content-Type: application/json' \
     -d '{"email":"you@example.com","password":"..."}'
   ```

3. Use that token to issue a durable agent key (default 90 days, 7–365 allowed). The plaintext key is shown **once**:

   ```bash
   curl -s -X POST https://api.tolk2go.com/api/v1/agent-access-keys \
     -H "Authorization: Bearer $LOGIN_TOKEN" \
     -H 'Content-Type: application/json' \
     -d '{"name":"My assistant","expires_in_days":90}'
   ```

4. Store the returned `key` in your client's secret store and send it as `Authorization: Bearer <key>`.

Keys can be listed, rotated and revoked with a fresh login token. Losing a key never locks you out: log in again and issue a new one.

## Connect an MCP client

```json
{
  "mcpServers": {
    "tolk2go": {
      "type": "http",
      "url": "https://api.tolk2go.com/api/v1/mcp",
      "headers": { "Authorization": "Bearer t2g_agent_..." }
    }
  }
}
```

`initialize` and `tools/list` work without a key, so you can inspect the tools first. Every tool call requires a valid key.

| Tool | Purpose |
| --- | --- |
| `tolk2go_interpreting_list_bookings` | List the account's bookings, or only changes since a timestamp/cursor |
| `tolk2go_interpreting_get_booking` | Read booking, payment, change and next-action state |
| `tolk2go_interpreting_get_responses` | Read interpreters who responded as available |
| `tolk2go_interpreting_create_booking` | Create an interpreter request |
| `tolk2go_interpreting_select_interpreter` | Select an available interpreter (may return a payment link) |
| `tolk2go_interpreting_request_change` | Submit a change through the customer approval flow |
| `tolk2go_interpreting_cancel_booking` | Cancel a booking |

Find languages, situations and available interpreters with the public REST endpoints first. The skill and `llms.txt` describe the full
journey.

## Install the agent skill

```bash
npx skills add Tolk2Go-Organization/agent-skills
```

Or copy [`skills/booking-tolk2go-interpreters/SKILL.md`](skills/booking-tolk2go-interpreters/SKILL.md) into your agent's skills directory.

## Support

support@tolk2go.com · [Terms](https://www.tolk2go.com/en/page/terms-and-conditions)
