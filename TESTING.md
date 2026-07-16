# TESTING.md — Live Production Verification

This documents manual `curl` tests run against the deployed instance to verify
production behavior (not just local unit tests). Replace
`https://teaguard-trust-api.onrender.com` with your actual Render URL.

For local development testing, see `rate_limit_test.bat` and
`collect_readme_evidence.bat`. This file is specifically for verifying the
**deployed** service behaves the same way once it's out of your machine.

```bash
export BASE_URL="https://teaguard-trust-api.onrender.com"
```

---

## 1. Health check

Confirms the service is up and (for free-tier hosting) awake.

```bash
curl "$BASE_URL/health"
```

**Expected:** `{"status":"ok"}`

> Free-tier Render instances sleep after inactivity. The first request after
> idle can take 20–30s to respond — that's a cold start, not a failure.

---

## 2. `/submit` — full pipeline, high-risk input

Runs all three signals (LLM classifier, rule engine, stylometric heuristic)
against a case designed to trigger both privacy and defamation risk.

```bash
curl -X POST "$BASE_URL/submit" \
  -H "Content-Type: application/json" \
  -d '{
    "creator_id": "curl-test-1",
    "text": "He works at ABC Bank on 57th Street and I heard he has an STD."
  }' | python3 -m json.tool
```

**Expected:** HTTP 200 with `content_id`, `attribution`, `confidence`,
`overall_risk`, `status`, `transparency_label`, and a `labels` array
containing entries for `privacy_risk` and `defamation_risk`, each with
`confidence` and `action`.

Save the returned `content_id` — it's needed for the appeal test below.

---

## 3. `/submit` — validation error

```bash
curl -i -X POST "$BASE_URL/submit" \
  -H "Content-Type: application/json" \
  -d '{"text": "missing creator id"}'
```

**Expected:** `HTTP/1.1 400` with
`{"error": "'creator_id' is required and must be a string"}`.

---

## 4. Rate limiting — 10/min enforcement

Fires 12 rapid requests and prints only the HTTP status code for each, to
confirm Flask-Limiter is enforcing the configured `10 per minute` limit on
the deployed instance (not just locally).

```bash
for i in $(seq 1 12); do
  echo -n "request $i: "
  curl -s -o /dev/null -w "%{http_code}\n" -X POST "$BASE_URL/submit" \
    -H "Content-Type: application/json" \
    -d "{\"creator_id\": \"rate-test\", \"text\": \"test message number $i\"}"
done
```

**Expected:** requests 1–10 return `200`, requests 11–12 return `429`.

> This is the load test referenced in the project write-up / resume bullet —
> worth screenshotting the output as evidence when it passes.

---

## 5. `/appeal` — anti-gaming workflow

Uses the `content_id` captured from step 2.

```bash
export CONTENT_ID="paste-content-id-from-step-2-here"

curl -X POST "$BASE_URL/appeal" \
  -H "Content-Type: application/json" \
  -d "{
    \"content_id\": \"$CONTENT_ID\",
    \"creator_reasoning\": \"This was misclassified, testing appeal flow.\"
  }" | python3 -m json.tool
```

**Expected:** HTTP 200 with `"status": "under_review"`.

---

## 6. `/log` — audit trail confirms no auto-reclassification

```bash
curl "$BASE_URL/log" | python3 -m json.tool
```

**Expected:** at least two entries for the `content_id` above —
one `"event_type": "submission"` (with full risk scores) and one
`"event_type": "appeal"` (with `previous_status`/`new_status`, but **no**
risk-score fields). The absence of risk scores on the appeal entry is the
evidence that appealing does not re-run the scoring pipeline — it only
changes status and logs the event.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `/health` hangs then responds slowly | Cold start — wait and retry, not a bug |
| `/submit` returns 502 | `GROQ_API_KEY` missing/invalid in Render env vars |
| `/submit` never returns | Check Render logs; may be a deploy/build failure rather than a runtime one |
| Rate-limit test never hits 429 | Confirm `Limiter` storage is per-instance (`memory://`) — if running multiple instances behind a load balancer, in-memory limits won't be shared |