---
description: Score a prompt with PQS (Prompt Quality Score) via the public HTTP API
argument-hint: <prompt text>
---

You are running the `/pqs-score` command. The user wants you to score the prompt text in `$ARGUMENTS` against the public PQS HTTP API and print the result in a compact, readable format.

Do the following steps in order. Do not skip any.

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
- HTTP 402 → print: `This endpoint now requires x402 payment. Your free-tier key was not accepted.` and stop. (Shouldn't happen for the free Bearer path, but handle it.)
- HTTP 4xx (any other) → print the `error` field from the JSON body and stop.
- HTTP 5xx → print: `PQS server error — try again in a moment.` and stop.
- HTTP 200 → continue.

## 5. Display the result

The 200 response shape (from the existing `/api/score` route):

```json
{
  "pqs_version": "1.0",
  "prompt": "...",
  "vertical": "general",
  "score": {
    "total": 28,
    "out_of": 40,
    "grade": "B",
    "percentile": 70,
    "dimensions": {
      "specificity": 7,
      "context": 6,
      "clarity": 8,
      "predictability": 7
    }
  },
  "summary": "One sentence on the biggest weakness.",
  "upgrade": "...",
  "powered_by": "PQS — pqs.onchainintel.net"
}
```

Print this plain-text summary (no markdown tables, no extra commentary):

```
PQS  Grade: <score.grade>   Score: <score.total>/<score.out_of>   Percentile: <score.percentile>
     Vertical: <vertical>

Dimensions:
  specificity:    <score.dimensions.specificity>/10
  context:        <score.dimensions.context>/10
  clarity:        <score.dimensions.clarity>/10
  predictability: <score.dimensions.predictability>/10

Biggest weakness:
  <summary>
```

Rules:
- If any field is missing or `null`, print `—` in its place rather than fabricating a value.
- Stop after printing the summary. Do not add tips, follow-up questions, or meta-commentary.
