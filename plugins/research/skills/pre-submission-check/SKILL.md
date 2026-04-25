---
name: pre-submission-check
description: Use when Sidd is about to submit, share, or publish a research artifact (paper, report, blog post, thesis chapter, slides) and wants a final integrity sweep. Orchestrates citation-check, replication-pack, and a structured manuscript audit, blocking on any unresolved issue. Activates on phrases like "ready to submit", "final version", "before I submit", "pre-submission check", "is this ready", "submission deadline", or any signal that the user is finishing a research artifact. Should also be PROACTIVELY OFFERED whenever the user appears to be wrapping up a research project — ask "want to run a pre-submission check?" before they proceed.
---

# Pre-Submission Check Skill

You are doing the final pass on a research artifact before Sidd submits or publishes it. This skill is the integrity gate. **Default verdict is BLOCK until every check passes or is explicitly waived with a documented reason.**

This skill orchestrates the others. It does not duplicate their work — it invokes them and aggregates their verdicts.

## Inputs

- Path to the manuscript (PDF, .tex, .docx, .md, or directory).
- Optional: target venue (so venue-specific rules can apply — page limit, anonymous review, supplementary material rules).
- Optional: list of `run-id`s for AI experiments referenced in the paper.

If the user invokes this skill without a manuscript, ask for the path. Don't proceed on memory of "the paper we were working on".

## The checklist

Walk through these checks in order. For each, produce a verdict: **PASS / WARN / FAIL / SKIPPED (with reason)**. Don't continue past a FAIL until the user has resolved it or explicitly waived it.

### 1. Manuscript integrity

- **Cross-references resolve.** Every `Section X`, `Figure Y`, `Table Z`, `Equation N` has a target. Use `Grep` on the manuscript source.
- **No `UNVERIFIED` or `TODO` markers remain.** Search for `UNVERIFIED`, `TODO`, `FIXME`, `XXX`, `<!-- ... -->` HTML comments, `\todo{}`, and similar. Any hits → FAIL with a list of locations.
- **No placeholder text.** "Lorem ipsum", "<insert citation>", "[REF]", "TBD" → FAIL.
- **Author list complete.** Names, affiliations, ORCIDs (if required), corresponding author marked.
- **Conflict of interest, funding, ethics statements present** if the venue requires them.
- **Word / page count within limits** for the target venue.

### 2. Citations (delegate to `citation-check`)

Run `citation-check` over the entire bibliography. Specifically:

- **Check 1 (existence):** every entry in the references section resolves to a real paper.
- **Check 2 (metadata):** every entry's title, authors, year, venue, DOI match canonical records.
- **Check 3 (claim grounding):** every in-text citation is run through grounding — locate the supporting passage in the cited paper for every cited claim.

This is the most expensive check. Tell the user upfront: "claim grounding requires reading every cited paper for every claim — this will take a while and may make API calls. Confirm to proceed."

Aggregate the results. The verdict for this section is FAIL if any citation fails any check. WARN for metadata-only mismatches the user might want to accept.

### 3. AI experiments (delegate to `replication-pack`)

For every AI-related claim, number, or output reported in the manuscript:

- **Is there an `ai-analysis` run that produced this number?** If the manuscript reports "Claude Opus achieved 87.3% on benchmark X", there must be a logged run that shows that result. Use the format-based search from `replication-pack` Step 1 to locate raw-call data — don't assume any directory layout. If no matching run can be found anywhere in the project, FAIL with "claim is not backed by a logged experiment". If raw-call data exists but the user keeps it somewhere unconventional, surface that and confirm with the user before treating it as the source.
- **Does the located run have a complete raw-calls log?** Use the validation step from `replication-pack` Step 2.
- **Has a replication pack been built for the run?** If not, build one now via `replication-pack`. Citing experiments without a reproducibility bundle is a form of unverifiability.
- **Numbers in the manuscript match the raw-call logs.** Cross-check every reported metric against the located `_index.json` (or generated equivalent) and the located cost log. Discrepancies → FAIL.
- **Model snapshots are pinned.** No alias model IDs (`claude-opus-latest`, `gpt-5`) cited as the model used; only dated snapshots.

### 4. Parity claims (delegate to `ai-analysis`)

If the manuscript makes any cross-model comparison claim, verify against the run's parity table:

- **Were the configs matched?** Pull the Setup section from the run's final report. Confirm temperature, max_tokens, system prompt, seed, reasoning effort were all matched OR explicitly waived.
- **Is every waiver disclosed in the manuscript?** If a parity check was waived during `ai-analysis`, that waiver MUST appear in the paper's Limitations / Methods / Caveats section. Hidden waivers → FAIL.
- **Tier match holds.** Models compared are at the same capability tier per current provider docs (re-verify, since tiers shift).

### 5. Reproducibility surface

- **Code release.** If the paper claims a code release, the link works and the repo contains what the paper says it contains. If not yet released, the manuscript says so explicitly.
- **Data release.** Same standard.
- **Replication packs are linked from the manuscript.** For every cited experiment, the manuscript should reference the replication pack ID (or include it as supplementary material).
- **Environment metadata is captured.** `environment.json` from each replication pack is consistent with what the methodology section claims (same library versions, same OS family if relevant).

### 6. Venue-specific rules (if target venue specified)

Use `WebSearch` to fetch the current call-for-papers / submission-guidelines page for the venue. Check:

- **Page / word limit.**
- **Anonymity** — paper has no author-identifying info if the venue is double-blind. Check title page, acknowledgements, self-citations using "we" instead of "the authors".
- **Supplementary material rules** — what's allowed, what's required.
- **Format.** LaTeX template version, font, margin, line-numbering, etc.
- **Ethics / data use statements** required by the venue.

If no venue is specified, SKIP this section with a note.

### 7. Final hygiene

- **Spelling and grammar pass.** Quick automated check; not a substitute for proofreading.
- **Figures readable at print resolution.** If figures are reported in low-DPI raster, flag.
- **Tables fit page width** in the rendered version.
- **Bibliography style consistent.** If the bibliography mixes styles, FAIL.
- **Acknowledgements include funding sources** if any work was funded.
- **AI use disclosure** (if venue requires it). Be specific: which models, for what (writing assistance, code generation, translation, ideation).

## Output

Produce a single markdown report `<manuscript-name>-pre-submission-<date>.md` next to the manuscript itself, or in whatever output directory the project uses. Tell the user the chosen path.

```markdown
# Pre-submission check: <manuscript title>

**Manuscript path:** <path>
**Target venue:** <venue or "not specified">
**Run on:** <date>
**Overall verdict:** PASS / BLOCK

## Section verdicts
| Section | Verdict | Issues |
|---------|---------|--------|
| 1. Manuscript integrity | PASS | — |
| 2. Citations             | BLOCK | 3 grounding failures, 1 missing DOI |
| 3. AI experiments        | WARN  | 1 metric not backed by logged run |
| 4. Parity claims         | PASS | — |
| 5. Reproducibility       | PASS | — |
| 6. Venue rules           | SKIPPED | no venue specified |
| 7. Hygiene               | PASS | — |

## Detailed findings
<list every issue with file/line, what it is, what to do>

## Required actions before submission
<numbered checklist of every BLOCK and WARN, with concrete fixes>

## Waivers carried into submission
<any waiver the user accepted, with the reason — these go in the paper's caveats section>
```

## Behavior on FAIL

When the overall verdict is BLOCK, do not let the user proceed without acknowledgement:

1. Show the report.
2. Ask: "Resolve these and re-run, or accept specific items as documented limitations?"
3. If the user wants to resolve: leave them to it; offer to re-run when they're done.
4. If the user accepts limitations: capture each accepted item with a reason. The reason goes into the manuscript's caveats / limitations section, not just into a log.

## Tools to use

- All five other research-plugin skills, invoked sequentially or as needed.
- `Read` / `Grep` — to scan the manuscript.
- `WebSearch` / `WebFetch` — for venue rules.
- `Write` — for the report.

## When to offer this proactively

Even when not explicitly invoked, OFFER this skill when:

- The user mentions "submission", "deadline", "final draft", "ready to send".
- The user asks you to check spelling, formatting, or "is this good".
- The user has just finished a long writing session on a research artifact.

The offer is a single line: *"Want me to run a pre-submission check before you proceed?"* Don't run it without consent — it's expensive and reads many files.
