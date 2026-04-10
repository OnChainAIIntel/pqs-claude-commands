---
description: Pre-flight prompt quality check via PQS — 8-dimension score + 3 actionable fixes
argument-hint: <prompt text>
---

You are running the `/pqs-score` command. The user wants you to score the prompt text in `$ARGUMENTS` against the public PQS HTTP API and display the result in a compact, readable format.

Follow these steps in order. Do not skip any.

## 1. Load the API key from `~/.pqs/config`

Use the Read tool to read `~/.pqs/config`. It is KEY=VALUE format and contains at minimum:

```
PQS_API_KEY=PQS_<base64url>
PQS_API_BASE=https://pqs.onchainintel.net
```

- If the file does not exist, or `PQS_API_KEY` is missing/empty, print **exactly** this message and stop:

  > No PQS key found. Run: `curl -fsSL https://pqs.onchainintel.net/install.sh | bash`

- Otherwise extract `PQS_API_KEY` and `PQS_API_BASE` (default `PQS_API_BASE` to `https://pqs.onchainintel.net` if absent).

## 2. Validate the argument

If `$ARGUMENTS` is empty or whitespace-only, print:

> Usage: `/pqs-score <prompt to score>`

and stop.

## 3. Call the scoring endpoint

Use the Bash tool to POST to `${PQS_API_BASE}/api/score` with:

- Header: `Authorization: Bearer ${PQS_API_KEY}`
- Header: `Content-Type: application/json`
- Body: a JSON object `{"prompt": "<$ARGUMENTS>"}` — JSON-escape the prompt text properly. Use `jq` if available, otherwise `python3`:

```bash
# jq path (preferred)
PAYLOAD=$(jq -Rs '{prompt: .}' <<<"$ARGUMENTS")

# python3 fallback
PAYLOAD=$(python3 -c 'import json,sys; print(json.dumps({"prompt": sys.argv[1]}))' "$ARGUMENTS")

curl -sS -X POST "${PQS_API_BASE}/api/score" \
  -H "Authorization: Bearer ${PQS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD"
```

Important:
- Do **not** echo `PQS_API_KEY` to the terminal or include it in any output shown to the user.
- Capture both the HTTP status code and the JSON body — use `curl -sS -o /tmp/pqs-resp.json -w '%{http_code}'` so you can branch on status.

## 4. Handle the response

- HTTP 401 → print: `Invalid or expired PQS API key. Re-run install.sh to refresh it.` and stop.
- HTTP 402 → print: `This endpoint now requires x402 payment. Your free-tier key was not accepted.` and stop.
- HTTP 4xx (any other) → print the `error` field from the JSON body and stop.
- HTTP 5xx → print: `PQS server error — try again in a moment.` and stop.
- HTTP 200 → continue.

The 200 response shape (PQS v2.0):

```json
{
  "pqs_version": "2.0",
  "prompt": "...",
  "vertical": "general",
  "score": {
    "total": 18,
    "out_of": 80,
    "grade": "D",
    "percentile": 23,
    "dimensions": {
      "clarity": 6,
      "specificity": 3,
      "context": 2,
      "constraints": 1,
      "output_format": 2,
      "role_definition": 1,
      "examples": 2,
      "cot_structure": 1
    }
  },
  "top_fixes": [
    "...",
    "...",
    "..."
  ],
  "upgrade": "...",
  "powered_by": "..."
}
```

## 5. Display the result

Print this exact layout. **All dimensions must be rendered as a 10-character ASCII bar**: fill `█` for each point of the score and `░` for each point remaining. Example: `6/10` → `██████░░░░`.

```
─────────────────────────────────────
PQS SCORE: <total>/<out_of> — Grade <grade> (<percentile>th percentile)
─────────────────────────────────────
Dimensions:
  Clarity          <bar>  <clarity>/10
  Specificity      <bar>  <specificity>/10
  Context          <bar>  <context>/10
  Constraints      <bar>  <constraints>/10
  Output Format    <bar>  <output_format>/10
  Role Definition  <bar>  <role_definition>/10
  Examples         <bar>  <examples>/10
  CoT Structure    <bar>  <cot_structure>/10

Top fixes:
  → <top_fixes[0]>
  → <top_fixes[1]>
  → <top_fixes[2]>
─────────────────────────────────────
```

Rules:
- Render dimensions in the exact order shown above (not whatever order the API returns).
- Pad dimension labels so the bars align (label column is 17 characters wide including the 2-space indent).
- If `top_fixes` has fewer than 3 items, print only what exists. If it's empty, omit the "Top fixes:" block entirely.
- If any numeric field is missing or null, render `—` in its place rather than fabricating a value.
- Stop after printing the block. Do not add tips, commentary, follow-up questions, or a summary of what you just did.
