# framework-html-meta-color-scheme.md

Comprehensive reference for HTML `<meta name="color-scheme">`, the page color scheme support declaration. Covers the distinction from `<meta name="theme-color">` (color-scheme declares which schemes the page SUPPORTS; theme-color declares a specific color for browser chrome), the valid values (`light`, `dark`, `light dark`, `dark light`, `only light`, `only dark`, `normal`), why the order matters (`light dark` vs `dark light` differs in browser canvas color and ambiguity resolution), the CSS equivalent (`color-scheme: light dark;` on `:root` or `html`), the browser default behavior without the declaration (legacy light forced rendering), the form control adaptation (date pickers, scrollbars, native form elements adopt scheme appropriate styling), the canvas color (the area outside the HTML element), the relationship with `prefers-color-scheme` media query (the media query is what your CSS reacts to; color-scheme is what your page declares), the flash of incorrect color (FOIC) prevention pattern, and the Bubbles per client decision framework. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the tenth framework in the HTML signal track**, following meta robots, charset, viewport, description, keywords, author, generator, copyright, and theme-color. Companion to the 12 wire layer frameworks. Closely related to framework-html-meta-theme-color.md (the two work together but serve different purposes).

Audience: humans implementing proper dark mode support across an entire site (not just brand colors), AI assistants generating HTML head sections with appropriate scheme declarations, developers fixing "white flash on page load in dark mode" issues, designers seeing native form controls render in the wrong color scheme, and anyone troubleshooting "scrollbar is dark in light mode", "date picker is white on dark page", "page background flashes white briefly in dark mode", or "iOS Safari forces light rendering despite my CSS".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Color Scheme Mental Model (read this first)
5. The Distinction From theme-color (CRITICAL)
6. The Valid Values
7. Why The Order Matters (light dark vs dark light)
8. The CSS Equivalent (color-scheme on :root)
9. The Browser Default Behavior Without It
10. Form Controls And Canvas Color
11. The prefers-color-scheme Media Query Relationship
12. The Flash Of Incorrect Color (FOIC) Prevention
13. Browser Support Reality
14. The Bubbles Decision Framework
15. The Relationship Between color-scheme And theme-color
16. Asset Class And Use Case Recipes
17. Bubbles Standard Pattern (paste ready)
18. Audit Checklist
19. Common Pitfalls
20. Diagnostic Commands
21. Cross-References

---

## 1. DEFINITION

`<meta name="color-scheme">` is the HTML directive declaring which color schemes (light, dark, or both) the page is designed to support. Defined in CSS Color Adjustment Module Level 1 (W3C). Also expressible via CSS `color-scheme` property on `:root` or `html`.

```html
<head>
    <meta name="color-scheme" content="light dark">
</head>
```

Three structural facts shape how the directive works:

* **It is a capability declaration, not a color value.** Color-scheme tells the browser "this page knows how to render in these schemes". Theme-color (the previous framework) tells the browser "use this specific color for chrome". Different purposes.
* **It affects browser default rendering decisions.** Form controls (date pickers, dropdowns, scrollbars, native UI elements) adopt scheme appropriate styling. The canvas (the area outside the HTML element) takes a scheme appropriate default background.
* **The value order matters.** `light dark` means "primary light, fallback dark"; `dark light` means "primary dark, fallback light". This affects browser default rendering when no `prefers-color-scheme` media query has rendered yet.

For Bubbles client sites in 2026, the convention is to include `<meta name="color-scheme" content="light dark">` (or the CSS equivalent) for sites supporting both modes. This enables native form controls to render appropriately, eliminates the flash of incorrect color in dark mode, and signals to the browser that the page is dark mode aware.

---

## 2. WHY IT MATTERS

Seven independent considerations push correct color-scheme usage from "obscure CSS detail" to "actively managed signal" in 2025 and forward.

**Form controls adapt to the declared scheme.** Native form elements (date pickers, dropdowns, scrollbars, checkboxes, radio buttons, scroll arrows) use the browser's native styling. Without color-scheme declaration, these elements render in light mode regardless of system preference. With proper declaration, they adapt: scrollbars become dark in dark mode, date pickers use dark theme, etc.

**The canvas color (area outside HTML element) follows the scheme.** When the user scrolls past the page bottom on overscroll (iOS Safari rubber band, Android pull) or when content is shorter than viewport, the area outside the HTML element shows. Without color-scheme declaration, this area is always white. With it, the area adapts to the system color scheme.

**Flash of incorrect color (FOIC) is prevented.** Sites that implement dark mode via CSS only (without color-scheme declaration) may briefly show light rendering before CSS loads, then flash to dark. The color-scheme meta tag (read very early by the browser) prevents this.

**Browser default rendering decisions are influenced.** Some browsers, when uncertain about page color scheme support, default to light mode rendering for compatibility with legacy sites. The color-scheme declaration tells the browser the page is dark mode aware.

**iOS Safari specific behavior.** iOS Safari, particularly in PWA mode, uses color-scheme to determine native UI element appearance. Without the declaration, even sites with extensive CSS dark mode may have light mode native elements interspersed.

**The meta tag vs CSS distinction matters for timing.** The meta tag is read during HTML parsing (very early). The CSS property is read during stylesheet evaluation (slightly later). For preventing FOIC, the meta tag is faster.

**The order in the value affects browser behavior.** `light dark` and `dark light` are both valid but produce different default rendering when the browser hasn't yet resolved the `prefers-color-scheme` media query. The order signals which scheme the page prefers as the default.

**Cost of getting it wrong.** Incorrect color-scheme handling produces measurable polish damage. Real examples:

* Bubbles client implemented dark mode via CSS only. Native form date picker rendered in light mode on a dark page, creating jarring contrast. Adding `<meta name="color-scheme" content="light dark">` fixed the date picker to adopt dark theme automatically.
* Site supported dark mode via CSS. Users on dark mode systems saw brief white flash on page load before CSS loaded. Adding color-scheme meta tag eliminated the flash; the browser knew immediately that dark rendering was acceptable.
* Real estate client's website had a calendar widget. Without color-scheme declaration, the native iOS date picker rendered white on dark page. Users complained about the visual mismatch. Adding the meta tag synchronized the picker styling.
* Editorial blog (Joseph's ThatDeveloperGuy) had perfect dark mode CSS. iOS Safari users saw the scrollbar render as light on dark pages. Adding color-scheme meta tag fixed the scrollbar styling.
* Mental health client (Arkansas Counseling and Wellness) used dark mode to reduce visual stress for some users. Without proper color-scheme declaration, native browser tooltips and form elements rendered light, breaking the calming aesthetic. Adding the meta tag synchronized all UI elements.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The meta color-scheme tag plus its full operational context:

1. **The distinction from theme-color**: different purposes, sometimes confused.
2. **The valid values**: light, dark, both variants, only variants, normal.
3. **The value ordering**: light dark vs dark light differences.
4. **The CSS equivalent**: color-scheme property on :root.
5. **The browser default behavior**: what happens without the declaration.
6. **The form controls and canvas**: what changes when declared.
7. **FOIC prevention**: how the meta tag helps load timing.
8. **The Bubbles application**: per client decisions.

Section 5 disambiguates from theme-color (critical because they're often confused). Section 14 is the Bubbles decision framework.

---

## 4. THE COLOR SCHEME MENTAL MODEL (READ THIS FIRST)

When the browser fetches your page, it needs to make rendering decisions before all CSS loads:

```
Browser starts rendering the page.
   |
   v
==================== EARLY DECISIONS ====================
   |
   |---> What color scheme should the canvas use?
   |     (area outside HTML element; overscroll area)
   |
   |---> What styling for native form controls?
   |     (date pickers, scrollbars, dropdowns)
   |
   |---> What default text and background colors?
   |     (before custom CSS specifies)
   |
   |---> Should I render light or dark by default?

==================== WITHOUT color-scheme ====================
   |
   v
   Browser assumes LIGHT (for legacy compatibility):
   - White canvas.
   - Light form controls.
   - Black on white default text.
   - Light rendering.
   |
   v
   If page CSS specifies dark mode via media query:
   - Page background turns dark when CSS loads.
   - But canvas stays white (jarring).
   - Native form controls stay light (jarring).
   - Brief light flash visible.

==================== WITH color-scheme: light dark ====================
   |
   v
   Browser knows page supports both schemes:
   - Canvas adapts to system color scheme.
   - Native form controls adapt to system color scheme.
   - Default text and background colors appropriate to system scheme.
   - No flash; rendering matches scheme from first paint.

==================== THE DECLARATION FORMS ====================
   |
   v
   HTML meta tag (recommended for timing):
   <meta name="color-scheme" content="light dark">
   |
   CSS on :root (equivalent functionality, slightly later):
   :root { color-scheme: light dark; }
   |
   Both: works fine; redundant but harmless.

==================== ORDER VARIANTS ====================
   |
   "light dark" - primary light, fallback dark.
   "dark light" - primary dark, fallback light.
   "only light" - light only; force light even on dark mode systems.
   "only dark" - dark only; force dark even on light mode systems.
   "normal" - no preference (browser default).
```

Six rules govern the system:

1. **For dual mode sites: include `color-scheme: light dark`.**
2. **For dark only sites: use `only dark`.**
3. **Place the meta tag early in head.**
4. **Pair with proper CSS dark mode media queries.**
5. **Verify form controls and scrollbars adapt correctly.**
6. **Distinguish from theme-color (different purpose).**

A correctly configured Bubbles client site has color-scheme declaration matching the site's CSS dark mode support, theme-color for chrome coloring, and proper CSS variables that respond to prefers-color-scheme.

---

## 5. THE DISTINCTION FROM THEME-COLOR (CRITICAL)

Two related but distinct meta tags. Frequently confused.

### 5.1 The Side By Side Comparison

| Aspect | `theme-color` | `color-scheme` |
|---|---|---|
| What it declares | A specific color | Which schemes the page supports |
| Value | A color (e.g., `#6B46C1`) | A scheme list (e.g., `light dark`) |
| Affects | Browser chrome (URL bar, status bar) | Form controls, canvas, defaults |
| Browser support | Chrome, Firefox, iOS Safari (pre 26) | Most modern browsers |
| Order matters | No | Yes (`light dark` vs `dark light`) |
| Media query variants | Yes (`media="(prefers-color-scheme: dark)"`) | No (single declaration covers all) |

### 5.2 The Conceptual Distinction

* **theme-color** is "the color I want my chrome to be".
* **color-scheme** is "the schemes my page knows how to render in".

A site might:

* Declare `theme-color: #6B46C1` (purple chrome).
* Declare `color-scheme: light dark` (page supports both modes; form controls adapt).

Both can and should coexist.

### 5.3 The Combined Pattern

```html
<head>
    <!-- Color scheme support declaration -->
    <meta name="color-scheme" content="light dark">

    <!-- Chrome color (per scheme variant) -->
    <meta name="theme-color" content="#6B46C1">
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
</head>
```

The color-scheme declares the page supports both. The theme-color values declare the chrome color for each.

### 5.4 The Order Of Operations

When the browser renders:

1. Parses HTML head.
2. Sees `<meta name="color-scheme" content="light dark">` (decides scheme support).
3. Sees `<meta name="theme-color" content="#6B46C1">` and dark variant.
4. Reads `prefers-color-scheme` from system.
5. Applies appropriate color-scheme rendering (canvas, form controls).
6. Applies appropriate theme-color (chrome).
7. Loads CSS and applies further customization.

### 5.5 The Common Mistakes

**Mistake A: setting `theme-color` without `color-scheme`.**

The chrome color is set, but form controls and canvas still render light. Inconsistent appearance.

**Mistake B: setting `color-scheme` without dark mode CSS.**

The browser thinks page supports dark mode, but CSS only has light styles. Page may render with dark canvas but light content, producing visual breakage.

**Mistake C: confusing the two tags.**

Setting `<meta name="color-scheme" content="#6B46C1">` (wrong value type) does nothing useful. Setting `<meta name="theme-color" content="light dark">` is invalid color value.

### 5.6 The Bubbles Approach

Always include both when supporting dual mode:

```html
<meta name="color-scheme" content="light dark">
<meta name="theme-color" content="#6B46C1">
<meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
```

Plus CSS:

```css
@media (prefers-color-scheme: dark) {
    body {
        background-color: #1a1a1a;
        color: white;
    }
}
```

Three layers handle different rendering aspects coherently.

---

## 6. THE VALID VALUES

The `content` attribute accepts space separated keywords.

### 6.1 light

```html
<meta name="color-scheme" content="light">
```

The page is designed for light mode only. The browser will:

* Render form controls in light style.
* Use light canvas.
* Apply light defaults regardless of system preference.

Use for: light only sites where dark mode is not supported.

### 6.2 dark

```html
<meta name="color-scheme" content="dark">
```

The page is designed for dark mode only. The browser will:

* Render form controls in dark style.
* Use dark canvas.
* Apply dark defaults regardless of system preference.

Use for: dark only sites where light mode is not supported.

### 6.3 light dark

```html
<meta name="color-scheme" content="light dark">
```

The page supports both. The browser will:

* Use system preference (`prefers-color-scheme`) to choose.
* Default to light if undetermined.
* Adapt form controls and canvas accordingly.

Use for: sites with both light and dark mode CSS. **Most common Bubbles pattern.**

### 6.4 dark light

```html
<meta name="color-scheme" content="dark light">
```

Same as `light dark` but with dark preferred as default when undetermined. The browser will:

* Use system preference (`prefers-color-scheme`) to choose.
* Default to dark if undetermined.
* Adapt form controls and canvas accordingly.

Use for: sites that primarily target dark mode but support light. Joseph's example query used this ordering.

### 6.5 only light

```html
<meta name="color-scheme" content="only light">
```

The page forces light rendering. Overrides system dark mode preference.

* Browser ignores `prefers-color-scheme: dark` for canvas and form controls.
* CSS dark mode media queries still apply if author wrote them.

Use for: light only sites where you want to override aggressive system dark mode behavior.

### 6.6 only dark

```html
<meta name="color-scheme" content="only dark">
```

The page forces dark rendering. Overrides system light mode preference.

* Browser ignores `prefers-color-scheme: light` for canvas and form controls.

Use for: dark only sites where you want to ensure dark rendering even for users without dark mode enabled.

### 6.7 normal

```html
<meta name="color-scheme" content="normal">
```

The browser uses its default behavior. Same as not having the meta tag at all.

Use for: rarely. Effectively equivalent to omitting the tag.

### 6.8 The Bubbles Recommended Values

| Scenario | Recommended value |
|---|---|
| Dual mode site (CSS supports both light and dark) | `light dark` |
| Dark first site (Joseph's example) | `dark light` |
| Light only site | omit or `only light` |
| Dark only site (rare) | `only dark` |
| Sites unsure or in transition | `light` (safe default) |

For most Bubbles client work: `light dark`.

For Joseph's own sites (thatdeveloperguy.com, thataiguy.org) where Awwwards caliber aesthetics often favor dark: `dark light` is the natural choice.

---

## 7. WHY THE ORDER MATTERS (LIGHT DARK VS DARK LIGHT)

The order of values affects browser behavior in subtle but real ways.

### 7.1 The Ambiguity Case

When the browser cannot determine system preference (e.g., user has not set system theme, or browser doesn't expose the preference):

* `light dark`: browser uses light.
* `dark light`: browser uses dark.

This is the "first listed wins" behavior.

### 7.2 The Initial Render

Before the system preference is fully read and applied, the browser uses the first listed value for initial render:

* `light dark`: initial render is light, then may switch to dark if system prefers it.
* `dark light`: initial render is dark, then may switch to light if system prefers it.

This affects FOIC (flash of incorrect color). If most users are on light mode:
* `light dark`: no flash; initial is light, system confirms.
* `dark light`: brief dark flash, then light. Worse experience.

If most users are on dark mode:
* `light dark`: brief light flash, then dark. Worse experience.
* `dark light`: no flash; initial is dark, system confirms.

### 7.3 The Demographic Question

For the Bubbles client base (NWA/SWMO local businesses, federal subcontractors):

* Most users are on default light mode systems.
* Dark mode adoption is growing but still minority (~30-40% per various studies).
* Default `light dark` matches majority experience.

For Joseph's own sites with technical audience:

* Developers and designers more likely to use dark mode.
* `dark light` may match audience better.

### 7.4 The CSS Connection

The order also affects CSS behavior when using the `light-dark()` CSS function:

```css
color: light-dark(black, white);
/* Returns black in light mode, white in dark mode */
/* The order in color-scheme affects which is the default */
```

The function uses the color-scheme order to determine which value to pick first.

### 7.5 The Bubbles Choice

For most clients: `light dark` (matches user expectations).

For Joseph's tech sites: `dark light` (matches audience).

Document the choice per client.

---

## 8. THE CSS EQUIVALENT (COLOR-SCHEME ON :ROOT)

The CSS `color-scheme` property provides equivalent functionality.

### 8.1 The CSS Pattern

```css
:root {
    color-scheme: light dark;
}
```

Or:

```css
html {
    color-scheme: light dark;
}
```

Both work. `:root` is the more modern convention.

### 8.2 The HTML Meta vs CSS Difference

| Aspect | HTML meta | CSS property |
|---|---|---|
| Parse timing | During HTML parsing (very early) | During stylesheet loading (slightly later) |
| FOIC prevention | Better | Acceptable |
| Code location | head | stylesheet |
| Browser support | Slightly more variable | More consistent |
| Override behavior | Both have similar specificity | Both have similar specificity |

### 8.3 The Combined Use

Both can be present; they don't conflict:

```html
<head>
    <meta name="color-scheme" content="light dark">
    <link rel="stylesheet" href="/styles.css">
</head>
```

```css
/* /var/www/sites/example.com/styles.css */

:root {
    color-scheme: light dark;
}
```

Redundant but harmless. The meta tag provides earlier signal; the CSS provides the same information with stylesheet context.

### 8.4 The Bubbles Preference

For Bubbles: meta tag preferred (better FOIC prevention).

The CSS property is a fine alternative when the meta tag cannot be added (e.g., third party CMS that doesn't allow custom meta tags but allows CSS).

### 8.5 The Per Element Use

The CSS color-scheme can also be applied to specific elements:

```css
.dark-content-section {
    color-scheme: dark;
    /* Form controls inside this section render in dark mode */
}

.light-content-section {
    color-scheme: light;
    /* Form controls inside this section render in light mode */
}
```

Useful for sections with intentional contrast against the rest of the page.

For typical Bubbles sites: only `:root` or `html` level declarations.

---

## 9. THE BROWSER DEFAULT BEHAVIOR WITHOUT IT

When no color-scheme is declared, browsers apply legacy behavior.

### 9.1 The Legacy Default

Without color-scheme:

* Browser assumes the page is light only.
* Form controls render in light style.
* Canvas is white.
* Default text is black.
* Native UI elements use light styling.

### 9.2 The Behavior With CSS Dark Mode (No color-scheme)

If the page has CSS dark mode via `prefers-color-scheme: dark`:

```css
@media (prefers-color-scheme: dark) {
    body {
        background-color: #1a1a1a;
        color: white;
    }
}
```

But no color-scheme meta tag:

* Page body renders dark per CSS.
* But form controls remain LIGHT (legacy default).
* Canvas remains WHITE.
* Visual mismatch.

The result: dark body with light form controls and white canvas areas. Jarring.

### 9.3 The Fix

Add color-scheme:

```html
<meta name="color-scheme" content="light dark">
```

Now form controls and canvas adapt to scheme.

### 9.4 The Verification

```bash
# Find a page with dark mode CSS but no color-scheme
URL=$1
HAS_DARK_CSS=$(curl -s "$URL" | grep -c "prefers-color-scheme: dark")
HAS_COLOR_SCHEME=$(curl -s "$URL" | grep -c 'name="color-scheme"')

if [ "$HAS_DARK_CSS" -gt "0" ] && [ "$HAS_COLOR_SCHEME" = "0" ]; then
    echo "PROBLEM: $URL has dark mode CSS but no color-scheme declaration"
fi
```

### 9.5 The Bubbles Convention

For every Bubbles client site with dark mode CSS: include color-scheme meta tag. Always.

---

## 10. FORM CONTROLS AND CANVAS COLOR

The two main browser rendering aspects affected by color-scheme.

### 10.1 Form Controls That Adapt

The native browser styling of these elements changes with color-scheme:

* `<input type="date">`, `<input type="datetime-local">`, `<input type="time">`: date/time pickers adapt.
* `<input type="color">`: color picker UI adapts.
* `<input type="file">`: file upload button adapts.
* `<input type="range">`: range slider adapts.
* `<input type="checkbox">`, `<input type="radio">`: native styling adapts.
* `<select>`: dropdown rendering adapts.
* `<progress>`, `<meter>`: progress bar styling adapts.
* Scrollbars (where browser native): adapt to scheme.

When color-scheme is set to `light dark`:

* Light mode users: all the above render in light style.
* Dark mode users: all the above render in dark style.

### 10.2 Form Controls That Don't Automatically Adapt

* `<input type="text">`, `<input type="email">`, `<textarea>`: these have minimal native styling; need CSS for dark mode.
* `<button>`: only basic adaptation; full styling requires CSS.

For these, your CSS still needs to handle dark mode explicitly:

```css
input[type="text"],
input[type="email"],
textarea {
    background: white;
    color: black;
    border: 1px solid #ccc;
}

@media (prefers-color-scheme: dark) {
    input[type="text"],
    input[type="email"],
    textarea {
        background: #2a2a2a;
        color: white;
        border: 1px solid #555;
    }
}
```

### 10.3 The Canvas Color

The "canvas" in CSS terms is:

* The area between the viewport and the HTML element.
* Visible during overscroll (iOS rubber band, Android pull-to-refresh).
* Visible when content is shorter than viewport (the area below `<body>`).

Without color-scheme: white canvas.
With color-scheme matching dark mode: dark canvas.

This eliminates white flashes during overscroll on dark mode sites.

### 10.4 The Scrollbar Specifics

CSS allows custom scrollbar styling but native scrollbars are still common. The color-scheme declaration affects them:

```css
/* Custom scrollbar (optional override) */
::-webkit-scrollbar {
    width: 10px;
}

::-webkit-scrollbar-thumb {
    background: var(--scroll-thumb);
}

:root {
    --scroll-thumb: rgba(0, 0, 0, 0.3);
}

@media (prefers-color-scheme: dark) {
    :root {
        --scroll-thumb: rgba(255, 255, 255, 0.3);
    }
}
```

With color-scheme set: even without custom CSS, the native scrollbar adapts.

### 10.5 The Tooltip Specifics

Browser tooltips (HTML `title` attribute) use native rendering. With color-scheme set, they adopt the scheme's tooltip styling.

---

## 11. THE PREFERS-COLOR-SCHEME MEDIA QUERY RELATIONSHIP

The CSS media query and the meta tag work together but are distinct.

### 11.1 The Two Concepts

* **`color-scheme` (HTML meta or CSS property)**: declares which schemes the page supports.
* **`prefers-color-scheme` (CSS media query)**: detects which scheme the user prefers.

### 11.2 The Working Pattern

```html
<head>
    <!-- 1. Declare the page supports both schemes -->
    <meta name="color-scheme" content="light dark">
</head>
```

```css
/* 2. Provide CSS for light mode (default) */
body {
    background: white;
    color: black;
}

/* 3. Override for dark mode users */
@media (prefers-color-scheme: dark) {
    body {
        background: #1a1a1a;
        color: white;
    }
}
```

Step 1 enables the browser to render appropriately at parse time (canvas, form controls). Steps 2 and 3 provide the custom CSS.

### 11.3 The Absence Pattern

Without color-scheme:

```css
/* Without color-scheme meta */
@media (prefers-color-scheme: dark) {
    body {
        background: #1a1a1a;
        color: white;
    }
}
```

* CSS dark mode applied to body.
* Form controls stay light (no color-scheme signal).
* Canvas stays white.
* Mismatch.

### 11.4 The Override Pattern

```css
/* Force light or dark on specific sections */
.force-light {
    color-scheme: light;
}

.force-dark {
    color-scheme: dark;
}
```

```html
<div class="force-light">
    <!-- Always renders light, regardless of system or page-level color-scheme -->
</div>
```

For most Bubbles sites: only the root level color-scheme is used.

### 11.5 The JavaScript Detection

For dynamic behavior:

```javascript
// Detect current scheme
const isDarkMode = window.matchMedia('(prefers-color-scheme: dark)').matches;
console.log('Dark mode:', isDarkMode);

// React to changes
window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', (e) => {
    console.log('Color scheme changed:', e.matches ? 'dark' : 'light');
    // Update any JavaScript dependent styling
});
```

For typical Bubbles sites: CSS handles scheme switching; JavaScript usually unnecessary.

---

## 12. THE FLASH OF INCORRECT COLOR (FOIC) PREVENTION

FOIC is the brief moment when a page renders in the wrong scheme before correcting.

### 12.1 The FOIC Phenomenon

1. User on dark mode system loads page.
2. Browser renders initial HTML (no CSS yet).
3. Default white background appears briefly.
4. CSS loads with dark mode media query.
5. Page transitions to dark.
6. User sees brief white flash.

### 12.2 The Meta Tag Solution

```html
<head>
    <meta name="color-scheme" content="light dark">
</head>
```

Now:

1. User on dark mode loads page.
2. Browser parses meta tag in head.
3. Browser knows page supports dark mode.
4. Initial render uses dark canvas (matches system preference).
5. CSS loads; page already in dark mode.
6. No flash.

### 12.3 The Placement Matters

The meta tag must be in `<head>` and parsed before significant rendering. Place after charset and viewport:

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="color-scheme" content="light dark">

    <title>Page Title</title>
    <meta name="description" content="...">

    <!-- ... other tags ... -->
</head>
```

Within the first 1024 bytes for maximum effectiveness.

### 12.4 The CSS only Alternative

The CSS `color-scheme` property on `:root` also helps but is read slightly later:

```css
:root {
    color-scheme: light dark;
}
```

If CSS is in `<head>` via inline `<style>` or `<link>` early in head: FOIC prevention is good.
If CSS is in external file loaded later: FOIC prevention is less effective.

For maximum FOIC prevention: meta tag.

### 12.5 The JavaScript Anti Pattern

Some sites attempt to fix FOIC via JavaScript:

```javascript
// On page load, check scheme and apply
if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
    document.documentElement.classList.add('dark-mode');
}
```

```css
.dark-mode {
    background: #1a1a1a;
}
```

This works but has timing issues. The meta tag is faster and more reliable.

### 12.6 The Bubbles FOIC Prevention Stack

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="color-scheme" content="light dark">  <!-- FOIC prevention -->

    <style>
        /* Inline critical CSS with dark mode support */
        :root {
            --bg: white;
            --fg: black;
        }

        @media (prefers-color-scheme: dark) {
            :root {
                --bg: #1a1a1a;
                --fg: white;
            }
        }

        body {
            background: var(--bg);
            color: var(--fg);
            margin: 0;
        }
    </style>

    <!-- ... rest of head ... -->
</head>
```

Three layers (meta color-scheme, inline critical CSS, CSS variables) ensure no FOIC.

---

## 13. BROWSER SUPPORT REALITY

The color-scheme behavior across browsers in 2026.

### 13.1 Modern Browsers (Full Support)

| Browser | Meta tag | CSS property | Notes |
|---|---|---|---|
| Chrome 81+ | Yes | Yes | Full support |
| Edge 81+ | Yes | Yes | Full support |
| Firefox 96+ | Yes | Yes | Full support |
| Safari 13+ | Yes | Yes | Full support including iOS Safari |
| Brave | Yes | Yes | Full (Chromium based) |
| Opera | Yes | Yes | Full (Chromium based) |

### 13.2 The Mobile Coverage

iOS Safari 13+ supports color-scheme. iOS 26 Safari (where theme-color was dropped) still supports color-scheme. The two features were managed differently by Apple.

Android Chrome supports color-scheme.

### 13.3 The Legacy Browser Behavior

IE 11 and older browsers: no support; ignored. Page renders without scheme awareness (effectively `light`).

For 2026 Bubbles client sites: IE is not a concern (Microsoft retired IE; Edge replaced it).

### 13.4 The Cross Browser Verification

```bash
# Verify support by checking page rendering
# Manual: open in each browser, toggle dark mode, observe:
# - Canvas color
# - Scrollbar color
# - Date picker (if any)
# - Form control rendering

# Automated: Lighthouse can check for color-scheme presence
lighthouse https://example.com/ \
    --only-audits=color-scheme \
    --output=json --output-path=/tmp/lh.json --quiet
```

(Note: Lighthouse `color-scheme` audit is informational; not part of scoring.)

### 13.5 The Bubbles Confidence Level

For Bubbles client sites in 2026:

* Browser support is widespread and reliable.
* The meta tag works consistently.
* The CSS property works consistently.
* iOS 26 Safari supports it despite dropping theme-color.

Include the declaration. It works.

---

## 14. THE BUBBLES DECISION FRAMEWORK

Each client site needs a color-scheme decision.

### 14.1 The Decision Questions

1. **Does the site support dark mode (via CSS media queries)?**
   - Yes: include color-scheme with both light and dark.
   - No: omit, or use `only light`.

2. **What is the audience demographic?**
   - General consumer: `light dark` (light default).
   - Technical/design audience: consider `dark light`.

3. **What is the brand aesthetic?**
   - Bright/light brand: `light dark`.
   - Dark/moody brand: `dark light`.

4. **Are there form heavy pages?**
   - Yes: color-scheme is critical for form control consistency.
   - No: less critical but still helpful.

### 14.2 The Per Client Recommendations

**ThatDeveloperGuy.com (Joseph's primary site)**:

```html
<meta name="color-scheme" content="dark light">
<!-- Technical audience; brand aesthetic supports either; dark first matches Joseph's example -->
```

**TCB Fight Factory (purple/black brand)**:

```html
<meta name="color-scheme" content="dark light">
<!-- Black is a brand color; dark first matches aesthetic -->
```

**Arkansas Counseling and Wellness Services**:

```html
<meta name="color-scheme" content="light dark">
<!-- Calming, welcoming aesthetic; light default; dark for users who prefer reduced visual stress -->
```

**Handled Tax and Advisory (Amanda Emerdinger)**:

```html
<meta name="color-scheme" content="light dark">
<!-- Professional financial service; light default -->
```

**White River Cabins**:

```html
<meta name="color-scheme" content="light dark">
<!-- Travel/leisure; light evokes outdoors, openness -->
```

**Greenough's Guide Service**:

```html
<meta name="color-scheme" content="light dark">
<!-- Outdoor activity; light evokes daylight fishing -->
```

**Real estate clients (Local Living, Diana Undergust)**:

```html
<meta name="color-scheme" content="light dark">
<!-- Standard professional service; light default -->
```

**Federal subcontractor (WeCoverUSA)**:

```html
<meta name="color-scheme" content="light dark">
<!-- Standard government/business aesthetic -->
```

### 14.3 The Documentation Pattern

For each client project, document:

```
Color scheme:
  Declared: light dark
  CSS dark mode: yes (via prefers-color-scheme media query)
  Form controls verified: yes
  FOIC tested: yes
```

### 14.4 The Default For Bubbles

For most Bubbles client work: `light dark`.

For sites with intentional dark first aesthetic: `dark light`.

---

## 15. THE RELATIONSHIP BETWEEN COLOR-SCHEME AND THEME-COLOR

The two meta tags work together for complete browser chrome and rendering coverage.

### 15.1 The Coverage Matrix

| Concern | Handled by |
|---|---|
| Browser chrome color | theme-color |
| Status bar color (Android) | theme-color |
| Browser chrome dark mode | theme-color media variant |
| Canvas color (overscroll area) | color-scheme |
| Form control styling | color-scheme |
| Scrollbar color | color-scheme |
| Date picker styling | color-scheme |
| FOIC prevention | color-scheme |
| iOS 26 Safari chrome | body background CSS (not theme-color anymore) |

### 15.2 The Combined Bubbles Pattern

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Scheme support (FOIC, form controls, canvas) -->
    <meta name="color-scheme" content="light dark">

    <!-- Chrome color (Android, Chrome Desktop) -->
    <meta name="theme-color" content="#6B46C1">
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">

    <title>Page Title</title>
    <meta name="description" content="...">
</head>
<body>
    <!-- ... content ... -->
</body>

<style>
/* Page level dark mode */
:root {
    --bg: white;
    --fg: black;
    --brand: #6B46C1;
}

@media (prefers-color-scheme: dark) {
    :root {
        --bg: #1a1a1a;
        --fg: white;
        --brand: #6B46C1; /* brand color may stay or shift */
    }
}

body {
    background: var(--bg);
    color: var(--fg);
    margin: 0;
}

/* For iOS 26 Safari chrome coverage */
body {
    background-color: #6B46C1;  /* matches theme-color */
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #0a0a0a;  /* matches dark theme-color */
    }
}
</style>
```

Five signals:

1. `color-scheme` meta: declares support.
2. `theme-color` meta: chrome color (light).
3. `theme-color` meta media variant: chrome color (dark).
4. CSS variables: page level theme.
5. Body background-color: iOS 26 Safari chrome workaround.

All working together for comprehensive browser chrome and rendering control.

### 15.3 The Audit For Both

```bash
#!/bin/bash
# Check both tags
URL=$1

echo "=== color-scheme and theme-color audit for $URL ==="

# color-scheme
CS=$(curl -s "$URL" | grep -oE 'meta name="color-scheme" content="[^"]+"' | head -1)
echo "color-scheme: ${CS:-NOT SET}"

# theme-color
TC=$(curl -s "$URL" | grep -oE 'meta name="theme-color" content="[^"]+"' | head -3)
echo "theme-color tags:"
echo "$TC"

# CSS dark mode
HAS_DARK=$(curl -s "$URL" | grep -c "prefers-color-scheme: dark")
echo "Dark mode CSS occurrences: $HAS_DARK"

# Cross check
if [ "$HAS_DARK" -gt "0" ] && [ -z "$CS" ]; then
    echo "WARNING: dark mode CSS but no color-scheme declaration"
fi
```

---

## 16. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 16.1 Canonical Bubbles head (light dark)

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Color scheme support -->
    <meta name="color-scheme" content="light dark">

    <!-- Theme color (chrome) -->
    <meta name="theme-color" content="#6B46C1">
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- ... -->
</head>
```

### 16.2 Dark first variant (Joseph's example)

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <meta name="color-scheme" content="dark light">

    <meta name="theme-color" content="#0a0a0a">
    <meta name="theme-color" content="#6B46C1" media="(prefers-color-scheme: light)">

    <!-- ... -->
</head>
```

The page primarily targets dark mode. The default chrome is near black; light mode override is brand purple.

### 16.3 Light only (no dark mode support)

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <meta name="color-scheme" content="light">
    <!-- Or just omit -->

    <meta name="theme-color" content="#6B46C1">

    <!-- ... -->
</head>
```

### 16.4 Force light (override system dark mode)

```html
<head>
    <meta name="color-scheme" content="only light">
</head>
```

User on dark mode system: form controls and canvas still render light. Useful when the site cannot be safely rendered in dark mode (legacy CSS, accessibility concerns).

Use sparingly; most users expect dark mode to work where they've requested it.

### 16.5 Force dark (override system light mode)

```html
<head>
    <meta name="color-scheme" content="only dark">
</head>
```

User on light mode system: form controls and canvas still render dark. Useful for dark first sites (e.g., immersive experiences).

For most Bubbles work: avoid; respect user preference.

### 16.6 CSS only declaration

```css
/* /var/www/sites/example.com/styles.css */

:root {
    color-scheme: light dark;
}
```

Use when meta tag cannot be added (third party CMS).

### 16.7 The full CSS scheme support pattern

```css
:root {
    color-scheme: light dark;
    --bg: white;
    --fg: #1a1a1a;
    --brand: #6B46C1;
}

@media (prefers-color-scheme: dark) {
    :root {
        --bg: #1a1a1a;
        --fg: white;
        --brand: #6B46C1;
    }
}

body {
    background: var(--bg);
    color: var(--fg);
    margin: 0;
    font-family: system-ui, -apple-system, sans-serif;
}

a {
    color: var(--brand);
}

input,
textarea,
select {
    background: var(--bg);
    color: var(--fg);
    border: 1px solid #ccc;
    padding: 0.5rem;
    font-size: 1rem;
}

@media (prefers-color-scheme: dark) {
    input,
    textarea,
    select {
        border-color: #555;
        background: #2a2a2a;
    }
}
```

### 16.8 The JavaScript theme toggle (manual user override)

```html
<button id="theme-toggle">Toggle Theme</button>

<script>
const toggle = document.getElementById('theme-toggle');
const root = document.documentElement;

// Detect current preference
let currentTheme = window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';

// Apply override on toggle
toggle.addEventListener('click', () => {
    currentTheme = currentTheme === 'dark' ? 'light' : 'dark';
    root.style.colorScheme = currentTheme;
    document.body.classList.toggle('force-dark', currentTheme === 'dark');
    document.body.classList.toggle('force-light', currentTheme === 'light');
});
</script>

<style>
body.force-dark {
    background: #1a1a1a;
    color: white;
}

body.force-light {
    background: white;
    color: #1a1a1a;
}
</style>
```

JavaScript override allows users to switch despite system preference.

### 16.9 FastAPI server side per request

```python
# Allow user to set scheme via cookie
from fastapi import FastAPI, Request, Response

app = FastAPI()

@app.get("/")
async def homepage(request: Request, response: Response):
    user_scheme = request.cookies.get("color_scheme", "light dark")
    return templates.TemplateResponse("homepage.html", {
        "request": request,
        "color_scheme": user_scheme,
    })

@app.post("/api/set-scheme/")
async def set_scheme(scheme: str, response: Response):
    response.set_cookie("color_scheme", scheme, max_age=31536000)
    return {"scheme": scheme}
```

```html
{# template #}
<head>
    <meta name="color-scheme" content="{{ color_scheme }}">
</head>
```

User's scheme preference persists across visits via cookie.

### 16.10 The audit script

```bash
#!/bin/bash
# /usr/local/bin/color-scheme-audit.sh

echo "=== Color scheme audit ==="

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"

    if [ -f "$INDEX" ]; then
        CS=$(grep -oE 'meta name="color-scheme" content="[^"]+"' "$INDEX" | head -1)
        HAS_DARK_CSS=$(grep -c "prefers-color-scheme: dark" "$INDEX")

        # Also check CSS files
        for css in "$site_dir"*.css; do
            if [ -f "$css" ]; then
                CSS_DARK=$(grep -c "prefers-color-scheme: dark" "$css")
                HAS_DARK_CSS=$((HAS_DARK_CSS + CSS_DARK))
            fi
        done

        if [ "$HAS_DARK_CSS" -gt "0" ] && [ -z "$CS" ]; then
            echo "MISSING color-scheme: $SITE has dark mode CSS but no declaration"
        elif [ -n "$CS" ]; then
            : # OK
        else
            echo "NO DARK MODE: $SITE has no dark mode CSS or declaration (may be light only by design)"
        fi
    fi
done
```

---

## 17. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles color-scheme configuration.

### 17.1 The Template Default

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Color scheme and theme coverage -->
    <meta name="color-scheme" content="light dark">
    <meta name="theme-color" content="{{ brand_primary }}">
    <meta name="theme-color" content="{{ brand_dark }}" media="(prefers-color-scheme: dark)">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- Inline critical CSS for FOIC prevention -->
    <style>
        :root {
            color-scheme: light dark;
            --bg: white;
            --fg: #1a1a1a;
            --brand: {{ brand_primary }};
        }

        @media (prefers-color-scheme: dark) {
            :root {
                --bg: #1a1a1a;
                --fg: white;
                --brand: {{ brand_primary }};
            }
        }

        body {
            background: var(--bg);
            color: var(--fg);
            margin: 0;
            font-family: system-ui, -apple-system, sans-serif;

            /* iOS 26 Safari chrome workaround */
            background-color: {{ brand_primary }};
        }

        @media (prefers-color-scheme: dark) {
            body {
                background-color: {{ brand_dark }};
            }
        }

        main {
            background: var(--bg);
        }
    </style>

    <!-- External stylesheet -->
    <link rel="stylesheet" href="/styles.css">
</head>
<body>
    <main>
        <!-- ... content ... -->
    </main>
</body>
```

### 17.2 The Per Client Configuration

```python
# /opt/bubbles/services/example.com/client_config.py

CLIENT_COLOR_SCHEMES = {
    "thatdeveloperguy": {
        "scheme": "dark light",
        "brand_primary": "#6B46C1",
        "brand_dark": "#0a0a0a",
    },
    "tcb_fight_factory": {
        "scheme": "dark light",
        "brand_primary": "#6B46C1",
        "brand_dark": "#0a0a0a",
    },
    "arkansas_counseling": {
        "scheme": "light dark",
        "brand_primary": "#calm_color_here",
        "brand_dark": "#calm_dark_here",
    },
    "handled_tax": {
        "scheme": "light dark",
        "brand_primary": "#prof_color_here",
        "brand_dark": "#prof_dark_here",
    },
    # ... per client
}
```

### 17.3 The Audit Workflow

```bash
# Weekly across all sites
color-scheme-audit.sh

# After template changes
nginx -t && systemctl reload nginx
systemctl restart fastapi-sidecar
```

### 17.4 The Verification Per Page

```bash
# /usr/local/bin/color-scheme-check.sh
URL=$1

echo "=== Color scheme check for $URL ==="

CS=$(curl -s "$URL" | grep -oE 'meta name="color-scheme" content="[^"]+"' | head -1)
echo "color-scheme meta: ${CS:-NOT SET}"

CSS_PROP=$(curl -s "$URL" | grep -oE 'color-scheme: *[a-z ]+' | head -1)
echo "CSS color-scheme property: ${CSS_PROP:-NOT SET}"

DARK_CSS=$(curl -s "$URL" | grep -c "prefers-color-scheme: dark")
echo "Dark mode CSS occurrences: $DARK_CSS"

if [ "$DARK_CSS" -gt "0" ] && [ -z "$CS" ] && [ -z "$CSS_PROP" ]; then
    echo "PROBLEM: dark mode CSS without color-scheme declaration"
elif [ -n "$CS" ] || [ -n "$CSS_PROP" ]; then
    echo "OK: color-scheme declared"
else
    echo "INFO: light only (no dark mode)"
fi
```

---

## 18. AUDIT CHECKLIST

Run through these 35 items for production deployment.

### Core meta tag

1. [ ] `<meta name="color-scheme">` present on dual mode sites.
2. [ ] Value is appropriate per site design.
3. [ ] Value placement is early in `<head>` (after charset and viewport).

### Value choice

4. [ ] `light dark` used for sites with light default.
5. [ ] `dark light` used for sites with dark default (e.g., Joseph's tech sites).
6. [ ] `only light` used only when overriding system dark mode is intentional.
7. [ ] `only dark` used only when overriding system light mode is intentional.

### CSS alignment

8. [ ] CSS dark mode media queries present where color-scheme allows dark.
9. [ ] CSS variables for background, color, brand defined per scheme.
10. [ ] Body uses CSS variables (not hard coded colors).

### Form controls

11. [ ] Native form controls verified to adapt (date picker, scrollbar).
12. [ ] Text inputs and textareas have CSS dark mode rules.
13. [ ] Buttons have CSS dark mode rules.

### Canvas color

14. [ ] Overscroll area adopts dark canvas in dark mode.
15. [ ] No white flash when scrolling past content.

### FOIC prevention

16. [ ] Meta tag in head (parsed early).
17. [ ] Inline critical CSS for above the fold styling.
18. [ ] No flash of incorrect color observed on real device test.

### theme-color coordination

19. [ ] theme-color set for chrome (per framework-html-meta-theme-color.md).
20. [ ] theme-color dark mode variant set.
21. [ ] Body background-color matches theme-color (iOS 26 coverage).

### Browser testing

22. [ ] Chrome Android: scheme works.
23. [ ] iOS Safari: scheme works (form controls, canvas).
24. [ ] Firefox: scheme works.
25. [ ] Brave/Edge: scheme works.

### Color scheme transitions

26. [ ] User toggling system dark mode: page adapts.
27. [ ] No JavaScript required for basic scheme switching.

### Documentation

28. [ ] color-scheme decision documented per client.
29. [ ] Pairing with theme-color documented.
30. [ ] Dark mode design rationale documented.

### CSS conventions

31. [ ] Single canonical CSS for dark mode (no fragments).
32. [ ] CSS variables used (not duplicate selectors per scheme).

### Cross cutting

33. [ ] Pages with no dark mode have appropriate color-scheme (light or omit).
34. [ ] noindex pages don't need elaborate dark mode handling.
35. [ ] Print stylesheet considers neither scheme (light by default).

A site that passes all 35 has correctly configured color-scheme for cross browser dark mode compatibility, FOIC prevention, and native form control consistency.

---

## 19. COMMON PITFALLS

Twelve patterns to recognize and avoid.

**Pitfall 1: Dark mode CSS without color-scheme declaration.**
Symptom: page body turns dark, but scrollbars, date pickers, canvas stay light.
Why it breaks: browser doesn't know page supports dark mode.
Fix: add `<meta name="color-scheme" content="light dark">`.

**Pitfall 2: Confusing color-scheme and theme-color.**
Symptom: `<meta name="color-scheme" content="#6B46C1">` or `<meta name="theme-color" content="light dark">`.
Why it breaks: wrong value type for the wrong meta tag.
Fix: color-scheme takes scheme keywords; theme-color takes color values.

**Pitfall 3: color-scheme value with comma separator.**
Symptom: `<meta name="color-scheme" content="light, dark">` (with comma).
Why it breaks: spec requires space separated, not comma.
Fix: `<meta name="color-scheme" content="light dark">`.

**Pitfall 4: Hard coded background colors throughout CSS.**
Symptom: changing scheme doesn't affect most elements.
Why it breaks: CSS uses `background: white` not `background: var(--bg)`.
Fix: use CSS variables for all themeable colors.

**Pitfall 5: color-scheme with `light` value on a site that supports dark.**
Symptom: site has dark mode CSS but color-scheme says light only.
Why it breaks: contradictory signals; browser confused.
Fix: change color-scheme to `light dark`.

**Pitfall 6: color-scheme on body but expected to affect root.**
Symptom: `<body style="color-scheme: light dark">` instead of root.
Why it breaks: scope is body only; html root and canvas not affected.
Fix: place on `:root` or `html`, or use meta tag.

**Pitfall 7: Force dark on a light only site.**
Symptom: light only site has `color-scheme: dark`. Looks broken.
Why it breaks: form controls and canvas render dark; CSS still light.
Fix: change to `light` or `light dark`.

**Pitfall 8: Color scheme not aligned across iframe boundaries.**
Symptom: iframe content renders in one scheme, parent in another.
Why it breaks: iframes have their own color-scheme; parent doesn't propagate.
Fix: set color-scheme in the iframe content as well.

**Pitfall 9: Print stylesheet doesn't handle color scheme.**
Symptom: printed page renders dark.
Why it breaks: dark mode CSS applies to print.
Fix: explicit print stylesheet with light colors only.

```css
@media print {
    body {
        background: white !important;
        color: black !important;
    }
}
```

**Pitfall 10: User's manual override not respected.**
Symptom: user has explicitly chosen dark mode via UI toggle, but system preference overrides.
Why it breaks: CSS uses prefers-color-scheme; ignores user override.
Fix: implement toggle via class or cookie that overrides media query.

**Pitfall 11: color-scheme on multiple roots.**
Symptom: both `<meta name="color-scheme">` and `:root { color-scheme: ... }` with different values.
Why it breaks: undefined behavior; one wins (typically the more specific).
Fix: choose one source of truth; remove the other.

**Pitfall 12: WordPress or CMS auto generating color-scheme.**
Symptom: Yoast or theme plugin sets color-scheme without considering site design.
Why it breaks: plugin doesn't know if dark mode is actually implemented.
Fix: disable plugin auto generation; configure manually.

---

## 20. DIAGNOSTIC COMMANDS

Reference of commands useful for color-scheme investigation.

### Inspect a single URL

```bash
# Get color-scheme meta tag
curl -s https://example.com/ | grep -oE 'meta name="color-scheme" content="[^"]+"'

# Get CSS color-scheme property
curl -s https://example.com/ | grep -oE 'color-scheme: *[a-z]+( +[a-z]+)?' | head -3

# Check for dark mode CSS
curl -s https://example.com/ | grep -c "prefers-color-scheme: dark"
```

### Bulk audit Bubbles client sites

```bash
color-scheme-audit.sh
```

### Verify scheme support coverage

```bash
URL=$1
color-scheme-check.sh "$URL"
```

### Cross check with theme-color

```bash
URL=$1
echo "=== Both schemes coordinated for $URL ==="

CS=$(curl -s "$URL" | grep -oE 'meta name="color-scheme" content="[^"]+"' | head -1)
TC=$(curl -s "$URL" | grep -oE 'meta name="theme-color" content="[^"]+"' | head -1)

echo "color-scheme: ${CS:-MISSING}"
echo "theme-color: ${TC:-MISSING}"

if [ -n "$CS" ] && [ -z "$TC" ]; then
    echo "WARNING: color-scheme without theme-color"
fi

if [ -z "$CS" ] && [ -n "$TC" ]; then
    echo "WARNING: theme-color without color-scheme"
fi
```

### Browser DevTools verification

In Chrome DevTools:

1. Open the page.
2. Open Rendering panel (More Tools > Rendering).
3. Look for "Emulate CSS media feature prefers-color-scheme".
4. Toggle between light, dark, no-preference.
5. Verify page adapts (or stays per `only` declarations).

In Safari DevTools (for iOS Safari):

1. Open the page in iOS Safari.
2. Connect Mac Safari Web Inspector.
3. Develop > [Device] > [Page].
4. Use the Color Scheme override.
5. Verify scheme adaptation.

### Visual testing

For each Bubbles client site:

1. Set system to dark mode.
2. Load the page.
3. Observe:
   - Is there a flash of light content?
   - Is the scrollbar dark?
   - If forms present: are inputs dark? Are date pickers dark?
   - Overscroll to see canvas: is it dark?
4. Toggle to light mode.
5. Verify all of the above switches to light.

### After configuration changes

```bash
nginx -t && systemctl reload nginx
systemctl restart fastapi-sidecar

# Verify
color-scheme-check.sh https://example.com/
```

---

## 21. CROSS-REFERENCES

* [framework-html-meta-theme-color.md](framework-html-meta-theme-color.md): distinct from color-scheme; declares chrome color rather than scheme support. The two work together.
* [framework-html-meta-viewport.md](framework-html-meta-viewport.md): viewport-fit=cover for notched displays; combined with color-scheme for full bleed mobile.
* [framework-html-meta-charset.md](framework-html-meta-charset.md): place color-scheme after charset and viewport.
* [framework-html-meta-description.md](framework-html-meta-description.md): order in head (charset, viewport, color-scheme, description).
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): color-scheme not ranking but user experience signals matter.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including color scheme decisions per client.
* W3C CSS Color Adjustment Module Level 1: https://www.w3.org/TR/css-color-adjust-1/
* MDN meta name=color-scheme: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name/color-scheme
* MDN CSS color-scheme property: https://developer.mozilla.org/en-US/docs/Web/CSS/color-scheme
* MDN prefers-color-scheme media query: https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme
* Can I Use color-scheme: https://caniuse.com/?search=color-scheme
* CSS Tricks Color Scheme: https://css-tricks.com/come-to-the-light-dark-side/

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Include `<meta name="color-scheme" content="light dark">` on every site with dark mode CSS. Pair with theme-color for chrome.**

### The canonical patterns

```html
<!-- Standard light first (most clients) -->
<meta name="color-scheme" content="light dark">

<!-- Dark first (Joseph's tech sites, TCB) -->
<meta name="color-scheme" content="dark light">

<!-- Light only (no dark support) -->
<meta name="color-scheme" content="light">
<!-- Or omit entirely -->

<!-- Force light on dark mode systems -->
<meta name="color-scheme" content="only light">

<!-- Force dark on light mode systems -->
<meta name="color-scheme" content="only dark">
```

### The relationship with theme-color

| Tag | What it declares | Value type |
|---|---|---|
| `color-scheme` | Which schemes the page supports | scheme keywords (light, dark, both) |
| `theme-color` | Specific chrome color | CSS color value |

Both should be set on dual mode sites.

### Five rules to memorize

1. **color-scheme declares support; theme-color declares chrome color.**
2. **For dual mode sites: `light dark` (or `dark light` for dark first).**
3. **Place in `<head>` early (after charset and viewport).**
4. **Pair with CSS dark mode media queries.**
5. **Use CSS variables for themeable colors (no hard coded backgrounds).**

### Five commands every operator should know

```bash
# 1. Get color-scheme for a URL
curl -s URL | grep -oE 'meta name="color-scheme"[^>]+'

# 2. Check for CSS dark mode
curl -s URL | grep -c "prefers-color-scheme: dark"

# 3. Audit across all Bubbles client sites
color-scheme-audit.sh

# 4. Cross check with theme-color
URL=https://example.com/
echo "color-scheme: $(curl -s "$URL" | grep -oE 'meta name="color-scheme"[^>]+' | head -1)"
echo "theme-color: $(curl -s "$URL" | grep -oE 'meta name="theme-color"[^>]+' | head -1)"

# 5. Apply changes
nginx -t && systemctl reload nginx
systemctl restart fastapi-sidecar
```

### Three end to end tests

```bash
# 1. color-scheme declared
URL=https://example.com/
CS=$(curl -s "$URL" | grep -c 'meta name="color-scheme"')
[ "$CS" -ge "1" ] && echo "OK: color-scheme declared" || echo "FAIL"

# 2. Coordination with theme-color
TC=$(curl -s "$URL" | grep -c 'meta name="theme-color"')
[ "$TC" -ge "1" ] && [ "$CS" -ge "1" ] && echo "OK: both tags present" || echo "FAIL"

# 3. No mismatch (dark CSS without color-scheme)
DARK_CSS=$(curl -s "$URL" | grep -c "prefers-color-scheme: dark")
if [ "$DARK_CSS" -gt "0" ] && [ "$CS" = "0" ]; then
    echo "FAIL: dark mode CSS without color-scheme declaration"
else
    echo "OK: no mismatch"
fi
```

If all three pass AND form controls (date pickers, scrollbars) verified to adapt to scheme AND no FOIC on real device, the color-scheme layer is correctly wired.

---

End of framework-html-meta-color-scheme.md.
