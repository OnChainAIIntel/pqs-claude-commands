---
description: Score every prompt in a file with PQS and print a results table + aggregate metrics
argument-hint: <path to prompts file>
---

You are running the `/pqs-batch` command. The user wants you to score every prompt in the file at `$ARGUMENTS` against the PQS API, display a results table, print aggregate metrics, and write a full results file to the current directory.

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

> Usage: `/pqs-batch <path/to/prompts.txt>`

and stop.

Resolve `$ARGUMENTS` to the file path. If the file does not exist, print:

> File not found: `<path>`

and stop.

## 3. Read the prompts

Use the Read tool to read the file. Treat it as one prompt per line:

- Strip trailing whitespace from each line.
- **Skip blank lines** (lines that are empty or whitespace-only).
- Do not skip comment lines — there is no comment syntax. Every non-blank line is a prompt.
- Keep the original 1-indexed line order for display.

If the file contains zero non-blank lines, print:

> No prompts found in `<path>`.

and stop.

## 4. Score each prompt sequentially

For each prompt, in order, POST to `${PQS_API_BASE}/api/score`:

- Header: `Authorization: Bearer ${PQS_API_KEY}`
- Header: `Content-Type: application/json`
- Body: `{"prompt": "<line>"}` — JSON-escape properly via `jq -Rs` or `python3 -c 'import json,sys; print(json.dumps({"prompt": sys.argv[1]}))'`.

```bash
curl -sS -o /tmp/pqs-batch-resp.json -w '%{http_code}' \
  -X POST "${PQS_API_BASE}/api/score" \
  -H "Authorization: Bearer ${PQS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD"
```

Rate-limit protection:
- **Sleep 500ms between calls** (`sleep 0.5`). The delay comes *between* requests, not before the first one.

Do **not** echo `PQS_API_KEY` to the terminal.

Per-prompt status handling:
- **HTTP 401** → print `Invalid or expired PQS API key. Re-run install.sh to refresh it.` and stop the entire batch immediately.
- **HTTP 402** → print `This endpoint now requires x402 payment. Your free-tier key was not accepted.` and stop the entire batch.
- **HTTP 5xx** → record the row as an error (`score = —`, `grade = ERR`) and continue with the next prompt. Do not abort the whole batch for a single 5xx.
- **HTTP 4xx (other)** → same — record as error row and continue.
- **HTTP 200** → parse `score.total`, `score.out_of`, `score.grade` and record the row.

Accumulate rows in memory for display + results file.

## 5. Display the results table

After all prompts have been scored, print:

```
─────────────────────────────────────
PQS BATCH: <N> prompts scored
─────────────────────────────────────
  #  Prompt                                                         Score   Grade
  -  ------------------------------------------------------------   -----   -----
  1  <truncated prompt, max 60 chars, pad right with spaces>        <t>/<o>  <g>
  2  ...
  ...
─────────────────────────────────────
```

Rules:
- Truncate prompts longer than 60 characters to 57 characters + `...`.
- Right-pad shorter prompts with spaces so the `Score` column aligns.
- For error rows, print `  —    ERR` in the Score / Grade columns.
- The `#` column is the sequential 1-based index of non-blank lines (not the file line number).

## 6. Print aggregate metrics

Immediately after the table, print:

```
─────────────────────────────────────
Aggregate metrics
─────────────────────────────────────
Total prompts scored:  <N successfully scored, excluding errors>
Average score:         <avg>/<out_of>
Grade distribution:    A:<a>  B:<b>  C:<c>  D:<d>  F:<f>
Highest scoring:       <t>/<o> — "<truncated prompt, max 60 chars>"
Lowest scoring:        <t>/<o> — "<truncated prompt, max 60 chars>"
─────────────────────────────────────
```

Rules:
- Exclude error rows from *all* aggregate calculations.
- `Average score` is rounded to one decimal place.
- If multiple prompts tie for highest/lowest, show the first one in file order.
- If `Total prompts scored` is 0 (every call failed), print `No successful scores — all requests failed.` instead of the aggregate block, and still write the results file so the user can see the errors.
- The grade distribution only includes grades that PQS actually returns (A–F). If a grade outside A–F appears, add it as its own column.

## 7. Write the full results file

Write a file named `pqs-batch-results.md` in the **current working directory** (not the prompts file's directory). Use the Write tool. The file must contain:

```markdown
# PQS Batch Results

Source file: `<original $ARGUMENTS path>`
Run at: <ISO-8601 UTC timestamp, e.g. 2026-04-10T14:30:00Z — get it via `date -u +%Y-%m-%dT%H:%M:%SZ`>
Total prompts scored: <N>
Average score: <avg>/<out_of>
Grade distribution: A:<a> B:<b> C:<c> D:<d> F:<f>

## Results

| #  | Prompt | Score | Grade |
|----|--------|-------|-------|
| 1  | <full prompt, markdown-escaped: `|` → `\|`, newlines → space> | <t>/<o> | <g> |
| 2  | ... | ... | ... |

## Highest scoring

<full prompt text> — <t>/<o> (Grade <g>)

## Lowest scoring

<full prompt text> — <t>/<o> (Grade <g>)
```

Rules for the file:
- Include **full** (non-truncated) prompt text in the table and in the highest/lowest sections.
- Escape any `|` in prompt text as `\|` so the markdown table does not break. Replace embedded newlines in a prompt with a single space.
- If there were error rows, include a `## Errors` section listing them as `- Line <n>: HTTP <code> — <truncated prompt>`.
- Overwrite any existing `pqs-batch-results.md` in the current directory.

After writing the file, print a single final line:

```
Full results written to pqs-batch-results.md
```

Then stop. Do not add tips, summaries, or follow-up questions.
