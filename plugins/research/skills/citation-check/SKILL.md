---
name: citation-check
description: Use whenever the user is writing, reviewing, or editing research that contains citations to academic papers, preprints, books, or technical reports. Verifies (1) the cited work actually exists, (2) the metadata (authors, title, year, venue) is accurate, and (3) every claim attributed to a cited work is genuinely supported by that work. BLOCKS inclusion of any unverified citation. Activates on phrases like "add a citation for", "write the related work section", "check my references", "is this paper real", "did this paper actually say", "review my bibliography", or any task producing or modifying a reference list.
---

# Citation Verification Skill

You are checking citations for Sidd's research. The single most damaging failure in AI-assisted research writing is fabricated or misattributed citations — citations that look plausible but don't exist, or exist but don't say what's claimed. **Do not let one slip through.**

Default posture: **BLOCK every citation until all three checks pass.** If a check fails, surface the specific failure and let the user decide how to proceed. Never bluff "looks fine to me" — that is exactly the failure mode this skill exists to prevent.

## The Three Checks

For every citation in scope (whether the user is adding new ones or auditing an existing reference list), run these three checks in order. A failure in any check means the citation does not get included until resolved.

### Check 1 — Existence

The cited work must actually exist and be retrievable.

Procedure:
1. Identify a stable identifier — DOI, arxiv ID, ISBN, or a permanent URL. If the citation has none, that's a yellow flag — search for one.
2. Resolve the identifier:
   - DOI → `WebFetch` `https://doi.org/<doi>` and confirm it lands on a real paper landing page.
   - arxiv → `WebFetch` `https://arxiv.org/abs/<id>`.
   - URL → `WebFetch` and confirm the page contains the cited work, not a 404 / soft-404 / unrelated page.
3. Cross-check against a structured index when possible: Crossref (`https://api.crossref.org/works/<doi>`), arxiv API, OpenAlex, Semantic Scholar. These return canonical metadata that's harder to spoof than scraped HTML.
4. If you cannot retrieve a stable record after a reasonable search, the citation **fails existence**. Do not guess. Tell the user "I could not confirm this paper exists" and list what you tried.

Confabulated citations are common — LLMs often invent a plausible-sounding paper. Be especially suspicious when:
- Title is very on-the-nose for the claim being supported.
- Authors include real researchers known for the topic but in unusual combinations.
- Year is recent and the venue is a well-known journal but the DOI doesn't resolve.
- The same citation appears repeatedly across an LLM-generated draft for many distinct claims.

### Check 2 — Metadata accuracy

Every field in the citation as written must match the canonical record exactly.

Required fields to verify:
- **Title** — exact, including capitalization style if the citation format requires it.
- **Authors** — every name, every initial, every diacritic, and the **order** must match. Common errors: dropped middle authors, reversed first/last name (especially for non-Western names), missing accents, "et al." used too aggressively.
- **Year** — preprint year vs published year are different things. Cite the version you're actually using.
- **Venue** — journal / conference / publisher / "arxiv preprint". Don't promote a preprint to a journal it was never published in.
- **Volume / issue / pages / DOI** — when the citation style requires them, fill them from the canonical record, not from memory.

Source priority for canonical metadata: Crossref / arxiv API / OpenAlex > publisher landing page > Google Scholar / scraped HTML. The first group returns structured records; the rest can be stale or wrong.

Author-name traps to watch for:
- Eastern name order (family-name-first) silently rendered as Western order.
- Accents stripped (`Müller` → `Muller`).
- Hyphenated surnames split into two authors.
- Initial styles mixed within a single bibliography (`J. T. Smith` vs `Smith J.T.` vs `Smith, John Thomas`).

If anything mismatches, output the diff and the canonical version. Do not silently "correct" the citation — show the user what changed.

### Check 3 — Claim grounding (most important)

For every claim, sentence, statistic, or quote attributed to a cited work, you must locate the supporting passage in the actual paper. **This is the check that catches AI-generated nonsense citations.**

Procedure for each cited claim:

1. Retrieve the full text. Order of preference:
   - Open-access PDF (arxiv, the publisher's free version, institutional repository) — use `Read` with `pages` for PDFs longer than 10 pages.
   - HTML version on publisher site (`WebFetch`).
   - If the paper is paywalled and you cannot get full text: **STOP**, tell the user, do not infer from abstract alone. The user may have access; ask them to provide the PDF.
2. Classify the claim being made:
   - **Direct quote** — verify the exact string appears in the paper (allow only whitespace/line-break/hyphenation differences).
   - **Paraphrase** — locate the passage that supports the claim and verify the paraphrase is faithful: same scope, same hedging, same direction of effect, no claims invented.
   - **Statistic / number** — locate the exact figure in the paper. If only an approximation is cited, verify the rounding direction is honest.
3. Output the supporting passage (with page number or section heading) so the user can verify the match independently. Don't just say "yes I found it" — show the passage.

Common claim-grounding failures to flag explicitly:
- **Scope inflation** — paper finds an effect in one population, citation generalizes it.
- **Strength inflation** — paper says "may be associated with"; citation says "causes".
- **Wrong attribution** — the claim is true and the citation is a real paper, but the paper doesn't make that claim. The citation got attached to the wrong source.
- **Stat invention** — a number is cited that doesn't appear in the paper at all.
- **Sign flip** — paper found a negative result; citation describes it as positive (or vice versa).

If you cannot locate the supporting passage, the claim **fails grounding**. Tell the user where you looked and what you searched for. Suggest either: replace the citation with one that actually supports the claim, or rewrite the claim to match what the paper actually says.

## Output format

For each citation processed, produce a verdict block:

```
[N] Smith, J. T., Doe, A. B., & Jones, C. (2024). "Foo bar baz." Nature, 612(7945), 123-130.

  Existence:       PASS  — DOI 10.1038/s41586-024-XXXXX resolved (Crossref)
  Metadata:        WARN  — author list missing middle initial; canonical "Doe, A. B." vs cited "Doe, A."
  Claim grounding: FAIL  — quoted "X causes Y" not in paper. Closest passage (p.4, §3.2):
                          "X is associated with Y under condition Z"
                          Effect strength is weaker and conditional. Recommend rewording or
                          finding a different citation.

  Verdict: BLOCK — fix metadata and either replace claim wording or replace citation.
```

When auditing an existing bibliography, run check 1 (existence) and check 2 (metadata) on every citation in one pass before doing the expensive check 3 (claim grounding) on each one in turn. This way the cheap failures surface fast.

Final summary, when done with a batch:
- Total citations checked: N
- PASS: A   |   WARN: B   |   FAIL: C   |   UNREACHABLE: D
- Per-citation verdicts above.

## What to do when blocked

When a citation fails any check:
1. State the failure crisply (which check, what specifically failed).
2. Offer concrete next steps:
   - For existence failures: "Drop this citation, find a real one, or confirm the identifier."
   - For metadata mismatches: show the diff and ask the user to accept the canonical version.
   - For claim-grounding failures: suggest specific alternative citations (only if you can verify them with the same three checks) or suggest rewording the claim.
3. Wait for the user's call. Do not include the citation in the output until the failure is resolved or explicitly waived.

If the user explicitly waives a check ("just include it, I'll fix it later"):
1. Acknowledge.
2. Tag the citation in the draft with a clearly visible marker like `<!-- UNVERIFIED: claim grounding failed -->` so it's not forgotten.
3. List every waived citation in the final summary.

## Citation style

Determine the citation format using this three-step cascade. Do not hardcode a default.

1. **Detect from the manuscript.** If you have access to the document being edited (or its existing bibliography), parse a few existing entries and identify the format: BibTeX `@article{...}`, APA 7 (`Author, A. A. (Year). Title. Venue, vol(iss), pp.`), Chicago author-date, MLA, IEEE numbered, ACM, Nature, Vancouver, etc. If a consistent style is detected, use it. Be conservative: if multiple styles are present, treat the manuscript as having no detected style.

2. **Ask the user once.** If detection fails or there's no manuscript context, ask the user a single question: *"What citation format should I use? (e.g. BibTeX, APA 7, Chicago, IEEE, Nature, ACM, or paste an example)"*. Cache the answer for the rest of the session.

3. **Best judgment fallback.** If the user can't answer, says "you pick", or doesn't respond, output BibTeX entries with full author lists, DOI, and arxiv ID. Rationale: BibTeX is the most lossless intermediate form and converts cleanly to any other style later.

When you fix a citation, always show both the original and the corrected form so the user can verify the format change.

## Tools to use

- `WebFetch` — DOI resolution, arxiv landing pages, publisher pages, structured-API endpoints (Crossref, OpenAlex, Semantic Scholar).
- `Read` — open-access PDFs. Always use the `pages` parameter for papers longer than 10 pages so the file actually loads.
- `WebSearch` — when you have only a partial citation and need to find the canonical record.
- `mcp__firecrawl__*` — for sites that block plain `WebFetch` (some publisher pages, some preprint servers).
- `mcp__exa__*` — semantic search across academic sources when keyword search misses.
- `Grep` (on a downloaded paper file) — for fast direct-quote verification.

Never substitute a structured-database lookup with "I remember this paper". Memory of papers is exactly the surface that hallucinates citations. Always look it up.
