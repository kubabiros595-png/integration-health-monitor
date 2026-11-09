# Data and privacy

All shops, events and secrets in this project are fictional. Nothing comes from a real company or from my current job.

## Webhook contract

`POST /webhook/ihm/events`

Headers: `X-IHM-Shop`, `X-IHM-Timestamp` (Unix seconds), `X-IHM-Signature` (`v1=` + hex HMAC-SHA256 of `"<timestamp>.<raw body>"`).

| Field | Type and rule | Example |
| --- | --- | --- |
| `event_id` | text, `evt_` + 8–40 letters or digits, unique per shop | `evt_8F2KQ1ZT` |
| `event_type` | `order.created` or `customer.updated` | `order.created` |
| `occurred_at` | ISO 8601 date-time | `2026-11-09T19:03:54.000Z` |
| `shop_id` | must equal the `X-IHM-Shop` header | `shop_mossbank` |
| `data.order_id` | text, required for `order.created` | `ORD-1042` |
| `data.total` | number ≥ 0, required for `order.created` | `49.9` |
| `data.currency` | 3 capital letters, required for `order.created` | `EUR` |
| `data.customer_id` | text, required for `customer.updated` | `CUS-311` |
| `data.email_opt_in` | true or false, required for `customer.updated` | `true` |

Unknown extra fields are allowed and stored, so a shop can add fields without breaking the integration.

| Response | When |
| --- | --- |
| `200 {"status":"accepted"}` | stored |
| `200 {"status":"duplicate"}` | same shop + `event_id` already stored, nothing done |
| `400 {"error":"invalid payload","details":"..."}` | every validation problem listed |
| `401 {"error":"unauthorized"}` | unknown shop, bad or missing signature, old, future or missing timestamp. No details, so an attacker learns nothing |
| `500 {"error":"server error"}` | signing secret not configured on our side |

## Tables

| Table | What is in it |
| --- | --- |
| `IHM Events` | accepted events: `event_id`, `shop_id`, `event_type`, `occurred_at`, `received_at`, `payload`, `test_run`, `source` |
| `IHM Request Log` | every request, including rejected ones: `received_at`, `shop_id`, `event_id`, `outcome`, `http_status`, `reason`, `test_run`, `source` |
| `IHM Integrations` | `shop_id`, `shop_name`, `active` |
| `IHM Alerts` | `created_at`, `shop_id`, `shop_name`, `alert_type`, `facts`, `draft_subject`, `draft_body`, `check_passed`, `check_notes`, `status`, `reviewer_note` |
| `IHM Daily Reports` | one row per client per day: counts, error rate, last event, open alerts, status, summary |
| `IHM Intake Cases` | the 43 test cases |
| `IHM Test Results` | one row per test case per run |

`outcome` is one of `accepted`, `duplicate`, `rejected_signature`, `rejected_timestamp`, `rejected_validation`, `rejected_config`.

The request log is the error table: every rejected request has a row with the reason.

## Secrets

- Each shop has its own signing secret, stored only in n8n Variables (`IHM_SECRET_MOSSBANK`, `IHM_SECRET_QUILLMOOR`, `IHM_SECRET_TALLOWICK`).
- The workflow exports in this repository contain only the variable names, never the values.
- Responses never reveal why authentication failed.
- Draft checks block any draft that contains a real secret or a key-like string, remove the draft text before saving, and do not quote it in the notes.

## What the AI receives

Only the client name, the problem type, the computed facts (numbers and times) and the list of things to check. It never receives secrets, payloads, other clients' data or the request log.

## Data protection notes for a real deployment

- Payloads can contain personal data (customer IDs, opt-in status). Keep only what the business needs, and set a retention period for events, the request log and n8n execution history.
- Restrict who can open the alerts table and the n8n instance.
- Remove or restrict the `X-IHM-Test-Run` header, which only exists so the monitor can ignore test traffic.
