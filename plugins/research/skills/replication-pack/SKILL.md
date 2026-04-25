---
name: replication-pack
description: Use when the user wants to package a completed AI experiment (an ai-analysis run) into a self-contained, shareable, reproducible bundle. Produces a deterministic .zip containing every prompt, parameter, raw API call, response, thinking trace, cost log, and environment snapshot needed to replicate the experiment, plus a manifest with file hashes. Activates on phrases like "package this run", "make a replication bundle", "share this experiment", "generate a reproducibility pack", "zip up the latest run", or whenever the user is preparing supplementary material for a paper or blog post that includes AI experiments.
---

# Replication Pack Skill

You are bundling a completed AI experiment into a reproducibility artifact. The bundle must be **self-contained and deterministic**: anyone with the bundle should be able to reproduce the exact experiment without needing access to Sidd's machine, his memory, or any other file. The bundle hash should be stable so it can be cited.

## Hard rules

1. **Never fabricate. Never fill gaps.** If the raw-calls directory is missing or incomplete, STOP and tell the user. Do not synthesize plausible-looking calls.
2. **Bundle only what's already on disk.** Everything in the pack comes from `.research/raw-calls/<run-id>/`, `.research/cost-log.csv`, and the run's setup files. If something the user asked for isn't logged, that's a gap to surface, not to invent.
3. **Determinism matters.** Sort entries, fix timestamps in the zip metadata, and report the SHA256 of the final bundle. Two pack runs over the same source data must produce byte-identical bundles.
4. **Strip credentials, not content.** The raw-call logs already redact `Authorization` headers per the `ai-analysis` logging spec. Verify this before zipping. Leave everything else (prompts, responses, thinking traces) intact.

## Inputs

The user provides one of:
- An explicit `run-id`.
- "The latest run" → use the most recently modified candidate raw-calls directory you find (see below).
- A path to a custom run directory.

If the run is ambiguous, list the candidate run-ids with timestamps and ask the user to pick.

## Workflow

### Step 1 — Find the data (search by format, not by path)

You are not entitled to assume any particular directory layout. Different users keep their experiments in different places. **Search for the data by its file format.** Only if you cannot find it after a thorough search do you raise an alarm.

What you're looking for, by format:

1. **Raw-call JSON files** — JSON files matching the schema defined in `ai-analysis` SKILL.md § Raw call logging. Required keys: `run_id`, `call_index`, `timestamp_utc`, `provider`, `model_id`, `request`, `response`, `timing`, `cost_usd`. Glob for `*.json` and check the schema, don't filter on filename.
2. **Optional `_index.json`** — file in the same directory listing all calls for a run. If you find this, great. If not, build it on the fly from the per-call files.
3. **Cost log** — a CSV file with header `timestamp,run_id,provider,model,input_tokens,output_tokens,input_cost_usd,output_cost_usd,total_cost_usd,price_source_url`. Glob for `*.csv` and check headers.
4. **Setup / experiment definition** — JSON file containing prompts, sampling params, and model IDs for the run. Field names vary; look for keys like `prompts`, `messages`, `temperature`, `model_id`, `system`.

Where to look, in order:

1. **The path the user provided**, if any.
2. **Common conventions, depth-first from the working directory:** `.research/`, `experiments/`, `runs/`, `logs/`, `data/`, `output/`, `traces/`. Don't traverse `node_modules`, `.git`, `dist`, `build`, or `__pycache__`.
3. **Anywhere a project manifest hints at.** Check `.research/config.json`, `experiments.yaml`, `pyproject.toml` (`[tool.research]` section), `package.json` (`"research"` key) for a configured location.
4. **Use `Bash` `find`** with reasonable depth (`-maxdepth 6`) and the format-checks above to scan the working tree.

For each candidate found, confirm the file actually parses and matches the schema before treating it as a hit.

### Step 1a — When the search is ambiguous

- **Multiple candidate runs found:** list them (path, run-id, timestamp, call count, est. cost) and ask the user to pick.
- **One run found in a non-default location:** show the user where you found it and confirm before bundling. ("Found 24 raw-call JSON files matching the schema in `experiments/2026-04-25/calls/`. Treat this as the run? (y/n)")
- **Nothing found:** raise an alarm clearly. State what you searched for (the schema), where you searched (the directories), and what was missing. Suggest the user point you at the directory explicitly. Do NOT proceed.

### Step 1b — Validate inputs once located

For each candidate raw-call file:

### Step 2 — Verify integrity of raw calls

For every located raw-call file:

- Confirm it parses as valid JSON.
- Confirm the required fields exist (`run_id`, `call_index`, `timestamp_utc`, `provider`, `model_id`, `request.body`, `response.body`, `timing`, `cost_usd`).
- Confirm `Authorization` and `api-key` headers are redacted. If a credential leaked, STOP and tell the user — do not proceed until cleaned up.
- Confirm `model_id` is a pinned snapshot, not an alias like `claude-opus-latest` or `gpt-5`. If alias, flag it; the experiment is less reproducible than it should be.

Report any integrity issues before bundling. The user can choose to proceed with caveats noted in the manifest, or fix and rerun.

### Step 3 — Capture environment metadata

Write `environment.json` with whatever you can determine:

```json
{
  "captured_utc": "ISO-8601",
  "platform": "darwin / linux / ...",
  "os_version": "...",
  "python_version": "...",
  "node_version": "...",
  "git_repo": "<remote URL if any>",
  "git_sha": "<HEAD SHA>",
  "git_dirty": true | false,
  "relevant_packages": {
    "anthropic": "x.y.z",
    "openai": "x.y.z",
    "google-generativeai": "x.y.z"
  }
}
```

Use `Bash` to read these from the user's actual environment. If a value can't be determined, write `null` and note it — do not guess.

### Step 4 — Assemble the bundle

Layout inside the zip:

```
<run-id>/
├── MANIFEST.json
├── README.md                    # how to read and replicate this pack
├── setup/
│   └── setup.json               # the experiment's prompts, params, model IDs
├── raw_calls/
│   ├── _index.json
│   ├── 0001-anthropic-claude-opus-4-7-...json
│   ├── 0002-openai-gpt-5-...json
│   └── ...
├── cost/
│   └── cost-log.csv             # only rows for this run
├── environment/
│   └── environment.json
└── reports/
    └── final-report.md          # the ai-analysis Setup/Results/Cost/Caveats output, if present
```

### Step 5 — Write the manifest

`MANIFEST.json` is the source of truth for the bundle:

```json
{
  "schema_version": "1.0",
  "run_id": "...",
  "run_started_utc": "earliest call timestamp",
  "run_ended_utc": "latest call timestamp",
  "packed_utc": "ISO-8601 of when the bundle was made",
  "bundle_sha256": "SHA256 of the zip itself, computed AFTER zipping (filled in by step 6)",
  "files": [
    {"path": "setup/setup.json", "size": 1234, "sha256": "..."},
    {"path": "raw_calls/_index.json", "size": 5678, "sha256": "..."},
    {"path": "raw_calls/0001-anthropic-...json", "size": 9012, "sha256": "..."}
  ],
  "summary": {
    "providers": ["anthropic", "openai", "google"],
    "models": ["claude-opus-4-7-20XXMMDD", "gpt-5-20XXMMDD", "gemini-2.5-pro-20XXMMDD"],
    "total_calls": 24,
    "total_input_tokens": 12345,
    "total_output_tokens": 6789,
    "total_cost_usd": 1.23,
    "price_sources": ["https://...anthropic-pricing", "https://...openai-pricing", "https://...gemini-pricing"]
  },
  "integrity_warnings": [],
  "user_waivers": ["e.g. 'reasoning effort waived between Claude and Gemini per user'"]
}
```

### Step 6 — Build deterministic zip

When zipping:
- Sort entries alphabetically by path.
- Set every entry's mtime to the run's `packed_utc`.
- Use stored compression (no compression) OR fixed deflate level — pick one and stick to it.
- Compute SHA256 of the resulting zip.
- Open the zip, write the SHA into `MANIFEST.json` (in a separate `bundle_sha256.txt` adjacent to the zip rather than rebuilding — embedding the hash inside changes the hash).

Output two files: `<run-id>.zip` and `<run-id>.sha256`. Put them next to where the raw-calls came from (e.g. a sibling `replication-packs/` directory), or wherever the user specifies. Tell the user the chosen output path.

### Step 7 — Write the README inside the bundle

Brief markdown explaining: (1) what the experiment was, (2) how to read each directory, (3) how to replicate (re-run the prompts in `setup/setup.json` against the model IDs listed, with the params shown). Include a one-line warning that the model snapshots may be deprecated by the provider in the future, and that the raw-call logs in `raw_calls/` are the source of truth in that case.

## Output to the user

Show the user:
1. Path to the bundle: `.research/replication-packs/<run-id>.zip`.
2. Bundle SHA256.
3. Bundle size.
4. Total cost replicated in the bundle.
5. Any integrity warnings or user waivers carried into the manifest.
6. A copy-pasteable line for citing the bundle in a paper: e.g. `Replication pack: <run-id>, SHA256:<...>`.

## Edge cases

- **Run is incomplete (some calls failed mid-run).** Bundle anyway; mark in the manifest which calls succeeded vs failed. Failed calls are also reproducibility data.
- **Run mixes paid and local-model calls.** Local calls won't have `cost_usd > 0`, that's fine. Note "local inference" in the manifest summary.
- **Sensitive prompts in raw calls.** If the user flagged the run as containing sensitive content during `ai-analysis`, ask before bundling: redact-with-hashes or refuse to bundle.
