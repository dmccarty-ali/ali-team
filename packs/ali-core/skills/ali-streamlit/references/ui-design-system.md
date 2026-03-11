# ali-streamlit - UI Design System Reference

Comprehensive UI/UX patterns for professional Streamlit applications. Load when styling UI, implementing accessibility, or designing layouts.

---

## Professional CSS Design System

**From tax-ai-chat - Navy/Gold Professional Palette:**

```python
st.markdown("""
<style>
    /* Professional Design System - Tax/Finance/Healthcare */
    :root {
        --primary-navy: #003366;
        --primary-navy-light: #004080;
        --accent-gold: #C5A572;
        --accent-gold-light: rgba(197, 165, 114, 0.15);
        --success-green: #1B5E20;
        --success-green-light: #E8F5E9;
        --danger-red: #8B0000;
        --danger-red-light: #FFEBEE;
        --text-primary: #1a1a1a;
        --text-secondary: #5a5a5a;
        --border-light: #e0e0e0;
        --bg-surface: #ffffff;
        --bg-elevated: #f8f9fa;
        --shadow-sm: 0 1px 3px rgba(0,0,0,0.08);
        --shadow-md: 0 2px 8px rgba(0,0,0,0.12);
        --radius-md: 10px;
    }

    /* Primary buttons */
    .stButton > button[kind="primary"] {
        background: linear-gradient(135deg, var(--primary-navy) 0%, var(--primary-navy-light) 100%) !important;
        color: white !important;
        border: none !important;
        box-shadow: 0 2px 6px rgba(0,51,102,0.3) !important;
        border-radius: var(--radius-md) !important;
    }

    /* Context header */
    .context-header {
        background: linear-gradient(135deg, var(--primary-navy) 0%, var(--primary-navy-light) 100%);
        color: white;
        padding: 0.875rem 1.25rem;
        border-radius: var(--radius-md);
        border-left: 4px solid var(--accent-gold);
        margin-bottom: 1rem;
        box-shadow: var(--shadow-md);
    }
</style>
""", unsafe_allow_html=True)
```

**Color palette guidelines:**

| Category | Color | Hex | Use Case |
|----------|-------|-----|----------|
| Primary | Navy | `#003366` | Buttons, headers, branding |
| Accent | Gold | `#C5A572` | Highlights, borders, focus states |
| Success | Green | `#1B5E20` | Success messages, positive actions |
| Danger | Red | `#8B0000` | Errors, destructive actions |
| Text Primary | Dark Gray | `#1a1a1a` | Body text |
| Text Secondary | Medium Gray | `#5a5a5a` | Secondary text, hints |
| Border | Light Gray | `#e0e0e0` | Borders, dividers |
| Background | White | `#ffffff` | Main background |
| Elevated | Light Gray | `#f8f9fa` | Cards, panels |

**Industry-specific palettes:**

| Industry | Primary | Accent | Notes |
|----------|---------|--------|-------|
| Tax/Finance | Navy | Gold | Conservative, trustworthy |
| Healthcare | Blue | Teal | Calming, professional |
| Legal | Charcoal | Silver | Serious, authoritative |
| Education | Blue | Orange | Engaging, energetic |

---

## Accessibility (WCAG 2.1 AA Compliance)

### Focus Indicators

**Enhanced keyboard navigation:**

```css
/* Enhanced focus for keyboard navigation */
a:focus, button:focus, input:focus, select:focus, textarea:focus {
    outline: 3px solid var(--accent-gold) !important;
    outline-offset: 2px !important;
}

/* Skip link for screen readers */
.skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: var(--primary-navy);
    color: white;
    padding: 8px;
    z-index: 100;
}
.skip-link:focus {
    top: 0;
}
```

**Skip link implementation:**

```python
st.markdown("""
    <a href="#main-content" class="skip-link">Skip to main content</a>
    <div id="main-content">
        <!-- Main content here -->
    </div>
""", unsafe_allow_html=True)
```

**WCAG focus requirements:**
- Focus indicator must have 3:1 contrast with background
- Focus indicator must be visible for all interactive elements
- Outline width: minimum 2px, recommended 3px
- Outline offset: 2px recommended for clarity

---

### Reduced Motion Support

**Respect user preferences:**

```css
/* Respect user's motion preferences */
@media (prefers-reduced-motion: reduce) {
    * {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
    }
}
```

**When this matters:**
- Users with vestibular disorders
- Users prone to motion sickness
- Users with attention disorders
- Accessibility requirement in WCAG 2.1 (Level A)

---

### High Contrast Mode

**Windows High Contrast Mode support:**

```css
/* Windows High Contrast Mode support */
@media (prefers-contrast: high) {
    :root {
        --primary-navy: #000080;
        --accent-gold: #FFD700;
        --border-light: #000000;
    }
}
```

**Testing:**
- Windows: Settings > Ease of Access > High Contrast
- macOS: System Preferences > Accessibility > Display > Increase Contrast

---

### Dark Mode Support (CRITICAL PATTERN)

**The Pattern (Never Break This Again):**

1. Base styles OUTSIDE media query = Light mode
2. Overrides INSIDE `@media (prefers-color-scheme: dark)` = Dark mode
3. Always define BOTH - never style only in one mode!

**Example - File Uploader Button (from tax-ai-chat fix):**

```css
/* ❌ WRONG - Only dark mode styled, light mode broken */
@media (prefers-color-scheme: dark) {
    [data-testid="stFileUploader"] button {
        background-color: #2a2a2a !important;
        color: #ffffff !important;
    }
}
/* Light mode has invisible text - NOT DEFINED! */

/* ✅ CORRECT - Both modes explicitly defined */
/* Base styles = Light mode (OUTSIDE media query) */
[data-testid="stFileUploader"] button {
    background-color: #ffffff !important;  /* Light mode: white bg */
    color: #000000 !important;             /* Light mode: black text */
    border: 1px solid #e0e0e0 !important;
}

/* Dark mode overrides (INSIDE media query) */
@media (prefers-color-scheme: dark) {
    [data-testid="stFileUploader"] button {
        background-color: #2a2a2a !important;  /* Dark mode: dark bg */
        color: #ffffff !important;             /* Dark mode: white text */
        border: 1px solid #4a4a4a !important;
    }
}

/* WCAG 2.1 AA Contrast Requirements:
   - Light mode: Black on white = 21:1 (✅ Passes AAA)
   - Dark mode: White on dark gray = 12.6:1 (✅ Passes AAA)
   - Minimum required: 4.5:1 for normal text
*/
```

**Complete Dark Mode Design System:**

```css
/* Base palette = Light mode */
:root {
    --bg-primary: #ffffff;
    --bg-secondary: #f8f9fa;
    --text-primary: #1a1a1a;
    --text-secondary: #5a5a5a;
    --border-color: #e0e0e0;
}

/* Dark mode palette overrides */
@media (prefers-color-scheme: dark) {
    :root {
        --bg-primary: #1a1a1a;
        --bg-secondary: #2a2a2a;
        --text-primary: #ffffff;
        --text-secondary: #b0b0b0;
        --border-color: #4a4a4a;
    }
}

/* All components use CSS variables (work in both modes) */
.main-container {
    background-color: var(--bg-primary);
    color: var(--text-primary);
    border: 1px solid var(--border-color);
}
```

**Why This Matters:**
- Users toggle dark mode in browser/OS settings
- Streamlit respects `prefers-color-scheme` media query
- Without explicit light mode styles, text becomes invisible
- WCAG 2.1 AA requires 4.5:1 contrast for normal text, 3:1 for large text
- Production incident: File uploader button invisible in light mode (fixed in tax-ai-chat)

**Testing checklist:**
- [ ] Test in light mode (default browser)
- [ ] Test in dark mode (browser dark mode enabled)
- [ ] Test in high contrast mode (accessibility settings)
- [ ] Verify text contrast ratios (use browser dev tools)
- [ ] Check focus indicators in both modes

---

### ARIA Labels and Semantic HTML

**Screen reader support:**

```python
st.markdown("""
    <button aria-label="Close dialog" aria-pressed="false">
        ✕
    </button>

    <div role="alert" aria-live="polite">
        Client created successfully!
    </div>

    <nav aria-label="Main navigation">
        <ul>
            <li><a href="#dashboard">Dashboard</a></li>
            <li><a href="#clients">Clients</a></li>
        </ul>
    </nav>
""", unsafe_allow_html=True)
```

**ARIA attributes:**

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `aria-label` | Label for icon buttons | "Close dialog", "Add item" |
| `aria-describedby` | Link to description | Points to help text ID |
| `aria-live` | Announce dynamic content | "polite", "assertive" |
| `role` | Define component role | "button", "alert", "navigation" |
| `aria-pressed` | Toggle button state | "true", "false" |
| `aria-expanded` | Expandable section state | "true", "false" |

**Semantic HTML:**

```html
<!-- Bad: div soup -->
<div class="header">
    <div class="title">Tax Practice AI</div>
</div>
<div class="content">...</div>
<div class="footer">...</div>

<!-- Good: semantic elements -->
<header>
    <h1>Tax Practice AI</h1>
</header>
<main>...</main>
<footer>...</footer>
```

---

## Layout Patterns

### Split Layout (Sidebar + Main + Sticky Panel)

```python
# Page configuration
st.set_page_config(
    page_title="Tax Practice AI",
    page_icon="📋",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Sidebar for navigation
with st.sidebar:
    st.title("Navigation")
    if st.button("Dashboard"):
        st.session_state.active_panel = "dashboard"
    if st.button("Clients"):
        st.session_state.active_panel = "clients"

    st.divider()
    if st.button("Logout"):
        logout()
        st.rerun()

# Main content area with columns
col1, col2 = st.columns([2, 1])

with col1:
    # Chat area
    st.header("Chat")
    for message in st.session_state.messages:
        with st.chat_message(message["role"]):
            st.markdown(message["content"])

with col2:
    # Sticky form panel
    st.header("Client Info")
    with st.form("client_form"):
        name = st.text_input("Name")
        email = st.text_input("Email")
        submitted = st.form_submit_button("Save")
        if submitted:
            # Handle submission
            pass
```

**Layout ratios:**

| Purpose | Ratio | Use Case |
|---------|-------|----------|
| Equal split | `[1, 1]` | Two equal panels |
| Main + sidebar | `[3, 1]` | Main content + context |
| Main + form | `[2, 1]` | Chat + form panel |
| Three columns | `[1, 2, 1]` | Left nav + main + right context |

---

### Context Header Pattern

```python
def render_context_header(client_name: str, return_year: int):
    """Show current client/return context persistently."""
    st.markdown(f"""
        <div class="context-header">
            <h3>{client_name}</h3>
            <span class="context-badge">Tax Year {return_year}</span>
        </div>
    """, unsafe_allow_html=True)

# Usage
if st.session_state.current_client:
    render_context_header(
        st.session_state.current_client["name"],
        st.session_state.current_return["tax_year"]
    )
```

**Context header CSS:**

```css
.context-header {
    background: linear-gradient(135deg, var(--primary-navy) 0%, var(--primary-navy-light) 100%);
    color: white;
    padding: 0.875rem 1.25rem;
    border-radius: var(--radius-md);
    border-left: 4px solid var(--accent-gold);
    margin-bottom: 1rem;
    box-shadow: var(--shadow-md);
}

.context-badge {
    background: var(--accent-gold-light);
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    font-size: 0.875rem;
    margin-left: 0.5rem;
}
```

---

### Loading States

**Spinner with Context:**

```python
with st.spinner("Loading client data..."):
    clients = load_clients_from_db()

with st.spinner("Processing document with AI..."):
    result = st.session_state.async_bridge.run_flow(document_flow, inputs)
```

**Toast Notifications:**

```python
def show_toast(message: str, type: str = "success"):
    """Show toast notification via session state."""
    st.session_state.toast_message = {"message": message, "type": type}

# In main UI (after rerun)
if st.session_state.get("toast_message"):
    msg = st.session_state.toast_message
    if msg["type"] == "success":
        st.success(msg["message"])
    elif msg["type"] == "error":
        st.error(msg["message"])
    elif msg["type"] == "warning":
        st.warning(msg["message"])
    st.session_state.toast_message = None

# Usage
show_toast("Client created successfully!", "success")
st.rerun()
```

**Progress bars:**

```python
# Determinate progress
progress_bar = st.progress(0)
for i in range(100):
    time.sleep(0.01)
    progress_bar.progress(i + 1)

# Indeterminate (spinner)
with st.spinner("Processing..."):
    do_long_operation()
```

---

## Component Styling

### Custom Buttons

```css
/* Primary action buttons */
.stButton > button[kind="primary"] {
    background: linear-gradient(135deg, var(--primary-navy) 0%, var(--primary-navy-light) 100%) !important;
    color: white !important;
    border: none !important;
    box-shadow: 0 2px 6px rgba(0,51,102,0.3) !important;
    border-radius: var(--radius-md) !important;
    padding: 0.5rem 1.5rem !important;
    font-weight: 600 !important;
    transition: transform 0.2s, box-shadow 0.2s !important;
}

.stButton > button[kind="primary"]:hover {
    transform: translateY(-2px) !important;
    box-shadow: 0 4px 12px rgba(0,51,102,0.4) !important;
}

/* Secondary buttons */
.stButton > button[kind="secondary"] {
    background: transparent !important;
    color: var(--primary-navy) !important;
    border: 2px solid var(--primary-navy) !important;
    border-radius: var(--radius-md) !important;
}

/* Danger buttons */
.stButton > button[kind="danger"] {
    background: var(--danger-red) !important;
    color: white !important;
    border: none !important;
}
```

---

### Form Styling

```css
/* Form containers */
.stForm {
    background: var(--bg-elevated);
    padding: 1.5rem;
    border-radius: var(--radius-md);
    border: 1px solid var(--border-light);
    box-shadow: var(--shadow-sm);
}

/* Input fields */
.stTextInput input, .stTextArea textarea {
    border: 1px solid var(--border-light) !important;
    border-radius: 6px !important;
    padding: 0.5rem !important;
}

.stTextInput input:focus, .stTextArea textarea:focus {
    border-color: var(--accent-gold) !important;
    box-shadow: 0 0 0 3px var(--accent-gold-light) !important;
}
```

---

### Card Components

```css
.card {
    background: var(--bg-surface);
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    padding: 1.25rem;
    box-shadow: var(--shadow-sm);
    transition: box-shadow 0.3s;
}

.card:hover {
    box-shadow: var(--shadow-md);
}

.card-header {
    font-size: 1.125rem;
    font-weight: 600;
    margin-bottom: 0.75rem;
    color: var(--text-primary);
}

.card-body {
    color: var(--text-secondary);
    line-height: 1.6;
}
```

**Usage:**

```python
st.markdown("""
    <div class="card">
        <div class="card-header">Client Information</div>
        <div class="card-body">
            <p><strong>Name:</strong> John Doe</p>
            <p><strong>Email:</strong> john@example.com</p>
            <p><strong>Status:</strong> Active</p>
        </div>
    </div>
""", unsafe_allow_html=True)
```

---

## Responsive Design

### Mobile-First Breakpoints

```css
/* Mobile first (default styles for mobile) */
.container {
    padding: 1rem;
}

/* Tablet and up */
@media (min-width: 768px) {
    .container {
        padding: 1.5rem;
    }
}

/* Desktop and up */
@media (min-width: 1024px) {
    .container {
        padding: 2rem;
    }
}
```

**Streamlit column responsiveness:**

```python
# Desktop: 3 columns
# Mobile: stacks vertically automatically
col1, col2, col3 = st.columns(3)

# Responsive column widths
col1, col2 = st.columns([2, 1])  # 2:1 ratio on desktop, stacks on mobile
```

---

## Accessibility Checklist

Use this checklist when implementing UI:

**Visual:**
- [ ] Color contrast meets WCAG 2.1 AA (4.5:1 for normal text, 3:1 for large)
- [ ] Text resizable to 200% without loss of content
- [ ] Focus indicators visible (3px outline, 2px offset)
- [ ] No information conveyed by color alone

**Keyboard:**
- [ ] All interactive elements keyboard accessible
- [ ] Tab order logical and sequential
- [ ] Skip link provided for keyboard users
- [ ] No keyboard traps

**Screen Reader:**
- [ ] ARIA labels on icon buttons
- [ ] Semantic HTML (header, main, nav, footer)
- [ ] Dynamic content announced (aria-live)
- [ ] Form labels properly associated

**Motion:**
- [ ] Respects prefers-reduced-motion
- [ ] No auto-playing animations
- [ ] Animations can be paused

**Dark Mode:**
- [ ] Light mode styles defined (base styles)
- [ ] Dark mode overrides defined (media query)
- [ ] Contrast ratios verified in both modes
- [ ] All components styled in both modes

---

## References

- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [Streamlit Theming](https://docs.streamlit.io/library/advanced-features/theming)
- [MDN Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility)
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)
- [ali-frontend-developer skill](../ali-frontend-developer/SKILL.md) - General frontend patterns
