# Sample results

These fictional examples come from the demo run on 9 November 2026. Both alerts were saved as `pending review`. Nothing was sent to anyone.

## Silence: Quillmoor Books

**What the rules found:** no event for 300 minutes. The usual gap for this shop is 15 minutes, so the limit is 60 minutes (3 × 15 = 45, raised to the 60-minute floor).

Facts given to Claude:

```json
{"minutes_silent":300,"hours_silent":5,"events_last_7_days":652,"last_event_at":"2026-11-09 13:48 UTC","usual_gap_minutes":15,"threshold_minutes":60}
```

**Subject:** Quillmoor Books: no recent events from your store

> Hi Quillmoor Books team,
>
> We have not received an event from your store for 300 minutes, and it usually sends one about every 15 minutes.
>
> Could you check the following on your side:
> - Is the webhook still enabled in your store admin?
> - Did the store change the endpoint address, domain, or plugin version?
> - Does the store's webhook log show failed deliveries since the last successful event?
>
> Could you reply with what you find? We will confirm on our side once events start arriving again.
>
> Best regards,
> Customer Success

**Code check:** passed. Both numbers (300, 15) are in the facts.

## Error spike: Tallowick Candles

**What the rules found:** 10 of 11 requests rejected in the last 60 minutes (90.9%, limit 10%). The most common reason was a signature mismatch, so code chose the signing-secret checks.

Facts given to Claude:

```json
{"window_minutes":60,"requests":11,"rejected":10,"error_rate_pct":90.9,"threshold_pct":10,"top_reason":"rejected_signature: signature does not match","top_reason_count":10}
```

**Subject:** Webhook requests from Tallowick Candles being rejected

> Hi Tallowick Candles team,
>
> In the last 60 minutes, 10 of 11 webhook requests from your store were rejected, an error rate of 90.9%. The most common reason given was that the signature did not match.
>
> Could you check the following on your side:
> - Was the webhook signing secret changed or regenerated on the store side?
> - Is a proxy or plugin changing the request body before it is sent?
>
> Please reply with anything you find, even if it looks like nothing has changed. We will confirm on our side once events arrive again.
>
> Best regards,
> Customer Success

**Code check:** passed. All numbers (60, 10, 11, 90.9) are in the facts.

## Daily health report

| Client | Status | Summary |
| --- | --- | --- |
| Mossbank Ceramics | Healthy | 72 events accepted in 24 h, 3 duplicates ignored, last event less than 1 h ago. |
| Quillmoor Books | Problem | 76 events accepted in 24 h, last event 5 h ago, 1 alert(s) waiting for review. |
| Tallowick Candles | Problem | 70 events accepted in 24 h, 10 rejected (12.5%), mostly signature, last event less than 1 h ago, 1 alert(s) waiting for review. |

The 24-hour error rate for Tallowick (12.5%) is lower than the 90.9% in the alert, because the report covers the whole day and the alert covers only the last hour.
