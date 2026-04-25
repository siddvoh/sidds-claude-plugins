---
name: lit-review
description: Use when the user wants a literature search across multiple sources for a research question, NOT for summarizing a single known paper (that's paper-summarizer). Performs multi-pass search across arxiv, OpenAlex, and Semantic Scholar, deduplicates results, runs every candidate through citation-check before it can be cited, and optionally produces structured summaries via paper-summarizer. Activates on phrases like "find papers on", "what's the literature on", "do a lit review for", "find recent work in", "search for papers about", or any task that produces a multi-paper bibliography.
---

# Literature Review Skill

You are running a literature search for Sidd's research. Output is a bibliography of papers actually relevant to the user's question. **Hard requirement: nothing enters the output unless it has passed `citation-check` existence + metadata.** Confabulated citations are the failure mode this skill exists to prevent.

## Hard rules

1. **No paper enters the bibliography unverified.** Every result must pass `citation-check` checks 1 (existence) and 2 (metadata) before inclusion. Check 3 (claim grounding) is run only on papers the user actually wants to cite, since it's expensive.
2. **Search across multiple sources, not one.** Single-source searches inherit that source's biases (arxiv skews CS/physics; PubMed skews bio). Always cross-reference at least two of: arxiv, OpenAlex, Semantic Scholar.
3. **Deduplicate aggressively.** The same paper has different IDs across sources (arxiv ID, DOI, OpenAlex ID, Semantic Scholar ID). Merge by DOI when present, then by fuzzy title+first-author match.
4. **Show your search.** The user must be able to see exactly which queries were run, against which sources, and how many results came back. No black-box ranking.

## Workflow

### Step 1 — Clarify the research question

Before searching, restate the user's question in the form of a search-friendly research question. Confirm with the user if it's ambiguous. Write it down — it's needed for the report at the end.

Then derive search facets:
- **Core topic** — the noun phrase being searched (e.g. "test-time scaling for LLMs").
- **Synonyms / variants** — the same idea phrased differently (e.g. "inference-time compute", "reasoning compute", "thinking budget").
- **Time scope** — "last 2 years" by default unless the user specifies. State it.
- **Method scope** — empirical / theoretical / survey, depending on what's useful.
- **Exclusions** — things explicitly off-topic.

### Step 2 — Multi-pass search

Run each pass and log the queries used, the source, and the raw count of results. Sources, in order of preference for AI/ML topics:

1. **arxiv** — `WebFetch` `https://export.arxiv.org/api/query?search_query=...&sortBy=submittedDate&max_results=50` or use the arxiv API endpoint. Best for preprints, recent CS work.
2. **OpenAlex** — `WebFetch` `https://api.openalex.org/works?search=...&filter=publication_year:2024-`. Best for citation graph, broad coverage including non-CS fields.
3. **Semantic Scholar** — `https://api.semanticscholar.org/graph/v1/paper/search?query=...`. Best for semantic ranking and "papers similar to" recommendations.

For each source, run at least two queries:
- Strict: the core topic phrase exactly.
- Broad: core topic OR'd with synonyms.

Save raw query results as JSON, one file per (source, query) pair, named with a hash of the query so re-running is idempotent. Put them in a `lit-review/<topic-slug>/raw/` directory, parented under whatever location the project uses for research data (honor an existing convention, else default to a `.research/` or `research/` directory at the project root). Tell the user the chosen path.

### Step 3 — Deduplicate and merge

Merge results across sources using this priority order for matching:

1. Same DOI → same paper.
2. Same arxiv ID (including different versions, e.g. v1 vs v3) → same paper. Keep the latest version's metadata.
3. Same title (case-insensitive, whitespace-normalized) AND same first author → same paper.
4. Otherwise: separate papers.

After merging, each unique paper has a record with all source IDs (DOI, arxiv ID, OpenAlex ID, Semantic Scholar ID) listed.

### Step 4 — Verify each candidate

For each unique paper that passes initial relevance filtering, run `citation-check` checks 1 and 2:

- Existence: confirm the paper resolves to a real landing page.
- Metadata: confirm title, authors, year, venue match the canonical record (Crossref / arxiv API / OpenAlex).

Drop any paper that fails existence. Flag (do not drop) papers with metadata mismatches — note the diff for the user to resolve.

### Step 5 — Relevance triage

For each verified paper, classify relevance to the research question:

| Tier | Meaning | Action |
|------|---------|--------|
| A | Directly addresses the question | Include in main bibliography, queue for `paper-summarizer` |
| B | Closely related, useful context | Include in main bibliography |
| C | Tangentially related | Include in "see also" section |
| D | Same keywords, different topic | Drop |

Triage is based on title + abstract only at this stage. Don't read full text yet — `paper-summarizer` does that.

### Step 6 — Optional: summarize tier-A papers

If the user wants summaries (or by default, for the top 5 tier-A papers), invoke `paper-summarizer` for each. The summarizer picks its own output location per its rules.

### Step 7 — Produce the report

Output a markdown document `REPORT.md` in the same `lit-review/<topic-slug>/` directory as the raw query results from Step 2:

```markdown
# Literature review: <research question>

**Run on:** <date>
**Time scope:** <e.g. 2023-present>
**Sources searched:** arxiv, OpenAlex, Semantic Scholar
**Queries run:** <count>
**Raw results retrieved:** <count>
**After dedup:** <count>
**Verified (passed citation-check):** <count>
**Tier A:** <count>   |   **Tier B:** <count>   |   **Tier C:** <count>

## Tier A — directly addresses the question
| # | Citation | DOI / arxiv | Why relevant | Summary path |
|---|----------|-------------|--------------|--------------|

## Tier B — context and adjacent work
| # | Citation | DOI / arxiv | Why relevant |
|---|----------|-------------|--------------|

## Tier C — see also
...

## Search queries used (for replication)
| Source | Query | Date | Result count | Raw file |
|--------|-------|------|---------------|----------|

## Verification failures and warnings
...

## Coverage caveats
- Sources NOT searched: <e.g. "did not search PubMed since topic is CS-only">
- Time-scope cutoff: <date>
- Language: English only unless specified
- Anything else that limits the review's coverage
```

Citations are formatted using the cascade in `citation-check` (detect from manuscript → ask user → BibTeX fallback).

## LLM calls during lit-review

If you use any LLM API calls for query expansion, semantic re-ranking, or relevance classification, log them per the **Raw call logging** protocol in `ai-analysis/SKILL.md`. Use a `run-id` that ties them to this lit-review run, and honor that same location-selection logic for where the per-call JSON files go.

## Tools to use

- `WebFetch` — arxiv API, OpenAlex API, Semantic Scholar API, individual paper landing pages.
- `WebSearch` — when API search misses a known relevant paper and you need to find it by partial info.
- `mcp__exa__*` — semantic search across academic content; useful when keyword search is too narrow.
- `mcp__firecrawl__*` — for sources that block plain `WebFetch`.
- `Write` — to save raw query results, the report, and the bibliography.

## Edge cases

- **Topic is too broad** (e.g. "AI safety"). Push back, ask the user to narrow. A lit review of a vague topic produces a useless mountain of papers.
- **Topic is too narrow** (returns 0-3 papers). Try broader synonyms; if still empty, tell the user the literature is thin and report what searching actually surfaced.
- **Dominant paper not in initial results.** If you (or the user) know of a key paper that should be there but didn't surface, search for it explicitly, verify it, and include it — but flag in the report that it wasn't found by the systematic queries (the queries may need broadening).
- **User wants only open-access papers.** Filter to OA in OpenAlex; arxiv is OA by definition; for Semantic Scholar use the `openAccessPdf` field.
