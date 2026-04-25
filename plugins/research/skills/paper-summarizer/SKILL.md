---
name: paper-summarizer
description: Use whenever the user wants a structured summary of a specific academic paper (arxiv preprint, DOI-resolved publication, technical report, or local PDF). Produces a fixed-schema summary covering claim, evidence, methodology, limitations, and replication effort, with EVERY extracted point grounded line-by-line in the source paper. Activates on phrases like "summarize this paper", "what does <paper> say about", "TLDR this arxiv link", "give me the key claims from", or any URL/DOI/arxiv ID handed in with intent to digest. NOT for literature search across multiple papers — that's lit-review.
---

# Paper Summarizer Skill

You are producing a structured summary of a single academic paper for Sidd's research. The summary must be **strictly grounded** in the source — every claim, number, and quote must trace to a specific passage. A summary that drifts from the paper is worse than no summary, because it gets cited downstream as if it were the paper itself.

## Hard rules

1. **Never summarize a paper you cannot access in full.** Abstract-only is not enough. If the paper is paywalled and you cannot retrieve full text, STOP and tell the user. Ask them to provide the PDF.
2. **Every extracted point must cite a specific location** in the paper (page number, section heading, figure/table number, equation number).
3. **Distinguish the paper's claims from your interpretation.** The schema below has separate columns for "what the paper says" and "implication" precisely so this is never blurred.
4. **No synthesis from outside the paper.** Don't fold in what other papers say, don't lean on training data, don't infer beyond what's written. If the paper doesn't address something the user asked about, say so.

## Workflow

### Step 1 — Verify the paper exists and matches what was requested

Before reading, run the existence + metadata checks from the `citation-check` skill on the paper. If the citation can't be resolved, stop. If the metadata in the user's request doesn't match the canonical record (wrong year, wrong authors), surface the discrepancy and ask which paper they actually meant.

### Step 2 — Retrieve full text

Order of preference:
- arxiv PDF (most papers, easy to read).
- Publisher's open-access HTML / PDF.
- Author's personal page or institutional repository.
- User-provided file path.

Use `Read` with the `pages` parameter for any PDF longer than 10 pages — a single Read call without `pages` will fail on long papers. Read in chunks (e.g. pages 1-10, 11-20, ...) and track section boundaries as you go.

### Step 3 — Build the summary

Produce a markdown document with this exact schema. Do not add or remove sections.

```markdown
# <Title> — <First-author et al.>, <Year>

**Citation:** <full citation, in the manuscript's detected style or BibTeX>
**Source URL:** <arxiv abs URL / DOI URL>
**Version read:** <e.g. arxiv v3, dated YYYY-MM-DD>
**Read on:** <today's date>

## TL;DR
One paragraph (3-5 sentences) of what the paper claims and why it matters. Every sentence here must be a faithful condensation of content stated in the paper, not your interpretation.

## Core claims
| # | Claim (paraphrased) | Supporting passage | Page / Section |
|---|---------------------|--------------------|-----------------|
| 1 | ... | "<direct quote from paper>" | p.4, §3.2 |
| 2 | ... | "<direct quote>" | p.7, Fig. 4 caption |

## Evidence
For each numbered claim above, describe the evidence the paper provides (experiment, dataset, proof, theoretical argument). Cite the table/figure/section.

| Claim # | Evidence type | What the paper actually shows | Location |
|---------|---------------|-------------------------------|----------|
| 1 | Experiment on <dataset> | <metrics with numbers from the paper> | Table 2, p.6 |

## Methodology
- **Approach:** ...
- **Data:** ...
- **Model / system:** ...
- **Evaluation:** ...
- **Compute / scale:** ...
Cite section numbers for each line. If a methodology detail is absent, write "Not reported" — do not fill in plausible defaults.

## Limitations
List limitations the paper itself acknowledges (with location). Then, separately, list limitations YOU notice that the paper does not acknowledge — clearly marked as `(reviewer observation)`.

## Replication effort
- Code release: <yes / no / partial — link if any>
- Data release: <yes / no / partial>
- Compute required: <as reported, or "not reported">
- Closed dependencies: <e.g. proprietary API, paid dataset>
- Estimated person-time to replicate: <rough hours/days, with reasoning>

## What the paper does NOT address
List adjacent questions the user might assume are answered but aren't. This is the section that prevents downstream misattribution.

## Quotes worth keeping verbatim
3-5 direct quotes that capture key claims, with page numbers. Use these in downstream writing instead of paraphrasing.
```

### Step 4 — Self-check before output

Before showing the summary to the user, do this:

1. For every numbered core claim, re-locate the supporting passage in the paper text. If you can't find it, drop the claim or annotate it as `(reviewer interpretation, not stated)`.
2. For every number / statistic, verify the exact figure appears in the paper.
3. For every author name and date in the citation, confirm against the canonical record (Crossref / arxiv API / OpenAlex).

This is the same standard `citation-check` applies. The summarizer is doing citation-grade work on its own output.

## Edge cases

- **Multiple versions of the paper exist.** State explicitly which version you read (e.g. "arxiv v2, 2024-03-15"). Claims may differ across versions; pin one.
- **Paper is a preprint that was later published.** Cite both, and read the published version if accessible — it's often updated.
- **Paper has supplementary material / appendix.** Treat the appendix as part of the paper. Many key methodology details live there.
- **Paper is a position / opinion piece, not empirical.** Adapt: "Evidence" becomes "Argument structure", "Replication effort" becomes "How to engage with the argument".
- **Paper is in a language other than English.** Translate quotes literally and label them `(translated)`. Include the original language quote in a footnote.

## LLM calls during summarization

If you make any LLM API calls during this skill (e.g. asking another model to extract structured fields), log them per the **Raw call logging** protocol in the `ai-analysis` SKILL.md. Same JSON schema. Use the same logging directory as any concurrent ai-analysis run, or pick one per the location-selection rules in that section.

## Tools to use

- `WebFetch` — DOI / arxiv landing pages, structured-API metadata (Crossref, arxiv API, OpenAlex, Semantic Scholar).
- `Read` (with `pages` for long PDFs) — for full paper text.
- `mcp__firecrawl__*` — when publisher pages block plain `WebFetch`.
- `Grep` — to verify direct quotes appear in the paper file.
- `Write` — to save the summary to disk so it's reusable downstream.

Save every summary to disk by default. Pick a sensible location (honor any existing `summaries/` or similar directory in the project; otherwise default to a `summaries/` directory next to wherever the user keeps research notes). Tell the user the chosen path. Downstream skills (`lit-review`, `pre-submission-check`) search for summaries by file format (markdown with the schema above), not by path.
