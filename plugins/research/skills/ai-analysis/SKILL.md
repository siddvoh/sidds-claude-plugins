---
name: ai-analysis
description: Use whenever the user is doing research, benchmarks, or comparisons that involve TWO OR MORE AI models, agents, or LLMs (closed or open source). Enforces experimental parity across model tier, sampling configuration (temperature, max_tokens, reasoning effort, system prompt, seed), and cost transparency. BLOCKS execution until parity is verified or the user has explicitly approved an asymmetry. Activates on phrases like "compare GPT-4 vs Claude", "benchmark these models", "evaluate Llama against Gemini", "which model is better at X", or any research workflow that produces side-by-side AI outputs.
---

# AI Analysis Parity Skill

You are running a research workflow that compares two or more AI models or agents. Your job here is to act like a careful experimentalist: **never produce a comparison until parity is verified or explicitly waived by the user**. Unfair comparisons published as "research" are a real problem and Sidd's research must not contribute to it.

## The Five Parity Checks

Before any model is called, work through these five checks in order. If any check fails, STOP and surface the gap to the user. Do not proceed on assumptions.

### 1. Tier parity (model class match)

Models must be at the same capability tier (flagship vs flagship, mid vs mid, small vs small). Use `WebSearch` and the providers' official model docs to confirm tier classifications, since the tier landscape shifts frequently.

Common 2026-era tier rough mappings (verify online before relying on these):
- Flagship: Claude Opus 4.x, GPT-5 / o-series flagship, Gemini 2.5 Ultra, Llama 3.x 405B
- Workhorse: Claude Sonnet 4.x, GPT-5 mid, Gemini 2.5 Pro, Llama 3.x 70B
- Fast/cheap: Claude Haiku 4.x, GPT-5 nano, Gemini Flash, Llama 3.x 8B

**BLOCK** if the user is comparing Claude Opus to GPT nano. Tell them, suggest the matched pair, get explicit approval to proceed unmatched.

### 2. Sampling parameter parity

All inference-time parameters must match across models unless the user has explicitly opted out for a documented reason. The mandatory matched set:

| Parameter | Notes on cross-provider equivalence |
|-----------|-------------------------------------|
| `temperature` | Must be identical. Default values differ across providers, do not leave them unset. |
| `top_p` | Should match. If a provider doesn't expose it, document the asymmetry. |
| `max_tokens` (output cap) | Must match in *token count*. Note the tokenizer caveat below. |
| `system prompt` | Byte-identical across all models. |
| `user prompt` | Byte-identical across all models. |
| `seed` | Match if the provider supports it. If only some support seeds, run multiple samples and report variance. |
| `reasoning effort / thinking budget` | See section 3 below — this is the easiest place to introduce unfairness. |

**Tokenizer caveat:** `max_tokens=1000` produces different *amounts of content* across models because tokenizers differ. For a fair comparison, either (a) match by output character/word budget rather than tokens, or (b) document that comparisons are at "equal token budget" with the asymmetry noted. Ask the user which they want.

If the user has not specified values, do NOT pick defaults silently. Either ask the user or pull the documented defaults from each provider's API reference (via `WebSearch` or `mcp__plugin_context7_context7__query-docs`) and surface them in a parity table for approval.

### 3. Reasoning / thinking parity

This is the most-failed parity check in published AI comparisons. Each major provider exposes "thinking" differently:

- **Anthropic Claude**: `thinking: {type: "enabled", budget_tokens: <N>}`. Disabled by default on most models.
- **OpenAI o-series / GPT-5**: `reasoning_effort: "minimal" | "low" | "medium" | "high"`. Default varies by model.
- **Google Gemini 2.5+**: `thinkingConfig.thinkingBudget: <N>` or `"auto"`. Default is `auto`.
- **Open-source reasoning models (DeepSeek-R1, etc.)**: typically always reasoning, sometimes with `<think>` tag controls.

Before running the experiment:
1. Use `WebSearch` to confirm the **current default reasoning behavior** for each model in the comparison.
2. Show the user a table like:
   ```
   Model              | Reasoning default              | Proposed setting
   Claude Opus 4.7    | thinking disabled              | thinking budget 4096
   GPT-5              | reasoning_effort = "medium"    | reasoning_effort = "medium" (~equivalent)
   Gemini 2.5 Pro     | thinkingConfig = "auto"        | thinkingBudget = 4096
   ```
3. Get explicit user sign-off on the proposed settings. The cross-provider mapping is approximate — say so.
4. If the user wants "no thinking", verify it's actually OFF on every model. Some models (e.g. DeepSeek-R1) cannot disable reasoning; flag this as an unavoidable asymmetry.

### 4. Cost transparency (mandatory pre-flight)

Costs change. Always pull the latest pricing fresh via `WebSearch` against the official pricing page for each provider — never rely on prices from training data, conversation memory, or older runs.

**Before running anything**, present a cost preview:

```
Cost preview (rates fetched 2026-MM-DD from <official URLs>)

Model              | Input $/Mtok | Output $/Mtok | Est. input tok | Est. output tok | Est. cost
-------------------|--------------|---------------|----------------|-----------------|----------
Claude Opus 4.7    | $X.XX        | $Y.YY         | A              | B               | $Z.ZZ
GPT-5              | $X.XX        | $Y.YY         | A              | B               | $Z.ZZ
Gemini 2.5 Pro     | $X.XX        | $Y.YY         | A              | B               | $Z.ZZ
-------------------|--------------|---------------|----------------|-----------------|----------
                                                                    Total estimate:   $TOTAL
```

Show the *source URL* for each price you pulled so the user can verify. If you cannot confirm a current price, stop and tell the user — do not estimate from memory.

Get an explicit "yes proceed" before any paid API call. If the experiment is large (>$5 estimated, or >50 calls), additionally suggest a dry-run on 1-2 prompts first.

### 5. Provider-side variability

Document anything that cannot be matched and would affect results:
- **Caching:** Anthropic, OpenAI, and Google all have prompt caching that can affect both latency and cost. Disable it for benchmark fairness, or enable it consistently.
- **Service tier:** OpenAI `service_tier`, Anthropic priority tier, Google routing. Match these.
- **Region / endpoint:** different regions may have different model versions.
- **Snapshot vs alias:** prefer pinned snapshot model IDs (e.g. `claude-opus-4-7-20XXMMDD`) over moving aliases (`claude-opus-latest`) so the experiment is reproducible.

## Raw call logging (mandatory for every API call)

Every call to any LLM made during this skill — closed-source (Anthropic, OpenAI, Google) or open-source (via Together, Groq, Fireworks, vLLM, local) — MUST be saved to disk in full fidelity at the moment it happens. The API response cannot be recovered after the fact, so logging is non-negotiable.

This protocol is also referenced by `lit-review` and `paper-summarizer` for any LLM calls they make. If you're operating under any of those skills, follow this section.

### Where to save

Pick a location once per run and stick with it. Order of preference:

1. **Honor any existing convention in the project.** Before writing the first call, look around the working directory and parent directories for an existing logs/experiments/runs/raw-calls directory. If one exists with files matching the schema below, append to it.
2. **Use the user's stated location** if they specify one.
3. **Default fallback:** `.research/raw-calls/<run-id>/` at the project root.

Tell the user the chosen location at the start of the run. The consuming skills (`replication-pack`, `pre-submission-check`) search by file format — they don't depend on a specific path, so any sensible directory works.

### What to save

For every call, write a JSON file with a unique name (e.g. `<NNNN>-<provider>-<model>.json` where `NNNN` is a zero-padded sequence number per run). The file must contain:

```json
{
  "run_id": "string — matches the ai-analysis run this call belongs to",
  "call_index": 0,
  "timestamp_utc": "ISO-8601 with milliseconds",
  "provider": "anthropic | openai | google | together | groq | fireworks | local | ...",
  "model_id": "exact pinned snapshot, e.g. claude-opus-4-7-20XXMMDD (NOT an alias)",
  "endpoint_url": "the full URL the request was POSTed to",
  "request": {
    "headers_redacted": "headers with Authorization / api-key REDACTED",
    "body": "the FULL request body, byte-for-byte, including system prompt, messages, tools, sampling params, thinking config, seed"
  },
  "response": {
    "http_status": 200,
    "headers": "full response headers including any provider request-id / system_fingerprint",
    "body": "the FULL response body, byte-for-byte, including content, stop_reason, usage, system_fingerprint, AND the thinking/reasoning trace if present"
  },
  "timing": {
    "request_sent_utc": "ISO-8601",
    "first_byte_utc": "ISO-8601 (for streamed calls)",
    "response_complete_utc": "ISO-8601",
    "wall_ms": 1234
  },
  "cost_usd": {
    "input": 0.0,
    "output": 0.0,
    "cache_read": 0.0,
    "cache_write": 0.0,
    "total": 0.0,
    "price_source_url": "URL the rates were fetched from for this run"
  },
  "notes": "optional free-text — e.g. 'retry after 429', 'streaming consolidated'"
}
```

### Hard rules

1. **Save before parsing.** Write the raw response body to disk before extracting fields for analysis, so a parsing bug doesn't lose the data.
2. **Never strip the thinking/reasoning trace.** Anthropic's `thinking` blocks, OpenAI's `reasoning` summaries, and Gemini's `thoughtSignature` fields all go in the log verbatim. The trace is often the most replication-relevant part.
3. **Redact only credentials.** Authorization headers, API keys, and OAuth tokens get replaced with `"REDACTED"`. Everything else stays.
4. **Pin the model snapshot.** Always log the exact dated model ID returned by the provider, not the alias the user passed. `claude-opus-latest` is meaningless six months from now.
5. **Streamed calls** must be reassembled into the equivalent non-streamed body before logging, with `streamed: true` set in the response object so the mode is recoverable.
6. **One file per call.** Don't append to a single file — atomic per-call writes survive crashes.

### Index file

After each run, also write `_index.json` (in the same directory as the per-call files) listing every call file, in order, with: call_index, provider, model_id, total tokens, total cost, wall_ms. Consuming skills read this first when they find it; if it's missing, they regenerate it from the per-call files.

### Privacy

The raw calls contain user prompts. If the prompts include anything sensitive (PII, proprietary text, unpublished research), tell the user before logging and let them choose: (a) log anyway and `.gitignore` the chosen directory, (b) log with a content-hash placeholder instead of the actual prompt body. Whichever directory you picked above, make sure it's gitignored in the user's research repo before any sensitive call is written.

## Cost logging (after each run)

After the experiment runs, log actual costs to a file the user can audit. The schema:

```
timestamp,run_id,provider,model,input_tokens,output_tokens,input_cost_usd,output_cost_usd,total_cost_usd,price_source_url
```

Where to put it: same logic as raw-call logging. Honor any existing cost log in the project; otherwise default to `cost-log.csv` next to the raw-calls directory. Append one row per call.

Compute actual cost from the API response's usage block, not from estimates. Sum per-run totals at the bottom and report them back to the user.

## What to do when the user pushes back

If the user says "just run it, don't worry about parity":
1. Acknowledge.
2. Surface specifically what parity is being waived ("OK — proceeding with default reasoning settings on each model, which are NOT equivalent across providers").
3. Note the waiver in the final report so it's reproducible.

The goal is informed consent, not blocking the user.

## Final report format

Output a markdown report with these four sections, in this order:

1. **Setup** — table of every parameter set on every model, with a column flagging which were matched and which were waived. Include the run-id and the path to `.research/raw-calls/<run-id>/` for replication.
2. **Results** — side-by-side outputs per prompt. For multi-sample runs, include variance / agreement metrics. Quote outputs directly; do not paraphrase.
3. **Cost** — table of estimate vs actual per model, plus the grand total. Cite the price-source URLs from the cost preview.
4. **Caveats** — every parity asymmetry that survived into the run, with rationale and impact assessment. If any check was waived by the user, name it explicitly.

If the user wants a different format for a specific run (LaTeX table, JSON dump, Notion-friendly), they can ask. Default is markdown.

## Quick reference: tools to use

- `WebSearch` — for current pricing pages, model tier announcements, default parameter values.
- `WebFetch` — to read a specific provider docs page when you have the URL.
- `mcp__plugin_context7_context7__query-docs` — for SDK-level config details (parameter names, defaults).
- `Bash` (curl / jq) — only after parity is verified, for running the actual comparisons.
- `Write` / `Edit` — for the cost log and the final report.
