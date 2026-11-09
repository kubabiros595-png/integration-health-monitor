# Test results

All data is fictional. I set the expected result for every case before running the tests.

## 1. Webhook intake

`IHM Intake Tests` reads the 45 cases in `intake_test_cases.csv`, builds each request, signs it the way a shop would, and sends it to the live production URL of `IHM Webhook Intake`. It then reads the request log and the events table for that run.

A case passes only if all three match:
- the HTTP status the sender received
- the outcome written to the request log (exactly one log row per request)
- whether the event was stored or not

| Run | Cases | Passed | Failed |
| --- | --- | --- | --- |
| 2026-11-09 20:04 (Warsaw) | 43 | 38 | V03, D01, D02, D04, F11 |
| 2026-11-09 20:07 (Warsaw) | 43 | 43 | none |
| 2026-11-09 20:16 (Warsaw), after adding F15 and D06 | 45 | **45** | none |

### What the cases cover

| Group | Cases | Examples |
| --- | --- | --- |
| Valid | V01–V11 | both event types for all three shops, total of 0, an extra unknown field, timestamp 290 s old, 60 s in the future, non-ASCII text |
| Signature | S01–S08 | wrong secret, body changed after signing, header missing, no `v1=` prefix, unknown shop, another shop's secret, same JSON pretty-printed, timestamp changed after signing |
| Timestamp | T01–T05 | 310 s old, 10 min old, 400 s in the future, header missing, not a number |
| Validation | F01–F14 | each required field missing, unknown event type, number sent as text, negative total, lowercase currency, shop in body differs from header, invalid JSON, wrong `event_id` format, bad date |
| Size | F15 | body larger than 64 KB gets 413 |
| Duplicates | D01–D06 | same event twice, exact replay of a signed request within 5 minutes, same `event_id` with different content, same `event_id` from another shop (a new event), the same request three times at the same moment |

### Why the first run failed and what changed

| Case | Got | Cause | Change |
| --- | --- | --- | --- |
| V03, D04 | V03 duplicate, D04 accepted | The test sent requests in parallel, so D04 (a repeat of V03) arrived first | Test sends one request at a time in a loop |
| D01, D02 | D01 duplicate, D02 accepted | Same cause | Same change |
| F11 | 401 instead of 400 | The n8n HTTP Request node re-encoded a non-JSON body sent as `application/json`, so the received bytes differed from the signed bytes. The intake was right to reject them | That case is sent as `text/plain` |

Also changed: every request is now signed right before it is sent. In the first run all requests were signed at the start and some waited about 10 seconds, so the timestamp edge cases were less precise than intended (T01 arrived 320 s old instead of 310 s).

The parallel run showed a real limit: two identical requests at the same moment can both pass the duplicate lookup. I fixed it afterwards with a check after saving (the first stored copy wins, later copies delete themselves) and added D06. In the third run, both extra copies in D06 were caught by that new check (log reason "parallel request"), and only one row was stored.

## 2. Silence and error-spike detector

`IHM Monitor Tests` builds made-up histories and sends them to the same `Detect silence and error spikes` node that the hourly run uses. It does not read or write the real tables.

| Run | Scenarios | Passed | False alarms | Missed |
| --- | --- | --- | --- | --- |
| 2026-11-09 20:04 | 18 | 18 | 0 | 0 |
| 2026-11-09 20:07 | 18 | 18 | 0 | 0 |
| 2026-11-09 20:16 | 18 | **18** | **0** | **0** |

The first run's results were not saved to the results table, because the table was missing two columns. The results above come from that execution's output.

| Scenarios | What they check |
| --- | --- |
| M01 | three shops at once: one normal, one silent, one error spike |
| M02–M03 | busy shop, normal and silent for 4 hours |
| M04–M05 | low-traffic shop (an event every 2 h): quiet 3 h is normal, quiet 7 h alerts |
| M06–M07 | 59 and 61 minutes of silence against a 60-minute limit |
| M08–M10 | exactly 10% errors (no alert), 15% (alert), 2 of 3 rejected (too few to judge) |
| M11 | duplicates are not counted as errors |
| M12–M13 | new integration still learning; no events at all in 7 days |
| M14–M16 | same alert still pending (skipped), earlier alert 7 h old (new alert), earlier alert rejected (new alert) |
| M17–M18 | validation errors at 50%; silence and error spike together |

## 3. Draft checks

The same run sends hand-written drafts to the `Check draft` node. No AI is called.

| Run | Cases | Passed | Bad drafts let through |
| --- | --- | --- | --- |
| 2026-11-09 20:04 | 10 | 10 | 0 |
| 2026-11-09 20:07 | 10 | 10 | 0 |
| 2026-11-09 20:16, after adding C11–C14 | 14 | **14** | **0** |

All 10 passed on the first run, but reading the notes showed a problem: for the draft that contained the real signing secret (C05), the note "numbers not in the facts" listed digits taken from the secret. The draft was blocked, but part of the secret would have been saved in the alerts table.

Changes:
- when a secret or key-like string is found, the note no longer lists numbers
- the draft text is replaced with `[removed: the draft contained a secret or key]` before saving
- C05, C06 and C10 now also fail if the draft is not removed or if any 4-character piece of the secret appears in the notes

Later I added CSM rules to the check: no promised fix or deadline (C11), no call (C12), no free work (C13). C14 makes sure "feel free to reply" still passes.

## 4. Demo run with the AI step

`IHM Demo Data` loads a fictional 7-day history:
- Mossbank: healthy
- Quillmoor: silent for the last 5 hours
- Tallowick: every request rejected for the last 50 minutes after a "secret change"

| Run | Result |
| --- | --- |
| Monitor, first run | 2 alerts: Quillmoor silence (300 min, limit 60) and Tallowick error spike (90.9%, 10 of 11). No alert for Mossbank. Both Claude drafts passed the code checks. |
| Monitor, run again right after | 0 new alerts, 0 AI calls. Both problems were skipped as already open. |
| Daily report | Mossbank Healthy, Quillmoor Problem, Tallowick Problem |

The drafts are in `sample-results.md`.

## Limits

- 45 + 32 cases test the rules as written. They do not show how the thresholds would perform on real traffic.
- Draft quality was reviewed by hand on 2 live drafts.
