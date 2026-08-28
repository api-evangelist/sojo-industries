---
name: sojo-dieline-pallet-check
description: >-
  Check whether a customer's carton dieline and pallet pattern can actually run on a Sojo Industries
  packaging line, by uploading the drawing and calling the two packaging-engineering MCP tools.
api: SOJO Planning Assistant (Victoria)
transport: mcp+rest
endpoint: https://victoria-agent.sojoshield.com/mcp
operations:
  - analyze_dieline
  - analyze_pallet_pattern
rest_operations:
  - 'POST /upload-image'
generated: '2026-08-28'
method: generated
source: mcp/sojo-industries-mcp-tools.json + openapi/sojo-industries-victoria-agent-openapi.json
---

# Check a dieline and pallet pattern against a Sojo line

This is the flow Sojo's own tool descriptions instruct an assistant to run: **always** use
`analyze_dieline` when a user uploads or shares a carton, dieline or packaging drawing.

## Steps

1. **Upload the drawing.** `POST https://victoria-agent.sojoshield.com/upload-image` with
   `Content-Type: application/octet-stream` and the raw image bytes as the body. An empty body
   returns 400. Keep the returned image identifier.

   Note: this operation declares **no security requirement** in the provider's OpenAPI, unlike every
   conversation route. There is also no published delete-image operation — once uploaded, you have no
   documented way to remove it. Do not upload anything the customer would not want retained.

2. **Analyse the carton.** Call `analyze_dieline`. `pack_size` is the only **required** field.
   Supply whatever else you can read off the drawing rather than guessing: `case_length`,
   `case_width`, `case_height`, `minor_flap_length`, `major_flap_top`, `is_corrugate`, `board_type`,
   `can_size`, and `target_line` for the line it must run on. Pass `image_id` from step 1 and
   `session_id` to keep context. Put anything you inferred rather than read into `extraction_notes`.

3. **Confirm extracted values with the human before acting on them.** Both tools take a `confirmed`
   flag. Show the customer the dimensions you extracted from their drawing, get agreement, then
   re-call with `confirmed` set. Dimensions misread off a drawing are the expensive failure here.

4. **Analyse the pallet pattern.** Call `analyze_pallet_pattern`. `pack_config` is the only
   **required** field. Add `can_size`, `shipper_type`, `shipper_length`, `shipper_width`,
   `shipper_height`, `cases_per_layer`, `layers_per_load`, `picks_per_layer`, `shipper_pattern`,
   `pallet_pattern` and `target_palletizer` where known. The tool checks the pattern against **SOJO's
   commissioned patterns** — a pattern that is geometrically valid can still be unavailable on a
   given palletiser.

5. **Report both results together.** A dieline that runs on the target line but whose pallet pattern
   is not commissioned is not a workable job.

## Rules

- Both tools are read-only analyses. Nothing here books, schedules or commits work.
- Never state a dimension the customer did not give you and the tool did not return.
- Never assert a line or palletiser is compatible on your own reasoning — report only what the tools
  returned.
- No idempotency key exists on `POST /upload-image`. A retry after a timeout can create a second
  stored object.
