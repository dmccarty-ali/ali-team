---
name: ali-svg-spec
description: |
  SVG (Scalable Vector Graphics) specification and development patterns. Use when:

  PLANNING: Designing SVG-based graphics, planning icon systems, architecting
  SVG animations, considering accessibility for vector graphics

  IMPLEMENTATION: Writing SVG markup, creating paths and shapes, implementing
  animations, building responsive SVG components, embedding SVG in web apps

  GUIDANCE: Asking about SVG best practices, optimization, accessibility,
  browser compatibility, or SVG vs other image formats

  REVIEW: Checking SVG code for optimization, accessibility, security (XSS),
  or proper structure
---

# SVG Specification

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing SVG-based graphics systems
- Planning icon libraries or design systems
- Architecting SVG animations or interactions
- Considering vector vs raster for specific use cases

**Implementation:**
- Writing SVG markup (paths, shapes, text)
- Creating animations (CSS, SMIL, JavaScript)
- Building responsive SVG components
- Embedding SVG in HTML (inline, img, object)
- Generating SVG programmatically

**Guidance/Best Practices:**
- SVG optimization and file size reduction
- Accessibility for screen readers
- Browser compatibility considerations
- When to use SVG vs PNG/WebP

**Review/Validation:**
- Checking SVG code for optimization issues
- Validating accessibility (titles, descriptions)
- Reviewing for security (XSS in SVG)
- Verifying responsive behavior

---

## Key Principles

- **Scalability**: SVG scales infinitely without quality loss
- **Accessibility**: Use <title> and <desc> for screen readers
- **Optimization**: Remove unnecessary metadata, simplify paths
- **Security**: Sanitize SVG from untrusted sources (can contain scripts)
- **Responsive**: Use viewBox, not fixed width/height
- **Semantic**: Use meaningful element structure

---

## Patterns We Use

### Basic SVG Structure

Proper SVG document structure with accessibility:

```svg
<svg xmlns="http://www.w3.org/2000/svg"
     viewBox="0 0 100 100"
     role="img"
     aria-labelledby="title desc">
  <title id="title">Icon Name</title>
  <desc id="desc">Description of what this icon represents</desc>

  <!-- Graphics content -->
  <circle cx="50" cy="50" r="40" fill="#333"/>
</svg>
```

### Responsive SVG (No Fixed Dimensions)

Let SVG scale with container:

```svg
<!-- Good: Responsive -->
<svg viewBox="0 0 100 100" class="icon">
  <!-- content -->
</svg>

<!-- CSS controls size -->
<style>
.icon {
  width: 100%;
  max-width: 200px;
  height: auto;
}
</style>
```

### Common Shapes

```svg
<!-- Rectangle -->
<rect x="10" y="10" width="80" height="60" rx="5" ry="5"/>

<!-- Circle -->
<circle cx="50" cy="50" r="40"/>

<!-- Ellipse -->
<ellipse cx="50" cy="50" rx="40" ry="25"/>

<!-- Line -->
<line x1="10" y1="10" x2="90" y2="90" stroke="#000" stroke-width="2"/>

<!-- Polyline (open shape) -->
<polyline points="10,10 50,50 90,10" fill="none" stroke="#000"/>

<!-- Polygon (closed shape) -->
<polygon points="50,10 90,90 10,90" fill="#333"/>
```

### Path Commands

The <path> element is the most powerful:

```svg
<path d="
  M 10 10       <!-- Move to (start point) -->
  L 90 10       <!-- Line to -->
  H 90          <!-- Horizontal line to x -->
  V 90          <!-- Vertical line to y -->
  Q 50 50 10 90 <!-- Quadratic curve (control, end) -->
  C 20 20 80 20 90 90  <!-- Cubic curve (c1, c2, end) -->
  A 40 40 0 0 1 10 10  <!-- Arc (rx ry rotation large-arc sweep end) -->
  Z             <!-- Close path -->
"/>
```

Lowercase = relative coordinates, Uppercase = absolute.

### Grouping and Reuse

```svg
<svg viewBox="0 0 200 100">
  <!-- Define reusable elements -->
  <defs>
    <g id="icon-star">
      <polygon points="12,2 15,8 22,9 17,14 18,21 12,17 6,21 7,14 2,9 9,8"/>
    </g>

    <linearGradient id="grad1" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" style="stop-color:#ff0;stop-opacity:1"/>
      <stop offset="100%" style="stop-color:#f00;stop-opacity:1"/>
    </linearGradient>
  </defs>

  <!-- Use defined elements -->
  <use href="#icon-star" x="10" y="10"/>
  <use href="#icon-star" x="50" y="10"/>

  <rect x="100" y="10" width="80" height="40" fill="url(#grad1)"/>
</svg>
```

### CSS Styling

```svg
<svg viewBox="0 0 100 100">
  <style>
    .primary { fill: #0066cc; }
    .primary:hover { fill: #004499; }
    .stroke-only {
      fill: none;
      stroke: #333;
      stroke-width: 2;
      stroke-linecap: round;
      stroke-linejoin: round;
    }
  </style>

  <circle class="primary" cx="50" cy="50" r="40"/>
  <path class="stroke-only" d="M 20 50 L 80 50"/>
</svg>
```

### CSS Animation

```svg
<svg viewBox="0 0 100 100">
  <style>
    @keyframes spin {
      from { transform: rotate(0deg); }
      to { transform: rotate(360deg); }
    }
    .spinner {
      transform-origin: center;
      animation: spin 1s linear infinite;
    }
  </style>

  <circle class="spinner" cx="50" cy="50" r="40"
          fill="none" stroke="#333" stroke-width="4"
          stroke-dasharray="60 200"/>
</svg>
```

### Embedding Methods

```html
<!-- Inline (best for styling/scripting) -->
<svg viewBox="0 0 100 100">...</svg>

<!-- Image element (simplest, no styling) -->
<img src="icon.svg" alt="Icon description" width="100" height="100">

<!-- Object (supports scripting, fallback content) -->
<object type="image/svg+xml" data="icon.svg">
  <img src="fallback.png" alt="Fallback">
</object>

<!-- CSS background -->
<style>
.icon { background-image: url('icon.svg'); }
</style>

<!-- Data URI (inline in CSS/HTML) -->
<img src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg'...">
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Fixed width/height without viewBox | Not responsive | Use viewBox, CSS for sizing |
| Missing title/desc | Inaccessible | Add for screen readers |
| Inline styles everywhere | Hard to maintain | Use CSS classes or <style> |
| Unsanitized user SVG | XSS vulnerability | Sanitize or use <img> tag |
| Huge file from editor | Slow loading | Optimize with SVGO |
| Text without font fallback | Rendering issues | Use web fonts or convert to paths |
| Complex paths for simple shapes | Larger file size | Use basic shapes when possible |

---

## Quick Reference

### ViewBox Explained

```svg
viewBox="min-x min-y width height"
viewBox="0 0 100 100"  <!-- Origin at 0,0, coordinate space 100x100 -->
```

### Stroke Properties

```svg
stroke="#333"              <!-- Color -->
stroke-width="2"           <!-- Thickness -->
stroke-linecap="round"     <!-- butt | round | square -->
stroke-linejoin="round"    <!-- miter | round | bevel -->
stroke-dasharray="5 3"     <!-- Dash pattern -->
stroke-dashoffset="2"      <!-- Dash offset -->
stroke-opacity="0.5"       <!-- Transparency -->
```

### Fill Properties

```svg
fill="#333"                <!-- Solid color -->
fill="url(#gradient)"      <!-- Gradient reference -->
fill="none"                <!-- No fill -->
fill-opacity="0.5"         <!-- Transparency -->
fill-rule="evenodd"        <!-- evenodd | nonzero -->
```

### Transform

```svg
transform="translate(10, 20)"      <!-- Move -->
transform="rotate(45)"             <!-- Rotate (degrees) -->
transform="rotate(45, 50, 50)"     <!-- Rotate around point -->
transform="scale(2)"               <!-- Uniform scale -->
transform="scale(2, 0.5)"          <!-- Non-uniform scale -->
transform="skewX(10)"              <!-- Skew -->
transform="matrix(a,b,c,d,e,f)"    <!-- Full matrix -->
```

### Optimization Tips

1. Remove editor metadata (Illustrator, Inkscape comments)
2. Simplify paths (reduce decimal precision)
3. Use shapes instead of paths where possible
4. Combine paths with same styling
5. Remove unnecessary groups
6. Use SVGO for automated optimization

```bash
# SVGO command line
npx svgo input.svg -o output.svg
```

---

## Security

### XSS in SVG

SVG can contain executable JavaScript:

```svg
<!-- DANGEROUS - can execute scripts -->
<svg>
  <script>alert('XSS')</script>
  <a xlink:href="javascript:alert('XSS')">Click</a>
  <circle onload="alert('XSS')"/>
</svg>
```

**Mitigations:**
- Sanitize SVG from untrusted sources (DOMPurify)
- Use <img> tag for untrusted SVG (scripts don't execute)
- CSP headers to block inline scripts

```javascript
// Safe SVG loading
const clean = DOMPurify.sanitize(untrustedSvg, {
  USE_PROFILES: { svg: true }
});
```

---

## References

- [MDN SVG Tutorial](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial)
- [SVG Specification (W3C)](https://www.w3.org/TR/SVG2/)
- [SVG Path Commands](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial/Paths)
- [SVGO Optimizer](https://github.com/svg/svgo)
- [SVG Accessibility](https://www.w3.org/TR/SVG-access/)
