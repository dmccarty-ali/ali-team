---
name: ali-validate-deck
description: |
  Full deck review process for HTML presentation decks. Use when:

  PLANNING: Designing review workflows, planning quality gates for presentations,
  considering what validation checks to run before client delivery

  IMPLEMENTATION: Running polish-deck.js validation, capturing screenshots,
  spawning expert reviews, fixing identified issues

  GUIDANCE: Best practices for presentation QA, visual review methodology,
  when to use programmatic vs visual validation

  REVIEW: Evaluating presentation quality, checking brand compliance,
  validating zone system adherence, assessing readability and visual hierarchy
---

# Full Deck Review

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing a review process for presentation decks
- Planning quality gates before client delivery
- Considering what checks to run on slides

**Implementation:**
- Running validation on a presentation deck
- Capturing screenshots for visual review
- Fixing issues found during deck review

**Guidance/Best Practices:**
- Asking how to review a deck before sending to client
- Asking what a full validation includes
- Best practices for presentation QA

**Review/Validation:**
- Reviewing rendered slide quality
- Checking brand compliance visually
- Evaluating readability and information density

---

## Key Principles

- **Programmatic first**: Always run polish-deck.js before visual review. Fix automatable issues before spending expert time.
- **Visual review supplements automated checks**: Code-reading experts miss rendering issues. Always review actual rendered screenshots.
- **Fix-and-verify loop**: After fixes, re-run validation and re-capture screenshots to confirm.
- **Parallel expert review**: UX and graphics experts review independently for different concerns.
- **Screenshots are the source of truth**: Experts review PNG images, not HTML source code.

---

## The 7-Step Process

### Step 1: Programmatic Validation
```bash
cd presentations/<deck-name>
node ../../templates/modular/polish-deck.js --dry-run
```
Validates against style-rules.yaml: zone overflow, title length, contrast, content limits.

### Step 2: Auto-Fix
```bash
node ../../templates/modular/polish-deck.js
```
Applies auto-fixes (AI title shortening, contrast in cards). Re-validates in loop.

### Step 3: Capture Screenshots
```bash
node ../../templates/modular/polish-deck.js --screenshots
```
Captures per-slide PNGs to ./screenshots/ using Playwright at 11x8.5 landscape.

### Step 4: Visual Expert Review
Spawn in parallel:
- **ali-ux-expert**: Readability, information density, visual flow, audience appropriateness, diagram comprehension
- **ali-graphics-expert**: Branding accuracy, typography, color harmony, layout balance, visual polish

Both experts read the PNG screenshot files and write findings to .tmp/.

### Step 5: Fix Issues
Spawn **ali-frontend-developer** with specific fixes from expert findings. Common fixes: contrast violations, text visibility, color corrections.

### Step 6: Re-validate
```bash
node ../../templates/modular/polish-deck.js --screenshots --dry-run
```
Confirm programmatic issues resolved. Visual spot-check of fixed slides.

### Step 7: Generate PDF
```bash
node ../../templates/modular/polish-deck.js
```
Or: `node assemble-pdf.js` for PDF-only generation.

---

## Quick Reference Checklist

- [ ] polish-deck.js --dry-run passes (0 violations)
- [ ] Screenshots captured for all slides
- [ ] UX expert review complete (findings in .tmp/)
- [ ] Graphics expert review complete (findings in .tmp/)
- [ ] Critical issues fixed
- [ ] Re-validation clean
- [ ] PDF generated

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Validation script | templates/modular/polish-deck.js |
| Style rules | templates/styles/centric/style-rules.yaml |
| Zone CSS | templates/styles/centric/zone-system.css |
| Deck index | presentations/index.html |
| Screenshot output | presentations/<deck>/screenshots/*.png |
| PDF assembly | templates/modular/assemble-pdf.js |
| Slide validation | templates/modular/validate-slides.js |
| Full procedure | RUNBOOK.md > Full Deck Review section |

All paths relative to Presentation_Master root.

---

## References

- RUNBOOK.md: Full Deck Review section for detailed procedure
- CLAUDE.md: Modular Slide Tooling command table
- style-rules.yaml: All quantitative validation rules
- centric-style-guide.md: Brand standards (sections 8-10)
