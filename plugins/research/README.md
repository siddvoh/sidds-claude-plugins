# research

Research-grade workflows for Sidd. Pure prompt-driven skills (no MCP servers, no hooks), so the plugin is portable and inspectable.

The skills are designed to compose into one rigorous research pipeline: gather sources → ground claims → run experiments → bundle for replication → audit before submission. Every skill blocks on a specific failure mode that's common in AI-assisted research.

## Skills

### `ai-analysis` (model-invoked)

Activates whenever a conversation involves comparing or benchmarking two or more AI models or agents (closed or open source).

Enforces:
1. **Tier parity** — flagship vs flagship, not flagship vs mini.
2. **Sampling parity** — same temperature, top_p, max_tokens, system prompt, seed.
3. **Reasoning parity** — explicit, matched thinking budget across providers.
4. **Cost transparency** — fresh-from-the-web prices shown to you BEFORE any paid call, with source URLs.
5. **Provider variability** — caching, service tier, region, snapshot pinning surfaced.
6. **Raw call logging** — every API call's full request, response, thinking trace, timing, and cost saved to `.research/raw-calls/<run-id>/` at the moment it happens. This is the source of truth for `replication-pack` and `pre-submission-check`.

BLOCKS by default. Waivers are carried into the final report so the experiment is reproducible.

### `citation-check` (model-invoked)

Activates whenever the conversation involves writing, reviewing, or auditing research that contains citations.

Per-citation, enforces:
1. **Existence** — the work resolves to a real paper via DOI / arxiv / canonical URL. Catches confabulated citations.
2. **Metadata accuracy** — title, author list, author order, year, venue, DOI match the canonical record (Crossref / arxiv API / OpenAlex preferred over scraped HTML).
3. **Claim grounding** — every claim attributed to a paper is locatable in that paper, with the supporting passage shown to the user. Catches scope inflation, strength inflation, sign flips, stat invention, and wrong attribution.

Citation format is detected from the manuscript, asked from the user if undetectable, or falls back to BibTeX.

### `paper-summarizer` (model-invoked)

Activates when the user wants a structured summary of a single, specific paper.

Produces a fixed schema: TL;DR, core claims, evidence, methodology, limitations, replication effort, what the paper does NOT address, plus quotes worth keeping. **Every numbered claim is grounded line-by-line in the source paper** (same standard as `citation-check` check 3). Summaries are saved to `.research/summaries/` for downstream use.

Refuses to summarize a paper it can't access in full — abstract-only summarization is exactly the failure mode this skill exists to prevent.

### `replication-pack` (model-invoked)

Activates when the user wants to bundle a completed AI experiment into a shareable, reproducible artifact.

Produces a deterministic `.zip` containing the experiment's setup, every raw API call (with thinking traces), cost log, and environment snapshot. Includes a `MANIFEST.json` with per-file SHA256 hashes and a stable bundle SHA256, so the bundle can be cited.

Refuses to bundle if the raw-calls log is incomplete or has leaked credentials.

### `lit-review` (model-invoked)

Activates when the user wants a literature search across multiple sources for a research question.

Multi-pass search across arxiv, OpenAlex, and Semantic Scholar. Aggressive deduplication. **Every result must pass `citation-check` checks 1 and 2 before entering the bibliography.** Tier-A results can be auto-summarized via `paper-summarizer`. The full search trail (queries, sources, raw counts) is saved so the review is reproducible.

### `pre-submission-check` (model-invoked, also offered proactively)

Activates when the user is finishing a research artifact (paper, report, blog post) and signals submission intent. Should also be **offered** by Claude when finishing-up signals appear, even unprompted.

Final integrity sweep, orchestrating the other skills:
- Manuscript hygiene (no `UNVERIFIED` / `TODO` left behind, cross-references resolve, page limits met).
- Bibliography run through `citation-check` end to end.
- Every reported AI metric backed by a logged `ai-analysis` run with a built `replication-pack`.
- Every cross-model comparison claim verified against its parity table; hidden waivers blocked.
- Reproducibility surface checked (code/data release, environment metadata).
- Venue-specific rules pulled fresh from the venue's submission page.

Default verdict is **BLOCK** until every issue is resolved or explicitly waived with a reason that goes into the manuscript's caveats section.

## How the skills compose

```
lit-review ──▶ citation-check ──▶ paper-summarizer
                                                │
ai-analysis ────────────────▶ replication-pack ─┤
                                                │
                                                ▼
                                  pre-submission-check
                                  (orchestrates everything)
```

You can invoke any skill standalone. The composition is for when you're running a full research project end-to-end.

## Why these exist

Most AI comparison "research" you see on Twitter or in blog posts is broken: someone runs Claude Opus with thinking disabled against GPT-5 with `reasoning_effort: high` and posts the results. `ai-analysis` exists so Sidd's comparisons never have that problem.

Most AI-assisted research writing contains at least one fabricated or misattributed citation. `citation-check`, `paper-summarizer`, and `lit-review` exist so Sidd's references and summaries never have that problem.

Most "AI experiments" reported in papers cannot be replicated — the prompts, params, and raw outputs are gone by the time someone wants to reproduce them. `replication-pack` exists so every Sidd experiment ships with the data needed to re-run it.

`pre-submission-check` exists so none of the above slip through on deadline.

## Default directory layout (convention, not requirement)

The skills are **format-driven, not path-driven**. They search by the schema of the files they care about (raw-call JSON, cost-log CSV, summary markdown, etc.) — they don't depend on a specific directory structure. If you already have an experiments/runs/logs layout, the skills honor it. If you don't, here's the default convention they'll fall back to:

```
.research/                          (or wherever the project keeps research data)
├── raw-calls/<run-id>/             # per-call JSON, written by ai-analysis & lit-review & paper-summarizer
│   ├── _index.json
│   └── 0001-anthropic-...json
├── runs/<run-id>/setup.json        # the experiment's setup
├── cost-log.csv                    # one row per call
├── summaries/<paper-id>.md         # written by paper-summarizer
├── lit-review/<topic-slug>/
│   ├── REPORT.md
│   └── raw/<source>-<query-hash>.json
├── replication-packs/<run-id>.zip
├── replication-packs/<run-id>.sha256
└── pre-submission-checks/<manuscript>-<date>.md
```

If a skill can't find data it needs, it raises an alarm with what it searched for — it does not fabricate data or assume a path silently.

**Auto-gitignore.** Skills automatically add their output locations to `.gitignore` before writing the first file, so prompts, responses, and other artifacts aren't committed by accident. The canonical rule lives in `skills/ai-analysis/SKILL.md` § Auto-gitignore; every other skill references it. If you ever want to deliberately commit an artifact (e.g. publish a replication pack), use `git add -f`.
