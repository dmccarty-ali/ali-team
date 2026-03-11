---
name: ali-presentation-library
description: |
  Centric Consulting Presentation Library - modular HTML slide decks. Use when:

  PLANNING: Designing new presentations, planning slide structure,
  architecting presentation systems, considering visual layouts

  IMPLEMENTATION: Creating new decks, adding slides, modifying presentations,
  building slide templates, implementing zone layouts

  GUIDANCE: Asking about presentation structure, finding decks,
  zone system usage, or presentation best practices

  REVIEW: Validating slides, checking zone compliance, reviewing
  presentation structure and visual consistency
---

# Presentation Library

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing new presentation decks
- Planning slide structure or visual layouts
- Architecting presentation systems
- Considering zone-based layouts

**Implementation:**
- Creating new decks from templates
- Adding or modifying slides
- Building slide content
- Implementing zone system layouts

**Guidance/Best Practices:**
- Finding available presentations
- Listing deck inventory
- Understanding zone system CSS
- Working with presentation structure
- Manifest-driven architecture

**Review/Validation:**
- Visual validation of slides
- Zone system compliance checking
- Presentation structure review
- Slide content review

---

## Key Principles

- **Modular architecture**: Each deck is self-contained with manifest.json
- **Zone-based layout**: CSS zone system controls visual structure (see centric-style-guide.md sections 8-10)
- **Manifest-driven**: manifest.json defines slides, metadata, rendering behavior
- **HTML-based**: Pure HTML/CSS slides (no framework dependencies)
- **Browser player**: player.html renders any deck via query param
- **Index file**: presentations/index.html is auto-generated catalog of all decks

---

## Quick Reference

### List All Available Decks

**The index file is the primary discovery mechanism:**

```bash
# View the index file (auto-generated listing of all decks)
open /Users/donmccarty/Library/CloudStorage/OneDrive-CentricConsulting/Active\ Centric\ Work/Presentation_Master/presentations/index.html
```

The index shows:
- Deck titles
- Slide counts
- Directory names
- Links to player with deck parameter

### View a Specific Deck

```bash
# Start local server (required for player)
cd /Users/donmccarty/Library/CloudStorage/OneDrive-CentricConsulting/Active\ Centric\ Work/Presentation_Master/presentations
python3 -m http.server 8000

# Open in browser
open http://localhost:8000/player.html?deck=ml-ai-capabilities
open http://localhost:8000/player.html?deck=mlmic-azure-accelerate-poe
```

### Regenerate Index

```bash
cd /Users/donmccarty/Library/CloudStorage/OneDrive-CentricConsulting/Active\ Centric\ Work/Presentation_Master
node tools/update-deck-index.js
```

Updates presentations/index.html with current deck inventory.

### Validate Slides

```bash
cd /Users/donmccarty/Library/CloudStorage/OneDrive-CentricConsulting/Active\ Centric\ Work/Presentation_Master/templates/modular
node validate-slides.js
```

Checks zone system compliance and reports violations.

### Generate PDF

```bash
cd /Users/donmccarty/Library/CloudStorage/OneDrive-CentricConsulting/Active\ Centric\ Work/Presentation_Master/templates/modular
node assemble-pdf.js
```

Creates PDF from slide deck using Puppeteer.

---

## Project Structure

```
Presentation_Master/
  presentations/
    index.html                          # THE INDEX FILE (auto-generated)
    player.html                         # Browser-based slide player
    ml-ai-capabilities/                 # ML/AI Capabilities deck (16 slides)
      manifest.json
      slides/*.html
    mlmic-azure-accelerate-poe/         # MLMIC Claims Reporting deck (22 slides)
      manifest.json
      slides/*.html

  templates/
    modular/                            # Template for new decks
      manifest.json
      player.html
      lib/manifest.js
      validate-slides.js               # Zone violation checker
      assemble-pdf.js                  # PDF generation
      test/                            # Test fixtures

    styles/
      centric/
        zone-system.css                # Zone layout system
        debug-zones.css                # Debug overlay

  centric-style-guide.md              # Style guide (sections 8-10 cover zones)

  tools/
    update-deck-index.js              # Regenerates presentations/index.html
```

---

## Zone System Overview

The zone system is a CSS-based layout framework for consistent slide structure.

**Key concepts:**
- Zones define content regions (header, body, footer, sidebar, etc.)
- Zones are configured via CSS classes
- Debug overlay helps visualize zone boundaries
- Violations detected by validate-slides.js

**Reference:** See centric-style-guide.md sections 8-10 for zone system details.

---

## Manifest Structure

Each deck has a manifest.json file defining:

```json
{
  "title": "Deck Title",
  "author": "Author Name",
  "date": "2024-01-15",
  "slides": [
    "slides/01-intro.html",
    "slides/02-overview.html"
  ],
  "theme": "centric",
  "aspectRatio": "16:9"
}
```

The player reads manifest.json to load and render slides in order.

---

## Common Tasks

### Finding a Presentation

1. Open presentations/index.html (the auto-generated catalog)
2. Browse by title or directory name
3. Click link to launch in player

### Creating a New Deck

1. Copy templates/modular/ to presentations/new-deck-name/
2. Update manifest.json with deck metadata
3. Create slides in slides/ directory
4. Run node tools/update-deck-index.js to add to index
5. View with player.html?deck=new-deck-name

### Viewing a Deck Locally

1. Start local server: python3 -m http.server 8000
2. Navigate to: http://localhost:8000/player.html?deck=<deck-name>
3. Use arrow keys or click to navigate slides

### Checking Zone Compliance

1. cd templates/modular/
2. node validate-slides.js
3. Review reported zone violations
4. Fix violations by adjusting CSS classes or zone definitions

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Index file (deck catalog) | presentations/index.html |
| Slide player | presentations/player.html |
| ML/AI deck | presentations/ml-ai-capabilities/ |
| MLMIC deck | presentations/mlmic-azure-accelerate-poe/ |
| Deck template | templates/modular/ |
| Zone system CSS | templates/styles/centric/zone-system.css |
| Style guide | centric-style-guide.md |
| Index regenerator | tools/update-deck-index.js |
| Slide validator | templates/modular/validate-slides.js |
| PDF generator | templates/modular/assemble-pdf.js |

**Base path:** /Users/donmccarty/Library/CloudStorage/OneDrive-CentricConsulting/Active Centric Work/Presentation_Master/

---

## References

- Style Guide (sections 8-10): centric-style-guide.md
- Modular Template: templates/modular/
- Index Generator: tools/update-deck-index.js
