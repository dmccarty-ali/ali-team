---
name: ali-doc-review
description: |
  Document review suite knowledge base. Contains all pass definitions,
  checklists, severity rules, and output format for structured document reviews.
  Use when:

  PLANNING: Designing document review workflows, planning which review passes
  to run, preparing documents for submission review

  IMPLEMENTATION: Running a structured document review, executing review passes,
  checking document consistency before submission

  GUIDANCE: Asking about what to check in document reviews, severity levels,
  how to structure findings, review methodology

  REVIEW: Reviewing RFP responses, proposals, reports, technical documents,
  checking for structural errors, contradictions, broken references, style
  issues, and quality signals
---

# Document Review Suite

## When to Use

This skill is the knowledge base for the ali-doc-review agent suite. It contains all pass definitions, checklists, severity rules, and output format standards.

**Use this suite for:**
- Running a structured document review before submission
- Checking document consistency (RFP responses, proposals, reports, technical documents)
- Reviewing for structural errors, contradictions, broken references, style issues, and quality signals

**Document format requirement:**
This suite operates on **plain text / markdown**. If the document is DOCX or PDF, convert it first:

```bash
pandoc "document.docx" -o "document.md"
# or for PDFs:
pandoc "document.pdf" -o "document.md" --extract-media=.tmp/
```

Do not attempt to review binary files.

---

## Pass Definitions

The suite runs 5 passes in parallel. Each pass is self-contained. The orchestrator (ali-doc-review-expert) spawns 5 instances of ali-doc-review-pass, one per pass name below.

---

### Pass 1: Structural

**Output file:** structural.md

**What it checks:**

**1. Heading Numbering Sequence**
- Verify H1 → H2 → H3 progression has no gaps or skips
- Flag: jumping from H2 to H4 without H3, or H1 appearing mid-document as a sub-section
- Flag: duplicate heading numbers at the same level (e.g., two "3.2" sections)
- Flag: numbered headings that restart unexpectedly

**2. TOC Accuracy**
- Extract every entry in the Table of Contents
- For each entry, verify the corresponding heading exists in the body with the exact same title and number
- Flag: TOC entry with no matching body heading
- Flag: body heading with no TOC entry (for numbered/major sections)
- Flag: title text mismatch (e.g., "3.1 Project Scope" in TOC vs "3.1 Scope of Work" in body)
- If no TOC exists: note "No TOC present" and skip this check

**3. Section Completeness**
- Identify any section referenced elsewhere in the document but that does not exist
- Check for placeholder text ("TBD", "To be completed", "Insert here") indicating incomplete sections
- Flag: sections that contain only a heading with no body content

**4. Appendix Ordering**
- Identify all appendices (Appendix A, B, C or Appendix 1, 2, 3)
- Verify they are referenced in the body in the same order they appear
- Verify appendix labels are sequential with no gaps (A, B, C — not A, C, D)
- Flag: appendix referenced in body before it appears in document

**5. Orphaned Sections**
- Identify any major section never referenced by any other part of the document
- Flag only sections that appear structurally isolated and whose purpose is unclear without a reference

**Severity:**

| Severity | Condition |
|----------|-----------|
| CRITICAL | TOC entry with no matching body heading, or missing section that is referenced |
| HIGH | Heading sequence gap or skip, duplicate section numbers |
| MEDIUM | TOC title text mismatch, appendix ordering inconsistency |
| LOW | Orphaned section (minor), placeholder text in non-critical section |

**Output format:**

```markdown
# Structural Pass — {document filename}

**Date:** {YYYY-MM-DD}
**Document:** {absolute path}
**Pass:** Structural Integrity

## Summary

{2-3 sentence summary. State the count of CRITICAL/HIGH/MEDIUM/LOW issues.}

## CRITICAL Issues

### S-C1: {Issue Title}
**Type:** {TOC Mismatch | Missing Section | ...}
**Location:** {heading or section reference}
**Detail:** {specific description — what was expected vs what was found}

{Repeat for each CRITICAL issue. If none: "None found."}

## HIGH Issues

### S-H1: {Issue Title}
**Type:** {Sequence Gap | Duplicate Number | ...}
**Location:** {heading reference}
**Detail:** {description}

{If none: "None found."}

## MEDIUM Issues

### S-M1: {Issue Title}
**Type:** ...
**Location:** ...
**Detail:** ...

{If none: "None found."}

## LOW Issues

### S-L1: {Issue Title}
**Type:** ...
**Location:** ...
**Detail:** ...

{If none: "None found."}

## Pass Verdict

{CLEAN | ISSUES FOUND}

## Files Reviewed

- {absolute path to document}
```

**Important:**
- Read the full document before making any claims — do not skim
- Cite exact heading text and numbers — never paraphrase
- If no TOC exists, say so — do not fabricate a TOC check
- Every finding needs a location — section title, heading number, or line reference

---

### Pass 2: Contradiction

**Output file:** contradiction.md

**Framing: Minimal Unsatisfiable Subsets (MUS)**

A MUS is the smallest set of statements that cannot all be true at once. Find claim A and claim B such that both cannot simultaneously be true. Report both claims, their exact locations, and why they conflict. Identify which is likely authoritative.

Do not flag every numerical mention — only flag claims about the **same subject** that **disagree**.

**What it checks:**

**1. Numbers, Prices, Dollar Figures**
- Extract every dollar amount, percentage, and numerical figure
- Group by subject (e.g., "total project cost", "number of users", "response time SLA")
- Flag any subject where the same value is stated with different numbers in two or more places
- Examples: "Total cost: $2.4M" in Section 3 vs "Budget: $2.1M" in Appendix B

**2. Dates, Durations, Week Counts, Timelines**
- Extract all stated dates, durations, and timeline claims
- Verify internal consistency: if "Phase 1 starts Week 1 and runs 6 weeks" then Phase 2 should not start "Week 5"
- Flag: any stated end date that conflicts with the stated start date plus duration
- Flag: total project duration that doesn't match the sum of phase durations

**3. Named Counts**
- Extract counts tied to named entities: "5 populations", "8 requirements", "3 workstreams", "14 team members"
- Verify: if a list or table enumerates items, the stated count must match the actual number of items
- Flag: "The following 6 deliverables" when only 5 are listed

**4. Role Titles and Named Entities**
- Note how each person's title is stated at first mention
- Flag: same person referred to with different titles in different sections
- Flag: organization names that change spelling or capitalization inconsistently (only when it could cause genuine confusion)

**5. Authoritative Source Identification**
For each contradiction found, state:
- **Claim A:** Exact quote + location
- **Claim B:** Exact quote + location
- **Likely authoritative:** Which claim to trust and why

**Severity:**

| Severity | Condition |
|----------|-----------|
| CRITICAL | Financial figures or contractual timeline contradictions |
| HIGH | Count/population mismatches, phase timeline inconsistencies |
| MEDIUM | Role title inconsistency, named entity spelling drift |
| LOW | Minor numerical rounding differences unlikely to cause confusion |

If a contradiction involves money or a binding commitment, default to CRITICAL.

**Output format:**

```markdown
# Contradiction Pass — {document filename}

**Date:** {YYYY-MM-DD}
**Document:** {absolute path}
**Pass:** Internal Contradiction Detection (MUS framing)

## Summary

{2-3 sentence summary. State the count of CRITICAL/HIGH/MEDIUM/LOW issues found. Note subject areas where contradictions were found.}

## CRITICAL Issues

### C-C1: {Subject of Contradiction}
**Type:** {Financial | Timeline | Contractual}
**Claim A:** "{exact quote}" — {location}
**Claim B:** "{exact quote}" — {location}
**Likely authoritative:** {which claim to trust and why}
**Impact:** {why this matters}

{Repeat for each CRITICAL issue. If none: "None found."}

## HIGH Issues

### C-H1: {Subject of Contradiction}
**Type:** {Count Mismatch | Timeline Inconsistency | ...}
**Claim A:** "{exact quote}" — {location}
**Claim B:** "{exact quote}" — {location}
**Likely authoritative:** {reasoning}

{If none: "None found."}

## MEDIUM Issues

### C-M1: {Subject}
**Type:** {Role Title | Named Entity | ...}
**Claim A:** "{exact quote}" — {location}
**Claim B:** "{exact quote}" — {location}
**Likely authoritative:** {reasoning}

{If none: "None found."}

## LOW Issues

### C-L1: {Subject}
**Type:** ...
**Detail:** ...

{If none: "None found."}

## Pass Verdict

{CLEAN | ISSUES FOUND}

## Files Reviewed

- {absolute path to document}
```

**Important:**
- Always quote both claims verbatim — paraphrasing can introduce its own errors
- Provide exact locations — section title, heading, or paragraph reference
- Do not flag intentional variation — "approximately $2M" and "$2.0M" in a footnote are not a contradiction
- Do not flag evolution — if the document explicitly says "this supersedes the figure in Section X," that is not a contradiction
- MUS discipline — report the minimum set of statements that conflict; do not pile on related claims

---

### Pass 3: Cross-References

**Output file:** xrefs.md

**What it checks:**

**1. Section References ("See Section X")**
- Extract every instance of "see Section X", "refer to Section X", "as described in Section X", "per Section X", and similar patterns
- For each reference, verify a heading with that number or title exists in the document and the title matches if cited
- Flag: reference to a section number that does not exist
- Flag: reference to a section title that does not match any heading

**2. Appendix Callouts**
- Extract every mention of "Appendix A", "Appendix B", "see Appendix", etc.
- For each callout, verify an appendix with that label exists and the label matches exactly
- Flag: callout to an appendix that does not exist
- Flag: label mismatch (appendix called differently in body vs in appendix section)

**3. Figure and Table References**
- Extract every "Figure 1", "Figure 2", "Table 3", "see Table X", etc.
- For each callout, verify a figure or table with that number exists and caption matches if cited
- Flag: reference to a figure or table number that does not exist
- Flag: figure/table numbering that restarts or skips unexpectedly

**4. Footnote References**
- If footnotes are present, extract all footnote markers in the body
- Verify each marker has a corresponding footnote definition, and vice versa
- Flag: orphaned footnote marker (no definition) or orphaned definition (no marker)
- If no footnotes: note "No footnotes present" and skip

**5. Back-References**
- Extract phrases like "as described above", "as noted in Section X", "per the requirements outlined in Section Y"
- Verify the referenced content exists at the stated location
- Flag: "as described above in Section 4" when the content is actually in Section 6 (i.e., below, not above)
- Flag: back-reference to content that was never stated

**Severity:**

| Severity | Condition |
|----------|-----------|
| HIGH | Broken section or appendix reference (content the reader is directed to does not exist) |
| MEDIUM | Figure or table callout mismatch, footnote orphan |
| LOW | Minor label inconsistency (e.g., "Att. A" vs "Appendix A"), "above/below" direction error |

Nothing in this pass reaches CRITICAL. If a broken cross-reference indicates a missing required section, that is a CRITICAL finding — report it in structural.md (coordinate via the merge pass), not here. Flag it here as HIGH with a note: "May also be a structural gap."

**Output format:**

```markdown
# Cross-Reference Pass — {document filename}

**Date:** {YYYY-MM-DD}
**Document:** {absolute path}
**Pass:** Cross-Reference Resolution

## Summary

{2-3 sentence summary. State count of HIGH/MEDIUM/LOW issues. Note which reference types had the most problems.}

## HIGH Issues

### X-H1: {Broken Reference Description}
**Type:** {Section Reference | Appendix Callout | ...}
**Reference found:** "{exact text of the callout}" — {location}
**Expected target:** {what was expected to exist}
**Actual:** {what was found instead, or "Not found"}

{Repeat for each HIGH issue. If none: "None found."}

## MEDIUM Issues

### X-M1: {Issue Title}
**Type:** {Figure/Table Mismatch | Footnote Orphan | ...}
**Reference found:** "{exact text}" — {location}
**Detail:** {description of the mismatch}

{If none: "None found."}

## LOW Issues

### X-L1: {Issue Title}
**Type:** {Label Inconsistency | Direction Error | ...}
**Reference found:** "{exact text}" — {location}
**Detail:** {description}

{If none: "None found."}

## Pass Verdict

{CLEAN | ISSUES FOUND}

## Files Reviewed

- {absolute path to document}
```

**Important:**
- Quote the exact callout text — do not paraphrase cross-references
- Verify by scanning the full document — do not assume a section exists because it sounds like it should
- Distinguish broken from mislabeled — a broken reference is worse than a label inconsistency
- Do not flag external URLs — this pass covers internal document references only

---

### Pass 4: Style

**Output file:** style.md

**Severity ceiling: MEDIUM — nothing in this pass reaches HIGH or CRITICAL.**

Flag systemic issues (affecting multiple sections or a pattern throughout the document) as MEDIUM. Flag isolated instances as LOW.

**What it checks:**

**1. Bullet Grammar Consistency**
Within any given list:
- All bullets should follow the same grammatical pattern: either all complete sentences (with terminal punctuation) or all fragments (without terminal punctuation)
- Mixed patterns within one list are a finding
- Do NOT require consistency across different lists — each list can choose its own pattern, but must apply it consistently within itself
- Flag: "Delivers scalable results. | Supports multiple users | Integrates with existing systems." (mixed: two sentences, one fragment)

**2. Dash Style**
- Em dash (—): parenthetical breaks, abrupt changes, emphasis
- En dash (–): ranges (2020–2025), scores, compound adjectives
- Hyphen (-): compound words, word breaks
- Flag: hyphens used where em dashes are intended ("The project - which spans 3 phases - will deliver...")
- Flag: inconsistent em dash style (some with spaces, some without, within the same document)
- Do NOT flag a single unconventional choice — flag a pattern

**3. Capitalization Consistency**
- Role titles: if "Project Manager" is capitalized in one place, it must be capitalized everywhere
- Product/system names: consistent capitalization throughout
- Section headers: consistent title case vs sentence case throughout
- Do NOT flag standard sentence capitalization — only flag inconsistency

**4. Passive Voice Density**
- Do not flag every passive construction
- Flag only sections where passive voice is so dominant it obscures agency
- Threshold: a paragraph where 4 or more consecutive sentences are passive voice
- Provide one example sentence and suggest the active rewrite

**5. Hedging Language**
- Flag multi-word hedge stacks: "may possibly", "could potentially", "might perhaps", "should hopefully"
- Do not flag single hedges ("may", "could", "might") — these are often appropriate
- For each flag, provide a tighter alternative phrasing

**6. Sentence Fragment vs Full Sentence Consistency in Lists**
Same as bullet grammar, applied specifically to numbered lists and definition-style lists.

**Output format:**

```markdown
# Style Pass — {document filename}

**Date:** {YYYY-MM-DD}
**Document:** {absolute path}
**Pass:** Style and Formatting Consistency
**Severity ceiling:** MEDIUM

## Summary

{2-3 sentence summary. State count of MEDIUM/LOW issues. Identify the most prevalent style pattern problem.}

## MEDIUM Issues (Systemic)

### ST-M1: {Issue Type}
**Type:** {Bullet Grammar | Dash Style | Capitalization | Passive Voice | Hedging | Fragment Consistency}
**Pattern:** {description of the systemic issue}
**Locations:** {list affected sections or provide representative examples}
**Example:** "{quoted example of the violation}"
**Suggestion:** {tighter alternative or corrected version}

{Repeat for each systemic MEDIUM issue. If none: "None found."}

## LOW Issues (Isolated)

### ST-L1: {Issue Type}
**Type:** ...
**Location:** {section or heading}
**Example:** "{quoted example}"
**Suggestion:** {correction}

{If none: "None found."}

## Pass Verdict

{CLEAN | ISSUES FOUND}

## Files Reviewed

- {absolute path to document}
```

**Important:**
- Systemic vs isolated is the key distinction — a pattern throughout the document is MEDIUM; one slip is LOW
- Quote examples exactly — paraphrasing style examples defeats the purpose
- Do not enforce a preference — only enforce internal consistency
- Do not overlap with quality pass — clarity and readability are quality concerns; this pass covers measurable formatting rules
- Passive voice threshold is 4 consecutive passives — do not flag a single passive sentence

---

### Pass 5: Quality

**Output file:** quality.md

**Severity ceiling: HIGH — nothing in this pass reaches CRITICAL.**

Label all findings with the prefix "Quality" to distinguish them from structural findings. Most findings in this pass will be MEDIUM or LOW.

**Every finding MUST include a suggested rewrite. Advisory without a fix is not actionable.**

**What it checks:**

**1. Clarity — Sentences Requiring More Than One Read**
- Flag any sentence that requires a second read to understand the meaning
- Typically caused by: overly long sentences (30+ words with multiple clauses), ambiguous pronoun references, nested parentheticals, or structure that buries the subject
- For each flag: quote the sentence, explain what makes it unclear, provide a rewritten version

**2. Readability — Wall-of-Text Paragraphs**
- Flag any paragraph containing 5 or more sentences with no visual break
- Also flag paragraphs that lack a clear topic sentence
- For wall-of-text: recommend breaking into 2–3 shorter paragraphs or converting to a list
- For missing topic sentence: suggest a topic sentence

**3. Professionalism — Informal Register**
- Flag phrases too casual for a professional client-facing document:
  - Contractions ("we'll", "it's", "don't") in formal sections
  - Colloquialisms ("super easy", "a lot of", "tons of", "basically")
  - First-person informality ("I think", "we feel like") in recommendation sections
  - Client name used incorrectly (wrong abbreviation, wrong formal name)
- Suggest a professional alternative for each flagged phrase

**4. Filler Phrases**
Flag these specific patterns and suggest tighter phrasing:
- "It is important to note that" → omit entirely or restructure
- "It should be noted that" → same
- "As previously mentioned" → state the point directly or omit
- "In order to" → replace with "to"
- "Due to the fact that" → replace with "because"
- "At this point in time" → replace with "now" or "currently"
- "Going forward" → replace with a specific time reference or omit
- "Leverage" (overused) → replace with "use" unless the financial/mechanical sense is intended

**5. Voice Consistency**
- Identify whether the document uses first person, second person, or third person
- Flag any section that shifts voice without a clear reason
- Note: executive summaries often use a different voice than body sections — flag only unexplained drift

**Output format:**

```markdown
# Quality Pass — {document filename}

**Date:** {YYYY-MM-DD}
**Document:** {absolute path}
**Pass:** Quality Signals (Clarity, Readability, Professionalism)
**Severity ceiling:** HIGH
**Note:** All findings labeled "Quality" to distinguish from structural findings.

## Summary

{2-3 sentence summary. State count of HIGH/MEDIUM/LOW issues. Note which quality dimension had the most findings.}

## HIGH Issues

### Q-H1: Quality — {Issue Type}
**Dimension:** {Clarity | Readability | Professionalism | Voice}
**Location:** {section heading or paragraph reference}
**Problem:** {description}
**Original:** "{exact quote}"
**Suggested rewrite:** "{improved version}"

{Repeat for each HIGH issue. If none: "None found."}

## MEDIUM Issues

### Q-M1: Quality — {Issue Type}
**Dimension:** ...
**Location:** ...
**Problem:** ...
**Original:** "{exact quote}"
**Suggested rewrite:** "{improved version}"

{If none: "None found."}

## LOW Issues

### Q-L1: Quality — {Issue Type}
**Dimension:** {Filler Phrase | Minor Informality | ...}
**Location:** ...
**Original:** "{exact quote}"
**Suggested rewrite:** "{improved version}"

{If none: "None found."}

## Pass Verdict

{CLEAN | ISSUES FOUND}

## Files Reviewed

- {absolute path to document}
```

**Important:**
- Every finding must include a suggested rewrite — advisory without a fix is not actionable and will be rejected
- Quote the original text exactly — do not paraphrase the problem
- Be selective — flag patterns and significant instances, not every possible improvement
- Label all findings "Quality" — the merge agent uses this prefix to distinguish quality findings from structural ones
- Severity ceiling is HIGH — if you find yourself wanting to use CRITICAL, the issue is structural or contradiction, not quality

---

## Severity Reference

| Level | Meaning | Examples |
|-------|---------|---------|
| CRITICAL | Blocks submission | TOC mismatch, financial figure contradiction, missing section |
| HIGH | Fix before submission | Broken cross-reference, timeline contradiction, count mismatch |
| MEDIUM | Fix if time permits | Heading sequence error, terminology inconsistency, systemic style issue |
| LOW | Polish | Minor label inconsistency, isolated style instance |

---

## Report Structure

Two output files are produced per review run:
- `findings-report.md` — full detail, all passes, all findings
- `review-summary.md` — CoS-ready summary, 400 tokens max, verdict + top issues

---

## Verdict Logic

- Any CRITICAL finding → **REQUIRES REWORK**
- Any HIGH finding, no CRITICAL → **REVISE BEFORE SUBMIT**
- Only MEDIUM/LOW or clean → **READY TO SUBMIT**

---

**Document Version:** 1.0
**Created:** 2026-03-04
**Maintained By:** ALI AI Team
