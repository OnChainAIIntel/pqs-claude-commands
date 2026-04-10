---
description: Score a prompt with PQS; if it's below 60/80, rewrite it and re-score until it clears the bar
argument-hint: <prompt text>
---

You are running the `/pqs-optimize` command. The user wants you to score the prompt in `$ARGUMENTS`, and — only if it scores below 60/80 — produce an improved rewrite, re-score the rewrite, and show the before/after.

Follow these steps in order. Do not skip any.

## 1. Load the API key from `~/.pqs/config`

Use the Read tool to read `~/.pqs/config`. It is KEY=VALUE format:

```
PQS_API_KEY=PQS_<base64url>
PQS_API_BASE=https://pqs.onchainintel.net
```

- If the file does not exist, or `PQS_API_KEY` is missing/empty, print **exactly** this message and stop:

  > No PQS key found. Run: `curl -fsSL https://pqs.onchainintel.net/install.sh | bash`

- Otherwise extract `PQS_API_KEY` and `PQS_API_BASE` (default `PQS_API_BASE` to `https://pqs.onchainintel.net` if absent).

## 2. Validate the argument

If `$ARGUMENTS` is empty or whitespace-only, print:

> Usage: `/pqs-optimize <prompt to optimize>`

and stop.

## 3. Score the original prompt

Use the Bash tool to POST to `${PQS_API_BASE}/api/score` with:

- Header: `Authorization: Bearer ${PQS_API_KEY}`
- Header: `Content-Type: application/json`
- Body: `{"prompt": "<$ARGUMENTS>"}` — JSON-escape the prompt properly.

```bash
# jq path (preferred)
PAYLOAD=$(jq -Rs '{prompt: .}' <<<"$ARGUMENTS")

# python3 fallback
PAYLOAD=$(python3 -c 'import json,sys; print(json.dumps({"prompt": sys.argv[1]}))' "$ARGUMENTS")

curl -sS -o /tmp/pqs-orig.json -w '%{http_code}' \
  -X POST "${PQS_API_BASE}/api/score" \
  -H "Authorization: Bearer ${PQS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD"
```

Important: Do **not** echo `PQS_API_KEY` to the terminal.

Handle HTTP status:
- 401 → `Invalid or expired PQS API key. Re-run install.sh to refresh it.` and stop.
- 402 → `This endpoint now requires x402 payment. Your free-tier key was not accepted.` and stop.
- 4xx → print the `error` field and stop.
- 5xx → `PQS server error — try again in a moment.` and stop.
- 200 → continue.

Parse the response body (same shape as `/pqs-score`): `score.total`, `score.out_of`, `score.grade`, `score.dimensions`, `top_fixes`.

## 4. Early exit if already strong

If `score.total >= 60`, print **exactly** this and stop:

```
Your prompt is already strong (Score: <total>/<out_of>). No optimization needed.
```

Do not rewrite, re-score, or add commentary.

## 5. Construct an improved prompt

The original prompt scored below 60. Use the original prompt text, the dimension scores, and the `top_fixes` from the API to rewrite it. The improved prompt should:

- Preserve the user's original intent.
- Address every item in `top_fixes` directly.
- Strengthen the weakest dimensions (lowest scores in `score.dimensions`) — e.g. if `role_definition` is 1/10, add an explicit role; if `output_format` is low, specify the exact output structure; if `examples` is low, add at least one concrete example; if `cot_structure` is low, ask the model to reason step by step; if `constraints` is low, add explicit do/don't constraints.
- Stay a single prompt (one block of text), not a multi-turn conversation.
- Not be padded with filler — every added clause should map to a fix or a weak dimension.

Write the improved prompt to a shell variable `IMPROVED` so you can re-score it.

## 6. Re-score the improved prompt

POST the improved prompt to the same endpoint the same way:

```bash
PAYLOAD2=$(jq -Rs '{prompt: .}' <<<"$IMPROVED")
# or python3 fallback as above

curl -sS -o /tmp/pqs-new.json -w '%{http_code}' \
  -X POST "${PQS_API_BASE}/api/score" \
  -H "Authorization: Bearer ${PQS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD2"
```

Handle the same HTTP status codes as step 3.

Parse `score.total` and `score.grade` from the new response.

## 7. Display the result

Print this exact layout:

```
─────────────────────────────────────
PQS OPTIMIZE
─────────────────────────────────────
Original score:  <orig_total>/<orig_out_of>  (Grade <orig_grade>)

Improved prompt:
<improved prompt, verbatim, no code fence, preserve line breaks>

New score:       <new_total>/<new_out_of>  (Grade <new_grade>)
Delta:           +<new_total - orig_total>
─────────────────────────────────────
```

Rules:
- If the new score is **≥ 60**, stop after the block. Do not add commentary.
- If the new score is **still < 60**, append this single line after the block and stop:

  ```
  Note: best attempt still below 60/80. Run /pqs-score on the improved prompt for a full dimension breakdown.
  ```

- Do not re-attempt a second rewrite. One optimization pass, one re-score.
- If `new_total < orig_total` (rewrite regressed), show the numbers honestly — do not fabricate improvement.
- Do not add tips, follow-up questions, or summaries beyond the block above.
