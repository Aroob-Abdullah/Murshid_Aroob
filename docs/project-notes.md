# Murshid: technical notes

Technical documentation for the Murshid service assistant, a four-day cumulative project for **LLM Application Engineering (SDA-AIE-213)** at SDAIA Academy. Developer: Aroob Alosaili.

> **Scope:** the project runs in `mode=mock`. `FakeClient` returns fixed, hand-written answers; it is not a language model. Everything here describes the engineering around a model-shaped interface: validation, contracts, authorization, caching, tests and honest reporting. No external model API is called anywhere in the core project.

## 1. Architecture

```text
question
   │
   ▼
guard_input ──► blocked / invalid / too long / personal data ──► rejected (no model, no tool)
   │ ok
   ▼
build_prompt (versioned teaching template; not executed by the mock)
   │
   ▼
route
   ├─ cancellation words ───────────────► unsupported_action
   ├─ extract_ticket → validate_ticket ──► invalid_schema (before any tool call)
   ├─ status question without reference ─► needs_clarification
   ├─ status question with reference ────► check_status (read-only, ownership checked)
   ├─ the two public FAQ questions ──────► FakeClient.generate
   └─ anything else ─────────────────────► out_of_scope
   │
   ▼
validate_response ──► invalid_output if the reply is malformed
   │
   ▼
run_app (Day 4): exact cache for the two public FAQs only, plus per-request metrics
```

Only the two public FAQ questions can be cached. Application-status requests, malformed tickets and guarded requests never enter the cache.

## 2. Components

| Component | Stage | Role |
|---|---|---|
| `FakeClient.generate` | Day 1 | Finite lookup from three known questions to fixed answers; counts its calls; never touches a network. |
| `ask` | Day 1 | Baseline: rejects empty or non-text input before any client call and returns the response contract. |
| `extract_ticket` | Day 2 | Builds a four-field ticket with keyword rules and a regular expression. It is not language understanding. |
| `validate_ticket` | Day 2 | Accepts only the four agreed keys, allowed values, a real boolean, a well-formed reference, and the clarification rule. |
| `check_status` | Day 2 | Read-only tool: checks reference format, then existence, then ownership, before returning a status. |
| `route` | Day 2 | Rejects cancellations first, then extracts and validates the ticket and chooses the path. |
| `build_prompt` | Day 3 | Produces a versioned prompt template as a visible teaching artifact. |
| `guard_input` | Day 3 | Rejects empty input, over-long input, a short list of blocked phrases, and a 10-digit number. |
| `validate_response` | Day 3 | Checks that `answer` and `code` are text and `tool_calls` is the integer 0 or 1. |
| `run_pipeline` | Day 3 | Guard, prompt, route, response validation, then adds `mode`, `trace_id` and `latency_ms`. |
| `evaluate` | Day 3 | Runs the ten golden cases and reports the pass rate with reasons for any failure. |
| `run_app` | Day 4 | Wraps the pipeline with the exact-match cache and writes one metrics record per request. |

## 3. Data and fields

All data is hand-written and synthetic. It is embedded in the notebook and also written to `synthetic_data.json`.

**Service directory:** one Arabic and one English answer. Three questions are known to the mock client: the two FAQ questions and the Arabic phrase *Help me*.

**Application records:**

| Reference | Owner | Status |
|---|---|---|
| `REF-1001` | `demo-user-1` | `under_review` |
| `REF-1002` | `demo-user-2` | `approved` |
| `REF-9999` | does not exist | used for the negative test |

The session `demo-user-1` is a fixed test setting. It is never taken from the user's question, and it is not real authentication.

**Ticket (Day 2):**

| Field | Allowed values |
|---|---|
| `service` | `service_info`, `application_status`, `other` |
| `city` | `Riyadh`, `Jeddah`, `unknown` |
| `request_id` | `REF-` followed by four digits, or `null` |
| `needs_clarification` | a real boolean, and it must be `true` exactly when the service is `application_status` and the reference is missing |

**Response contract (Day 1):** `trace_id`, `answer`, `model` and `latency_ms`. From Day 3 the pipeline result carries `code`, `answer`, `tool_calls`, `mode`, `trace_id` and `latency_ms`, and from Day 4 also `cache_hit`.

**Result codes:** `ok`, `under_review`, `approved`, `not_found`, `access_denied`, `needs_clarification`, `unsupported_action`, `out_of_scope`, `invalid_schema`, `invalid_output`, `invalid_input`, `input_too_long`, `blocked_instruction`, `unnecessary_personal_data`. These are intended application results, not Python exceptions.

## 4. Validation

- **Input guard:** empty or non-text input, more than 500 characters, the phrases `ignore previous`, `ignore all previous`, `تجاهل التعليمات`, `reveal secrets` and `اكشف الأسرار`, and any 10-digit number.
- **Ticket validation:** runs before any tool call. A malformed ticket returns `invalid_schema`, which is a different failure from `invalid_output` (a malformed reply, caught after routing).
- **Output validation:** a reply with a non-text answer or code, or a tool count other than 0 or 1, is replaced by `invalid_output`.

The guards are small keyword and pattern checks. They are teaching examples, not a complete defence against prompt injection or personal-data leakage.

## 5. Read-only tool and authorization fixture

`check_status` never changes a record. The checks run in a fixed order:

1. The reference must match `REF-` plus four digits.
2. The record must exist.
3. The record's owner must equal the session.

A missing record (`not_found`) and another user's record (`access_denied`) are different internal codes, but the user sees the **same message** for both: *Unable to provide this application status.* The status of another user's record is never shown. The tests also confirm that the records are identical before and after every run.

## 6. Cache rules (Day 4)

| Rule | Behaviour |
|---|---|
| Eligible | Only when the trimmed question equals one of the two public FAQ questions exactly, no ticket override is given, caching is on, and the result code is `ok`. |
| Never cached | Application status, personal questions, any other question, and any rejected input. |
| Key | `(question text, language, model, prompt version, directory version)`. Changing the prompt or directory version creates a new key, so the first request after a change is always a miss. |
| Stored value | The answer and its creation time only, not the full reply, the trace ID or any session data. |
| Expiry | 300 seconds. The tests use an injected clock instead of waiting; a request at exactly 300 seconds is a miss. |
| Metrics | One record per request with `trace_id`, `mode`, `code`, `cache_hit`, `mock_calls`, `tool_calls` and `latency_ms`. The question, reference, owner and answer are never logged. |

**Controlled experiment:** the same English FAQ question three times, with a fresh counter and an empty cache each time.

| Measurement | Without cache | With cache |
|---|---:|---:|
| Requests | 3 | 3 |
| Mock-client calls | 3 | 1 |
| Cache hits | 0 | 2 |
| Hit rate | 0 % | 66.7 % |

The hit sequence is `false, true, true`. These are deterministic call counts for this experiment. They are not a promised speed-up and not a financial saving. A high hit rate is not success by itself: the cache must also return the right answer and never a stale or another request's answer.

## 7. Tests and results

| Stage | What is checked | Result |
|---|---|:---:|
| Day 1 | Three answers, the four-field contract, `training-mock` as the model, and that empty input adds no client call | ✅ |
| Day 2 | Five routed cases, the exact ticket, the tool-call counts, ownership, and unchanged records | ✅ |
| Day 3 | Ten golden cases (10/10), the 500-character and 10-digit guards, response validation, and the two added injection phrases | ✅ |
| Day 4 | Three-request cache experiment, version change, directory-version change, TTL expiry, status exclusion with unchanged cache size, and the ten golden cases rerun through `run_app` (10/10) | ✅ |

The cache tests are reported separately and are not counted inside the ten golden cases. A passing 10/10 means these listed examples passed; it does not measure a live language model.

## 8. Generated outputs

The notebook writes these files to `murshid_outputs/`; copies of the daily results are kept in `results/core/`.

| File | Content |
|---|---|
| `baseline_results.json` | Day 1 answers and contract fields |
| `day2_results.json` | The five Day 2 routed results |
| `day3_results.json` | The ten-case golden report |
| `day4_results.json` | The cache comparison |
| `final_report.json` | Mode, dataset, golden-test report, cache experiment, hypothetical cost and limits |
| `metrics.json` | One record per request, including measured Python time |
| `synthetic_data.json` | The synthetic directory and records |

**Latency** is the time Python spends running the code, in milliseconds. It does not measure model inference, and it changes on every run. **Cost** appears only as an illustration: 1,000 input and 400 output tokens at invented prices of USD 2 and USD 4 per million tokens give USD 0.0036 per request. Those counts and prices are made up for teaching; no tokenizer was used, no provider was called and nothing was billed.

## 9. How to reproduce

1. Open `notebooks/murshid.ipynb` in Google Colab with a Python 3 CPU runtime.
2. Choose **Runtime → Restart session and run all**.
3. Check that execution counts run from 1 to 14, that no cell reports an error, and that the last report cell prints `"checks": "passed"`, `"golden_passed": 10` and a cache comparison of 3 calls without the cache and 1 with it.

No package installation, API key, GPU or model download is needed: the project uses only the Python standard library.

## 10. Limitations and future improvement

- **Limitation:** the mock client can only repeat fixed answers, so no result here says anything about how a real model behaves. Ticket extraction and routing are finite rules, and the guards miss any phrasing or language they do not list.
- **Future improvement:** put a small real model behind the same `generate` interface, then rerun the unchanged golden cases and the cache experiment, and add measured token counts and latency. A second improvement is to grow the golden set with paraphrases and other languages for the injection guard, which the Day 3 documented change showed to be weak.

## 11. Attribution

Completed for **LLM Application Engineering (SDA-AIE-213)**, delivered by SDAIA Academy, instructor Meaad Al-Marri. Reference: [SDAIA Academy on GitHub](https://github.com/SDAIAAcademy). This is a learning project; it is not endorsed or certified by SDAIA.
