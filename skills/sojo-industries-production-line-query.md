---
name: sojo-production-line-query
description: >-
  Answer a production or machine-performance question about a Sojo Industries line, facility or SKU
  by calling the SOJO Planning Assistant MCP server. Handles the split between production totals
  (NetSuite) and efficiency/OEE (Raven), which is the single most common way this API is used wrong.
api: SOJO Planning Assistant (Victoria)
transport: mcp
endpoint: https://victoria-agent.sojoshield.com/mcp
operations:
  - query_database
  - list_tables
  - describe_table
  - get_table_sample
generated: '2026-08-28'
method: generated
source: mcp/sojo-industries-mcp-tools.json
---

# Query Sojo production and machine data

All four tools below are **real, live and read-only**, verified against
`https://victoria-agent.sojoshield.com/mcp` on 2026-08-28. `initialize` and `tools/list` succeed
with no credential, so you can confirm the tool contract before you call anything.

## Connect

POST to `https://victoria-agent.sojoshield.com/mcp` with:

- `Content-Type: application/json`
- `Accept: application/json, text/event-stream` — **omit this and you get HTTP 406**

Run `initialize` first, keep the `mcp-session-id` response header, and send it on every later call.
Protocol version `2025-06-18`.

## Steps

1. **Ask the question directly with `query_database`.** This is the primary tool and it is
   RAG-backed — it resolves the question to SQL patterns and domain documentation itself. Pass
   `question` in natural language. Narrow it with the optional dimensions when you have them:
   `skus`, `line`, `facility`, `pack_size`, `num_flavors`, `output_case`.

2. **Route the question to the right data source before you ask.** The tool's own description draws
   a hard line and getting it wrong returns a confidently wrong number:
   - "How many did we produce?" → production totals live in **NetSuite Assembly Build views**.
   - Efficiency, uptime, downtime, **OEE** → **Raven** machine data.
   - If the user explicitly says "Raven", honour that even for a production question.
   Say which source you used in your answer.

3. **When `query_database` cannot resolve a noun, introspect.** Call `list_tables` (optionally with
   `database`) to see the schema, then `describe_table` with the required `table_name` for columns
   and types, then `get_table_sample` with `table_name` and a small `limit` to see real rows before
   you reason about them.

4. **Carry `session_id` through the whole exchange** so follow-up questions keep context.

## Rules

- These tools are read-only. There is no write, no delete and no reversal on this surface.
- There is **no rate-limit header** on any response. Pace yourself deliberately; you will get no
  signal before you are throttled.
- The error envelope is `{"data":{"message":...},"error":{"code":"UPPER_SNAKE","details":{}}}` — not
  RFC 9457. Read `error.code`, not the HTTP status alone.
- Never invent a SKU, line, facility or pallet pattern. If `query_database` returns nothing, say so.
