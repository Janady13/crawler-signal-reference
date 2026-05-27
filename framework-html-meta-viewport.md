# framework-html-meta-viewport.md

Comprehensive reference for the HTML `<meta name="viewport">` declaration controlling mobile rendering, zoom behavior, and viewport sizing. Covers every property: `width` (the device width vs fixed pixel mode), `initial-scale` (initial zoom level), `maximum-scale` (zoom upper bound, the WCAG critical setting), `minimum-scale` (zoom lower bound), `user-scalable` (the most accessibility critical setting), `viewport-fit` (notched display handling for iPhone X and similar), and `interactive-widget` (the newer virtual keyboard behavior control). Plus the related CSS `env(safe-area-inset-*)` functions for notched layouts, the mobile usability ranking signal, and the WCAG 1.4.4 Resize Text compliance requirements (critical for SDVOSB federal subcontracting work). Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the third framework in the HTML signal track**, following framework-html-meta-robots.md (indexing control) and framework-html-meta-charset.md (encoding declaration). Companion to the 12 wire layer frameworks.

Audience: humans hand coding mobile responsive sites, AI assistants generating HTML head sections that work correctly on every device class, operators verifying WCAG accessibility compliance for federal subcontracting work, developers building Awwwards caliber sites that handle iPhone notches and Android edge cases (relevant: thataiguy.org Three.js GLSL hero, all Bubbles client mobile experiences), and anyone troubleshooting "site renders desktop layout on phone", "horizontal scroll on mobile", "pinch to zoom disabled", "form fields hidden under iOS keyboard", or "Lighthouse accessibility audit failing on viewport".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Viewport Mental Model (read this first)
5. The width Property (device-width vs fixed pixels)
6. The initial-scale Property (initial zoom level)
7. The maximum-scale Property (the WCAG critical upper bound)
8. The minimum-scale Property (rarely useful)
9. The user-scalable Property (the most accessibility critical setting)
10. The viewport-fit Property (notched display handling)
11. The interactive-widget Property (virtual keyboard control)
12. The Canonical Pattern (recommended default)
13. Mobile First Implications
14. CSS Media Queries Interaction
15. Notch And Safe Area Handling (env safe-area-inset)
16. The Mobile Usability Ranking Signal
17. WCAG 1.4.4 Compliance (critical for SDVOSB federal work)
18. Common Viewport Patterns Per Use Case
19. Asset Class And Use Case Recipes
20. Bubbles Standard Pattern (paste ready)
21. Audit Checklist (50+ items)
22. Common Pitfalls
23. Diagnostic Commands
24. Cross-References

---

## 1. DEFINITION

`<meta name="viewport">` is the HTML directive that controls how mobile browsers size and scale the viewport (the visible area of the web page). Defined informally across browser specifications, formalized in the CSS Device Adaptation specification. Mobile browsers without this directive render pages at a virtual desktop width (typically 980px) and then scale down to fit the device screen, producing the "tiny text on phone" effect that was universal before responsive design.

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

Three structural facts shape how the directive works:

* **Mobile only.** Desktop browsers ignore the viewport meta tag. The directive exists specifically to override mobile browsers' default "render at desktop width, scale to fit" behavior.
* **Comma separated properties in the content attribute.** Each property takes a value: `width=device-width`, `initial-scale=1`, etc.
* **Affects rendering immediately on page load.** The viewport configuration is needed before CSS layout can complete. Place the tag early in `<head>` so the browser sees it before computing layout.

For Bubbles client sites in 2026, the canonical pattern is `<meta name="viewport" content="width=device-width, initial-scale=1">` for standard responsive sites. Add `viewport-fit=cover` for full bleed designs that need to extend under iPhone notches. **Never** set `maximum-scale=1` or `user-scalable=no`: both are WCAG 1.4.4 violations and prevent users with low vision from zooming text.

---

## 2. WHY IT MATTERS

Nine independent pressures push correct viewport configuration from "default behavior" to "actively managed signal" in 2025 and forward.

**Without the viewport tag, mobile renders desktop layout shrunk to phone size.** The default mobile browser behavior is to assume a ~980 pixel wide viewport (the historical desktop minimum) and scale the rendered result down to fit the actual device screen. Text becomes microscopic; users must pinch zoom to read. Conversion rates collapse. Google flags the site as "Not mobile friendly" in Search Console.

**Google's mobile usability is a direct ranking signal.** Per Google's documented ranking factors, mobile usability affects rankings on mobile search results (which is the majority of search traffic in 2026). A site without viewport configuration appears as "Not mobile friendly" in Search Console's Mobile Usability report; rankings on mobile searches degrade.

**WCAG 1.4.4 Resize Text is a Level AA requirement.** Per W3C WCAG 2.1: users must be able to resize text up to 200% without loss of content or functionality. Per WCAG best practice: up to 500%. Disabling zoom via `user-scalable=no` or `maximum-scale=1` (or any value below 2) is a severe accessibility violation. Federal contracts (relevant: Joseph's SDVOSB certification) require Section 508 compliance which maps directly to WCAG 2.1 AA.

**`maximum-scale=1` and `user-scalable=no` are still common copy paste mistakes.** Despite being well documented WCAG violations, these settings still appear in templates, CMS defaults, and example code from various sources. Lighthouse, axe DevTools, WAVE, and other accessibility scanners flag this immediately. The fix is to remove the offending properties.

**Notched displays (iPhone X, 11, 12, 13, 14, 15, 16) need `viewport-fit=cover` for full bleed designs.** Without this property, full bleed backgrounds and edge to edge content respect a "safe area" that excludes the notch and home indicator regions. With `viewport-fit=cover`, content can extend under the notch (paired with CSS `env(safe-area-inset-*)` to ensure interactive elements remain in the safe area).

**Virtual keyboards change the viewport.** When a user taps a form input on mobile, the on screen keyboard appears, reducing the visible viewport area by 30 to 50%. The `interactive-widget` property (added in 2023) controls how this resize is reported to JavaScript and CSS. Default is `resizes-visual` (keyboard takes space; visual viewport shrinks). Alternatives are `resizes-content` (the layout reflows) or `overlays-content` (keyboard floats over content without resizing). Choosing the wrong mode produces form fields hidden under the keyboard.

**Awwwards caliber and immersive designs need precise viewport control.** The thataiguy.org Three.js GLSL hero, the WebGL canvas on art portfolio pages, the parallax scrolling on real estate sites: all depend on the viewport being correctly sized at first render. Incorrect viewport leads to canvas sizing bugs, scroll glitches, and broken animations.

**Print and email exports ignore the viewport tag.** PDF generation, email rendering, and printed pages all use document layout independent of the viewport. Sites that depend entirely on viewport scaling for layout break when printed. The fix is CSS that works at both viewport-controlled mobile sizes AND fixed sizes.

**Cost of getting it wrong.** Misconfigured viewport produces measurable user experience and ranking damage. Real examples:

* Bubbles client launched a new site. Forgot the viewport meta tag. Mobile bounce rate jumped from 30% to 78%. Google Search Console flagged "Not mobile friendly" within 4 days. Three weeks of mobile traffic loss before discovery. Fix: add the canonical `<meta name="viewport" content="width=device-width, initial-scale=1">`.
* Federal subcontractor site (SDVOSB context) had `user-scalable=no` in the viewport tag. Section 508 compliance audit failed. Contract renewal blocked pending remediation. Fix: remove `user-scalable=no`.
* iPhone X user complained that the menu button appeared cut off in the notch. The page had a fixed position header with no `env(safe-area-inset-top)`. Fix: add `viewport-fit=cover` to viewport, add `padding-top: env(safe-area-inset-top)` to fixed header.
* Form on a mobile site had the submit button hidden under the keyboard when users typed in any field. Default `interactive-widget=resizes-visual` was fine for layout but JavaScript was using `window.innerHeight` (which did not change with the keyboard). Fix: use `visualViewport.height` API or set `interactive-widget=resizes-content` for legacy compatibility.
* Real estate site looked perfect on iPad but had horizontal scroll on iPhone. The page had a fixed 1280px wide container. Even with viewport meta tag, the container forced overflow. Fix: use `max-width: 100%` and responsive CSS.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The viewport meta tag plus its full operational context:

1. **Every property**: width, initial-scale, maximum-scale, minimum-scale, user-scalable, viewport-fit, interactive-widget.
2. **The WCAG 1.4.4 requirements**: what to never set, what to always allow.
3. **Notch and safe area handling**: viewport-fit=cover plus CSS env() functions.
4. **The mobile usability ranking signal**: Google Search Console reporting and remediation.
5. **The mobile first methodology**: CSS that works correctly when viewport is configured.

Section 17 covers WCAG compliance in depth (critical for Joseph's federal subcontracting work). Sections 20 to 23 are the standard reference, audit, and diagnostic tooling.

---

## 4. THE VIEWPORT MENTAL MODEL (READ THIS FIRST)

A mobile browser fetches an HTML page. Without a viewport meta tag:

```
Mobile browser default behavior (without viewport meta):
  1. Assume viewport width is ~980 pixels (historical desktop width).
  2. Render the page at 980 pixel width.
  3. Scale the rendered result to fit the actual device screen (say 390 pixels).
  4. Result: layout looks "right" but at 40% scale; text is microscopic.
```

With the canonical viewport meta tag:

```
Mobile browser with <meta name="viewport" content="width=device-width, initial-scale=1">:
  1. Set viewport width to the device's CSS pixel width (e.g., 390 on iPhone 14).
  2. Render the page at that width.
  3. Display at 100% (no shrinking).
  4. Result: layout is responsive to actual device width; text is full size.
```

The decision tree for viewport settings:

```
Is this a mobile responsive site?
   |
   |---> YES: width=device-width, initial-scale=1 (the canonical default)
   |
   |---> NO: do not include the viewport tag (let mobile do its default behavior)
        |
        v
==================== ADDITIONAL OPTIONS ====================
        |
        v
Does the design extend full bleed under iPhone notch?
   |
   |---> YES: add viewport-fit=cover
   |        AND use CSS env(safe-area-inset-*) for interactive elements
   |
   |---> NO: leave viewport-fit default (auto)
        |
        v
Does the page have forms with on screen keyboard interaction?
   |
   |---> YES: consider interactive-widget=resizes-content if layout
   |          should reflow under keyboard
   |
   |---> NO: default (resizes-visual) is fine
        |
        v
==================== ACCESSIBILITY GUARDRAILS ====================
        |
        v
NEVER include any of these (WCAG 1.4.4 violations):
   - user-scalable=no
   - maximum-scale=1 (or any value less than 2)
   - minimum-scale=1 (locks initial zoom)
```

Six rules govern the system:

1. **Use the canonical pattern as default**: `<meta name="viewport" content="width=device-width, initial-scale=1">`.
2. **Never disable zoom**: WCAG 1.4.4 requires users can zoom to at least 200%.
3. **Place the tag early in head**: after `<meta charset>`, before other meta tags.
4. **Add viewport-fit=cover for full bleed designs**: handles iPhone notches.
5. **Mobile first CSS pairs with the viewport tag**: small screen styles first, then media queries up.
6. **Test on real devices**: Chrome DevTools simulation is not sufficient.

A correctly configured site has the canonical viewport tag, never restricts zoom, handles notches with viewport-fit plus env() functions, and pairs the viewport configuration with mobile first responsive CSS.

---

## 5. THE WIDTH PROPERTY (DEVICE-WIDTH VS FIXED PIXELS)

The `width` property defines the viewport's logical width.

### 5.1 width=device-width (Canonical)

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

The viewport width is set to the device's CSS pixel width (also called "ideal width"). This is the recommended default for all responsive sites.

CSS pixel widths for common devices:

| Device | CSS pixel width |
|---|---|
| iPhone SE (3rd gen) | 375px |
| iPhone 14/15/16 | 390px |
| iPhone 14 Pro Max | 430px |
| iPad (9th gen) | 810px |
| iPad Pro 12.9" | 1024px |
| Samsung Galaxy S22 | 360px |
| Google Pixel 8 | 412px |
| Tablet (generic) | 768px |

CSS `@media` queries use these CSS pixel widths, not the device's physical pixel resolution.

### 5.2 width=N (Fixed Pixel Width)

```html
<meta name="viewport" content="width=1024">
```

The viewport is treated as exactly N pixels wide. The mobile browser renders the page at that width and then scales to fit the device screen.

**Use cases:**

* Legacy sites with fixed width layouts that cannot be easily made responsive.
* Specific embedded contexts (kiosks, internal tools with known device dimensions).

**Disadvantages:**

* Loses responsive behavior across device sizes.
* Generally produces a "shrink to fit" experience, which is what device-width is meant to avoid.

**Not recommended** for new Bubbles client sites.

### 5.3 height Property

A `height` property exists but is rarely used:

```html
<meta name="viewport" content="width=device-width, height=device-height, initial-scale=1">
```

Setting `height=device-height` is generally unnecessary; browsers default to a sensible height. The property is reserved for legacy compatibility. Skip it.

### 5.4 The Verification

```bash
# Find viewport meta tag
curl -s https://example.com/ | grep -oE 'meta name="viewport"[^>]+'
# Expected: meta name="viewport" content="width=device-width, initial-scale=1"
```

Browser DevTools:

1. Open the page on a mobile device or in Chrome DevTools device mode.
2. Check that the viewport width matches the device width (no scaling).
3. Inspect `<html>` element CSS: `clientWidth` should match the device's CSS pixel width.

---

## 6. THE INITIAL-SCALE PROPERTY (INITIAL ZOOM LEVEL)

The `initial-scale` property defines the initial zoom level when the page loads.

### 6.1 initial-scale=1 (Canonical)

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

The page loads at 100% zoom. Combined with `width=device-width`, content renders at the natural size on the device. **This is the canonical default for all responsive sites.**

### 6.2 initial-scale Other Values

* `initial-scale=2`: page loads at 200% zoom. Content is large; user can zoom out.
* `initial-scale=0.5`: page loads at 50% zoom. Content is small; user can zoom in.
* `initial-scale=0.86`: a quirky default some older Android browsers used.

**Not recommended** to use values other than 1 for new sites.

### 6.3 The Relationship Between width And initial-scale

When both `width=device-width` and `initial-scale=1` are set, they are consistent. If they conflict:

```html
<meta name="viewport" content="width=320, initial-scale=1">
```

On a 390px device, this is inconsistent: width=320 but initial-scale=1 wants to render at device width. Browsers typically use the larger value (390 in this case), but behavior varies.

For consistency: always pair `width=device-width` with `initial-scale=1`.

### 6.4 The Common Sense Rule

If you ever find yourself wanting to set `initial-scale` to anything other than 1, reconsider. The site likely has a layout problem that should be fixed with responsive CSS, not by adjusting the initial zoom.

---

## 7. THE MAXIMUM-SCALE PROPERTY (THE WCAG CRITICAL UPPER BOUND)

The `maximum-scale` property defines the upper limit of user zoom.

### 7.1 The WCAG Rule

**Never set `maximum-scale` to a value less than 2.**

Per W3C WCAG 2.1 Success Criterion 1.4.4 (Resize Text, Level AA): users must be able to resize text up to 200% without loss of content or functionality. A `maximum-scale` value less than 2 violates this requirement.

Per WCAG ACT Rule b4f0c3: "the attribute value does not have a maximum-scale property with a value less than 2."

```html
<!-- WRONG: WCAG violation -->
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">

<!-- ACCEPTABLE: allows 200% zoom -->
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=2">

<!-- BEST: allows 500% zoom (no maximum-scale set; browser default is 10x) -->
<meta name="viewport" content="width=device-width, initial-scale=1">
```

### 7.2 Why People Set maximum-scale

Common misconceptions:

* "We don't want users to zoom and break the layout." If zoom breaks the layout, the layout is broken. Fix the layout.
* "It looks weird at 300% zoom on some pages." Fix the CSS to handle reflow at high zoom levels.
* "iOS auto zooms when users tap input fields." Fix this with `font-size: 16px;` minimum on form inputs (iOS does not auto zoom inputs with 16px+ font size).

### 7.3 The Bubbles Rule

For ALL Bubbles client sites:

* **Do not set `maximum-scale`** in the viewport meta tag.
* If a `maximum-scale` is present from a legacy template or CMS default, **remove it**.

### 7.4 The Federal Subcontracting Implication

For Joseph's SDVOSB certified work serving federal subcontractors:

* Section 508 of the Rehabilitation Act mandates federal sites meet WCAG 2.1 AA.
* `maximum-scale=1` is a documented Section 508 audit failure.
* Federal contract renewals can be blocked by accessibility audit findings.
* Verify viewport configuration as part of every federal subcontract handoff.

---

## 8. THE MINIMUM-SCALE PROPERTY (RARELY USEFUL)

The `minimum-scale` property defines the lower limit of user zoom.

### 8.1 The Default

Without explicit `minimum-scale`, browsers allow zoom out to roughly 0.25x (the browser default).

```html
<meta name="viewport" content="width=device-width, initial-scale=1, minimum-scale=0.5">
```

Users can zoom out to 50% but no further.

### 8.2 When To Use

Rarely. Almost no use cases warrant setting minimum-scale:

* Map applications where zooming out too far is unhelpful.
* Visualization dashboards where overly zoomed out renders are useless.

**Not recommended** for typical Bubbles client sites.

### 8.3 The Accessibility Consideration

Unlike `maximum-scale`, restricting `minimum-scale` does not violate WCAG (zoom in is the accessibility need, not zoom out). However, restricting `minimum-scale=1` prevents users from zooming out, which can frustrate users on small screens who want to see more context.

Convention: leave `minimum-scale` unset.

---

## 9. THE USER-SCALABLE PROPERTY (THE MOST ACCESSIBILITY CRITICAL SETTING)

The `user-scalable` property controls whether users can zoom at all.

### 9.1 The WCAG Rule

**Never set `user-scalable=no`.**

Per W3C WCAG ACT Rule b4f0c3: "the attribute value does not have a user-scalable property with a value of no."

Per accessibility researchers and MDN documentation: disabling zoom prevents users with low vision from reading content. WCAG 1.4.4 explicitly requires zoom capability.

```html
<!-- WRONG: WCAG violation, accessibility failure -->
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">

<!-- WRONG: equivalent to user-scalable=no in old browsers -->
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=0">

<!-- CORRECT: zoom enabled (default; equivalent to omitting the property) -->
<meta name="viewport" content="width=device-width, initial-scale=1">

<!-- EXPLICIT BUT REDUNDANT: same as default -->
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=yes">
```

### 9.2 Why People Set user-scalable=no

Common (wrong) reasons:

* "We're building an app-like experience and zoom shouldn't be possible." Apps must still meet accessibility standards. PWA progressive disclosure requirements include zoom.
* "Pinch zoom interferes with our custom touch gestures." Use touch event handling that doesn't conflict with browser zoom (most custom gestures and zoom can coexist).
* "Form inputs trigger automatic zoom on iOS." Fix with `font-size: 16px` on inputs, not by disabling zoom.

### 9.3 The Bubbles Rule

For ALL Bubbles client sites:

* **Do not set `user-scalable=no`** in the viewport meta tag.
* If `user-scalable=no` is present from a legacy template, CMS default, or copy paste from "app-like" examples, **remove it**.

### 9.4 The iOS Auto Zoom On Inputs

A frequent reason developers reach for `user-scalable=no`: iOS Safari auto zooms when a user taps a form input with font-size less than 16px. The fix is CSS:

```css
/* Prevent iOS auto zoom on form input focus */
input[type="text"],
input[type="email"],
input[type="number"],
input[type="password"],
input[type="search"],
input[type="tel"],
input[type="url"],
textarea,
select {
    font-size: 16px;
}
```

With this CSS, iOS does not auto zoom on input focus, and zoom remains available for users who need it.

### 9.5 The Lighthouse Audit

Google Lighthouse explicitly checks for `user-scalable=no` and `maximum-scale<2`:

```
Accessibility audit:
  ✗ [accessibility-2.21] [user-scalable="no"] is used in the <meta name="viewport"> element, or the [maximum-scale] attribute is less than 5.
```

Fixing this is required for an "Accessibility" score of 100 in Lighthouse.

---

## 10. THE VIEWPORT-FIT PROPERTY (NOTCHED DISPLAY HANDLING)

The `viewport-fit` property controls how the viewport relates to a device's safe area (the region not obscured by notches, home indicators, or rounded screen corners).

### 10.1 The Three Values

* **`viewport-fit=auto`** (default): viewport excludes the notch area; content is constrained to the safe area. Backgrounds do not extend under the notch.
* **`viewport-fit=contain`**: viewport is the largest rectangle fully inscribed in the display. Equivalent to auto for most cases.
* **`viewport-fit=cover`**: viewport covers the entire display, INCLUDING under notches. Content can extend full bleed.

### 10.2 The viewport-fit=cover Pattern

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

When this is set, the viewport extends to the edges of the device screen. Content (especially backgrounds) can extend behind the notch, home indicator, and rounded corners.

**The cost:** interactive elements positioned at the edges may be obscured by the notch. The fix is CSS:

```css
/* Pair viewport-fit=cover with safe area padding */
.fixed-header {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    padding-top: env(safe-area-inset-top);
    padding-left: env(safe-area-inset-left);
    padding-right: env(safe-area-inset-right);
}

.bottom-nav {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    padding-bottom: env(safe-area-inset-bottom);
}

body {
    /* Allow background to extend full bleed */
    margin: 0;
}
```

### 10.3 The env() Functions

CSS `env()` function returns the size of the safe area inset on each edge:

* **`env(safe-area-inset-top)`**: distance from viewport top to the safe area top (e.g., 44px on iPhone 14 notch).
* **`env(safe-area-inset-bottom)`**: distance from viewport bottom to safe area bottom (e.g., 34px for iPhone home indicator).
* **`env(safe-area-inset-left)`** and **`env(safe-area-inset-right)`**: for landscape orientation with notch on the side.

When `viewport-fit=cover` is NOT set, all `env(safe-area-inset-*)` values are 0 (the viewport is already inside the safe area).

### 10.4 The Use Case Decision

**Use `viewport-fit=cover`** for:

* Full bleed hero sections that should extend to all device edges.
* Immersive experiences (Three.js canvases, games, full screen video).
* Modern designs with edge to edge backgrounds.

**Do NOT use `viewport-fit=cover`** for:

* Traditional document like pages (articles, forms, tables).
* Sites with content that should stay within safe areas anyway.

### 10.5 The Verification

Test on actual notched devices (iPhone X+). Chrome DevTools simulation does not perfectly replicate notch behavior. Use:

* Real iPhone with notch.
* iOS Simulator (Xcode) with notched device profile.
* Safari Web Inspector connected to real device.

```css
/* Visual debugging: show the safe area boundaries */
body::before {
    content: 'safe area';
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    border: 2px solid red;
    border-top-width: env(safe-area-inset-top);
    border-bottom-width: env(safe-area-inset-bottom);
    border-left-width: env(safe-area-inset-left);
    border-right-width: env(safe-area-inset-right);
    pointer-events: none;
    z-index: 999999;
}
```

The red border shows where the safe area boundaries are.

---

## 11. THE INTERACTIVE-WIDGET PROPERTY (VIRTUAL KEYBOARD CONTROL)

The `interactive-widget` property (added in 2023, becoming more widely supported in 2025) controls how virtual keyboards affect the viewport.

### 11.1 The Three Values

* **`interactive-widget=resizes-visual`** (default): when keyboard appears, only the visual viewport shrinks. Layout viewport stays the same. CSS does not respond to keyboard appearance.
* **`interactive-widget=resizes-content`**: when keyboard appears, layout viewport also shrinks. CSS responds (e.g., media queries can detect smaller height). Layout reflows.
* **`interactive-widget=overlays-content`**: when keyboard appears, neither viewport changes. Keyboard floats over content. Content does not reflow.

### 11.2 The Use Case Decision

**Use `resizes-visual`** (default) for:

* Standard responsive sites.
* Pages where keyboard interaction is occasional.

**Use `resizes-content`** for:

* Form heavy applications where layout should adapt when keyboard appears.
* Sites where CSS height calculations should reflect available space.

**Use `overlays-content`** for:

* Apps with custom keyboard handling.
* Full screen experiences where keyboard should not affect layout (rare).

### 11.3 The Form Field Visibility Problem

The most common issue: form fields hidden under the on screen keyboard. The fix is usually:

```css
/* Ensure form elements scroll into view when focused */
input,
textarea,
select {
    scroll-margin-bottom: 100px;
}
```

```javascript
// Programmatic scroll on focus
document.querySelectorAll('input, textarea').forEach(el => {
    el.addEventListener('focus', () => {
        setTimeout(() => {
            el.scrollIntoView({ behavior: 'smooth', block: 'center' });
        }, 300); // wait for keyboard animation
    });
});
```

### 11.4 The Visual Viewport API

For precise JavaScript handling of keyboard state, use the Visual Viewport API:

```javascript
window.visualViewport.addEventListener('resize', () => {
    const heightDiff = window.innerHeight - window.visualViewport.height;
    if (heightDiff > 100) {
        // Keyboard is likely open
        document.body.classList.add('keyboard-open');
    } else {
        document.body.classList.remove('keyboard-open');
    }
});
```

CSS can respond:

```css
body.keyboard-open .bottom-nav {
    display: none; /* Hide bottom nav when keyboard is open */
}
```

### 11.5 The Bubbles Default

For typical Bubbles client sites: **omit `interactive-widget`** (use the default `resizes-visual`). Add it only when forms behave incorrectly with the default behavior, after exhausting CSS based fixes.

---

## 12. THE CANONICAL PATTERN (RECOMMENDED DEFAULT)

For 95% of Bubbles client sites, the recommended viewport meta tag is:

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

Two properties:

* **`width=device-width`**: viewport matches device's CSS pixel width.
* **`initial-scale=1`**: initial zoom is 100%.

Nothing else. No maximum-scale, no user-scalable, no minimum-scale, no viewport-fit (unless needed), no interactive-widget (unless needed).

### 12.1 The Extended Patterns

**For full bleed designs (extends under notch):**

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

**For form heavy mobile apps where keyboard should reflow layout:**

```html
<meta name="viewport" content="width=device-width, initial-scale=1, interactive-widget=resizes-content">
```

**For full bleed mobile apps with custom keyboard handling:**

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, interactive-widget=overlays-content">
```

### 12.2 The Forbidden Patterns

```html
<!-- WCAG 1.4.4 violations - NEVER USE -->
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">

<!-- Discouraged: fixed pixel widths -->
<meta name="viewport" content="width=1024">

<!-- Discouraged: non default initial scale without good reason -->
<meta name="viewport" content="width=device-width, initial-scale=0.5">
```

### 12.3 The Placement

Place the viewport meta tag in `<head>` immediately after `<meta charset="utf-8">`:

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">                                                  <!-- FIRST -->
    <meta name="viewport" content="width=device-width, initial-scale=1">    <!-- SECOND -->
    <title>...</title>
    <!-- everything else follows -->
</head>
```

This order ensures both critical tags are within the first 1024 bytes (the charset rule) and the viewport configuration is set before layout computation.

---

## 13. MOBILE FIRST IMPLICATIONS

The viewport meta tag enables mobile responsive behavior, but the CSS must be written to take advantage. The "mobile first" methodology is the standard companion approach.

### 13.1 The Mobile First Pattern

Write base CSS for the smallest screens first. Use `@media (min-width: ...)` to add styles for larger screens.

```css
/* Base styles: mobile (small screens) */
body {
    font-size: 16px;
    padding: 1rem;
}

.container {
    width: 100%;
    max-width: 100%;
}

.nav {
    display: flex;
    flex-direction: column;
}

/* Tablet and up */
@media (min-width: 768px) {
    body {
        padding: 2rem;
    }

    .nav {
        flex-direction: row;
    }
}

/* Desktop and up */
@media (min-width: 1024px) {
    .container {
        max-width: 1200px;
        margin: 0 auto;
    }
}
```

### 13.2 The Anti Pattern

Desktop first CSS (the opposite of mobile first):

```css
/* Desktop assumed */
body { padding: 4rem; }
.nav { display: flex; flex-direction: row; }

/* Override for tablet */
@media (max-width: 1024px) {
    body { padding: 2rem; }
}

/* Override for mobile */
@media (max-width: 768px) {
    body { padding: 1rem; }
    .nav { flex-direction: column; }
}
```

This works but cascades poorly. Mobile devices download all the desktop CSS even though they only use mobile styles. Mobile first inverts this: mobile gets the base CSS; larger screens get progressively more.

### 13.3 The Touch Target Sizes

WCAG 2.5.5 (Level AAA) recommends touch targets of at least 44 by 44 CSS pixels. Mobile first CSS naturally accommodates this:

```css
button,
a.cta,
.nav-link {
    min-height: 44px;
    min-width: 44px;
    padding: 12px 16px;
}
```

### 13.4 The Font Size Floor

To prevent iOS auto zoom on input focus AND for general readability:

```css
input,
textarea,
select {
    font-size: 16px;  /* iOS will not auto zoom if font-size >= 16px */
}

body {
    font-size: 16px;
    line-height: 1.5;
}
```

### 13.5 The Container Width Rule

Container elements should default to fluid widths:

```css
/* Good: fluid containers */
.container {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 1rem;
}

/* Bad: fixed widths force horizontal scroll on mobile */
.container {
    width: 1200px;
}
```

---

## 14. CSS MEDIA QUERIES INTERACTION

The viewport meta tag and CSS media queries work together. Understanding the relationship prevents subtle bugs.

### 14.1 The Min Width Approach (Mobile First)

```css
/* Default: phones */
.layout { display: block; }

/* Tablet: 768px and up */
@media (min-width: 768px) {
    .layout { display: grid; grid-template-columns: 1fr 1fr; }
}

/* Desktop: 1024px and up */
@media (min-width: 1024px) {
    .layout { grid-template-columns: 1fr 2fr 1fr; }
}
```

Each breakpoint adds styles for larger screens.

### 14.2 The Common Breakpoints

| Breakpoint | Devices |
|---|---|
| 320px (no media query) | Old/small phones |
| 375px | Standard phones (iPhone SE, etc) |
| 414px | Large phones (iPhone Plus, etc) |
| 768px | Tablets, small laptops |
| 1024px | Desktops, large tablets in landscape |
| 1280px | Standard desktops |
| 1920px | Large desktops |

Per Tailwind CSS conventions:

```css
/* sm */
@media (min-width: 640px) { ... }

/* md */
@media (min-width: 768px) { ... }

/* lg */
@media (min-width: 1024px) { ... }

/* xl */
@media (min-width: 1280px) { ... }

/* 2xl */
@media (min-width: 1536px) { ... }
```

### 14.3 The Orientation Detection

```css
/* Landscape only */
@media (orientation: landscape) { ... }

/* Portrait only */
@media (orientation: portrait) { ... }
```

Useful for mobile and tablet layouts that should adapt to rotation.

### 14.4 The High DPI Detection

```css
/* Retina/HiDPI displays */
@media (-webkit-min-device-pixel-ratio: 2),
       (min-resolution: 192dpi) {
    /* High DPI styles */
}
```

For images and visual assets that should be sharper on high DPI screens.

### 14.5 The Reduced Motion

```css
/* Respect user's motion preferences */
@media (prefers-reduced-motion: reduce) {
    .animated {
        animation: none;
        transition: none;
    }
}
```

Pairs with viewport configuration for full accessibility compliance.

### 14.6 The Color Scheme

```css
/* Light mode (default) */
:root {
    --bg: white;
    --fg: black;
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
    :root {
        --bg: #111;
        --fg: #eee;
    }
}
```

### 14.7 The Viewport Units

CSS viewport units (`vh`, `vw`, `dvh`, `lvh`, `svh`) interact with the viewport meta tag. With `width=device-width`:

* `100vw` = viewport width (typically full device width).
* `100vh` = viewport height (full device height).
* `100dvh` = dynamic viewport height (adjusts for browser chrome).
* `100lvh` = largest viewport height (without browser chrome).
* `100svh` = smallest viewport height (with browser chrome).

For full screen mobile sections:

```css
.hero {
    height: 100svh;  /* Smallest viewport, ensures content fits even with browser chrome */
}
```

---

## 15. NOTCH AND SAFE AREA HANDLING (ENV SAFE-AREA-INSET)

Modern phones (iPhone X+, many Android phones) have notches, home indicators, or rounded corners that intrude into the rectangular display area.

### 15.1 The Two Part Pattern

1. **Add `viewport-fit=cover`** to the viewport meta tag.
2. **Use CSS `env(safe-area-inset-*)`** to position interactive elements in the safe area.

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

```css
/* Fixed header that respects notch */
.site-header {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    padding-top: env(safe-area-inset-top, 0);
    padding-left: env(safe-area-inset-left, 0);
    padding-right: env(safe-area-inset-right, 0);
    background: white;
    z-index: 100;
}

/* Bottom navigation that respects home indicator */
.bottom-nav {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    padding-bottom: env(safe-area-inset-bottom, 0);
    padding-left: env(safe-area-inset-left, 0);
    padding-right: env(safe-area-inset-right, 0);
    background: white;
    z-index: 100;
}

/* Main content respects safe area */
main {
    padding-top: calc(60px + env(safe-area-inset-top));
    padding-bottom: calc(60px + env(safe-area-inset-bottom));
}
```

### 15.2 The Fallback Values

`env()` supports a fallback for browsers that don't recognize the variable:

```css
padding-top: env(safe-area-inset-top, 0px);
```

If `safe-area-inset-top` is not defined (older browser, no notch), use 0px.

### 15.3 The Constant Function (Legacy)

Older iOS versions used `constant()` instead of `env()`:

```css
/* iOS 11.0 to 11.2 used constant(); 11.2+ uses env() */
padding-top: constant(safe-area-inset-top);
padding-top: env(safe-area-inset-top);
```

For 2026, `env()` is fine; the `constant()` syntax is legacy.

### 15.4 The Full Bleed Hero Pattern

For sections that should extend edge to edge (under notch and home indicator):

```html
<section class="hero-full-bleed">
    <div class="hero-content">
        <h1>Welcome</h1>
        <button>Get Started</button>
    </div>
</section>
```

```css
.hero-full-bleed {
    position: relative;
    width: 100%;
    height: 100vh;  /* or 100svh */
    background: url('hero.jpg') center / cover;
    /* No padding for safe areas - background extends edge to edge */
}

.hero-content {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    /* Inner content respects safe area for interactive elements */
    padding: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left);
    max-width: 90%;
    text-align: center;
}
```

Background extends full bleed; text and button stay in safe area.

---

## 16. THE MOBILE USABILITY RANKING SIGNAL

Google uses mobile usability as a ranking factor. Sites that fail mobile usability checks rank worse on mobile searches.

### 16.1 The Mobile Friendly Test

Google's Mobile Friendly Test (search.google.com/test/mobile-friendly) evaluates:

* Viewport configured (`<meta name="viewport">` present).
* Text readable without zooming (font-size considerations).
* Content sized to viewport (no horizontal scroll).
* Touch targets adequately spaced.

Sites that fail any of these are flagged as "Not mobile friendly" in Search Console.

### 16.2 The Search Console Report

Google Search Console > Mobile Usability shows issues per URL:

* "Viewport not configured": missing or invalid viewport meta tag.
* "Viewport not set to 'device-width'": using `width=NNN` (fixed pixel width).
* "Content wider than screen": horizontal scroll on mobile.
* "Text too small to read": font sizes below ~12px.
* "Clickable elements too close together": touch targets less than 8px apart.

Each issue lists affected URLs and remediation guidance.

### 16.3 The Remediation Path

For "Viewport not configured":

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

For "Content wider than screen":

```css
body {
    overflow-x: hidden;
}

.container {
    width: 100%;
    max-width: 100%;
}

img, video {
    max-width: 100%;
    height: auto;
}
```

For "Text too small to read":

```css
body {
    font-size: 16px;
}
```

For "Clickable elements too close together":

```css
button,
a.button,
.nav-link {
    min-height: 44px;
    padding: 12px 16px;
}

button + button,
.nav-link + .nav-link {
    margin-left: 8px;
}
```

### 16.4 The Recovery Timeline

After fixing mobile usability issues:

1. Deploy fix.
2. Submit URL for recrawl in GSC URL Inspection tool.
3. Wait 1 to 7 days for Google to recrawl.
4. Mobile usability report updates within 1 to 2 weeks.
5. Mobile rankings improve within 2 to 4 weeks of remediation.

---

## 17. WCAG 1.4.4 COMPLIANCE (CRITICAL FOR SDVOSB FEDERAL WORK)

The viewport meta tag is one of the most commonly cited WCAG violations on audited sites. For Joseph's SDVOSB certified work serving federal subcontractors, this is critical.

### 17.1 The WCAG 1.4.4 Success Criterion

**WCAG 2.1 Success Criterion 1.4.4 (Resize Text, Level AA):**

> "Except for captions and images of text, text can be resized without assistive technology up to 200 percent without loss of content or functionality."

### 17.2 The Three Forbidden Configurations

1. `user-scalable=no` (or `user-scalable=0`): zoom disabled entirely.
2. `maximum-scale=1` (or any value below 2): zoom limited to less than 200%.
3. The combination of both (defense in depth wrong: doubly forbidden).

### 17.3 The Section 508 Mapping

Section 508 of the Rehabilitation Act (29 U.S.C. § 794d) requires federal information technology to be accessible. The Section 508 Standards (revised 2018) directly reference WCAG 2.0 Level A and AA as the technical standard. WCAG 2.1 Level AA (which includes 1.4.4) is the de facto current bar.

Sites that violate WCAG 1.4.4 fail Section 508 audits. Federal subcontractors with failing audits can:

* Lose contract renewals.
* Face remediation requirements before payment.
* Be removed from approved vendor lists.

### 17.4 The Audit Workflow

For every Bubbles client site (especially SDVOSB context):

```bash
# Find any sites with WCAG violating viewport
for site in /var/www/sites/*/index.html; do
    VIEWPORT=$(grep -oE 'meta name="viewport"[^>]+' "$site" | head -1)
    if [[ "$VIEWPORT" =~ "user-scalable=no" ]] || \
       [[ "$VIEWPORT" =~ "user-scalable=0" ]] || \
       [[ "$VIEWPORT" =~ "maximum-scale=1" ]]; then
        echo "WCAG VIOLATION: $site"
        echo "  $VIEWPORT"
    fi
done
```

### 17.5 The Lighthouse Verification

```bash
# Run Lighthouse with accessibility audit only
npx lighthouse https://example.com/ \
    --only-categories=accessibility \
    --quiet \
    --output=json \
    --output-path=/tmp/lighthouse-a11y.json

# Check viewport specific audit
python3 -c "
import json
with open('/tmp/lighthouse-a11y.json') as f:
    data = json.load(f)
viewport_audit = data['audits'].get('meta-viewport', {})
print(f'Score: {viewport_audit.get(\"score\")}')
print(f'Title: {viewport_audit.get(\"title\")}')
print(f'Description: {viewport_audit.get(\"description\")}')
"
```

A passing site shows `score: 1` for the `meta-viewport` audit.

### 17.6 The Public Statement For SDVOSB Subcontractor Clients

When delivering a site to a federal subcontractor client, include accessibility statement:

> Site meets WCAG 2.1 Level AA conformance requirements, including Success Criterion 1.4.4 (Resize Text). Users can zoom to at least 200% via standard browser controls. Viewport meta tag configured as `width=device-width, initial-scale=1` with no zoom restrictions.

This statement, combined with verified Lighthouse 100/100 accessibility score and manual axe DevTools audit, supports the client's Section 508 compliance posture.

### 17.7 The Continuous Enforcement

Add to deployment pipeline:

```bash
#!/bin/bash
# /usr/local/bin/wcag-viewport-check.sh
# Pre deploy check for WCAG viewport violations

FAILED=0

for site in /var/www/sites/*/index.html; do
    VIEWPORT=$(grep -oE 'meta name="viewport"[^>]+' "$site" | head -1)
    if [[ "$VIEWPORT" =~ user-scalable=(no|0) ]] || \
       [[ "$VIEWPORT" =~ maximum-scale=(0\.|1$|1\.0|1\.[0-9]+) ]]; then
        echo "FAIL: $site"
        echo "       $VIEWPORT"
        FAILED=1
    fi
done

if [ $FAILED -eq 1 ]; then
    echo ""
    echo "WCAG 1.4.4 violations found. Remediate before deploying."
    exit 1
fi

echo "OK: All sites pass WCAG viewport check."
```

Run as part of every deployment.

---

## 18. COMMON VIEWPORT PATTERNS PER USE CASE

The canonical pattern serves 95% of cases. The remaining 5% use one of these patterns.

### 18.1 Standard Responsive Site

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

Use for: marketing sites, blogs, e commerce, most Bubbles client sites.

### 18.2 Full Bleed Immersive Design

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

Use for: landing pages with full screen hero, portfolio sites, immersive experiences. Pair with CSS `env(safe-area-inset-*)` for interactive elements.

### 18.3 Form Heavy Mobile App

```html
<meta name="viewport" content="width=device-width, initial-scale=1, interactive-widget=resizes-content">
```

Use for: mobile first applications with extensive form interaction (booking, registration, account management).

### 18.4 Full Bleed Mobile App With Overlay Keyboard

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, interactive-widget=overlays-content">
```

Use for: PWA-like experiences with custom keyboard handling.

### 18.5 PWA Manifest Aware

For Progressive Web Apps:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="theme-color" content="#3B7434">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<link rel="manifest" href="/manifest.json">
```

### 18.6 Awwwards Caliber Hero With Three.js

For sites like thataiguy.org with WebGL canvases:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

```css
.webgl-canvas {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100svh;  /* smallest viewport height; ensures canvas doesn't shift when browser chrome changes */
    z-index: 0;
}

.content {
    position: relative;
    z-index: 1;
    padding: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left);
}
```

### 18.7 Email Templates (Different Rules)

Email clients have their own viewport rules. For HTML emails:

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
<!-- Email specific: -->
<meta name="format-detection" content="telephone=no">
<meta name="x-apple-disable-message-reformatting">
```

But email rendering varies wildly; the viewport meta tag is honored by some clients (Apple Mail, Gmail mobile) and ignored by others (Outlook).

### 18.8 Documentation Sites

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

Use the canonical pattern. Documentation should be highly accessible; no restrictions on zoom.

---

## 19. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 19.1 Canonical Bubbles HTML head

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Page Title | ThatDeveloperGuy</title>
    <meta name="description" content="...">

    <!-- everything else follows -->
</head>
```

### 19.2 Full bleed with safe area handling

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
    <title>...</title>

    <style>
        body {
            margin: 0;
            padding: 0;
        }

        .full-bleed-hero {
            position: relative;
            width: 100%;
            min-height: 100svh;
            background: linear-gradient(135deg, #3B7434 0%, #6B46C1 100%);
            color: white;
        }

        .full-bleed-hero .content {
            padding-top: calc(2rem + env(safe-area-inset-top));
            padding-bottom: calc(2rem + env(safe-area-inset-bottom));
            padding-left: calc(2rem + env(safe-area-inset-left));
            padding-right: calc(2rem + env(safe-area-inset-right));
            max-width: 800px;
            margin: 0 auto;
        }

        .fixed-header {
            position: fixed;
            top: 0;
            left: 0;
            right: 0;
            padding-top: env(safe-area-inset-top);
            background: rgba(0, 0, 0, 0.8);
            z-index: 100;
        }
    </style>
</head>
<body>
    <section class="full-bleed-hero">
        <div class="content">
            <h1>Crafted by ThatDeveloperGuy.com</h1>
        </div>
    </section>
</body>
</html>
```

### 19.3 Form prevention of iOS auto zoom

```html
<style>
    input,
    textarea,
    select {
        font-size: 16px;  /* iOS does not auto zoom inputs with font-size >= 16px */
    }
</style>

<form>
    <input type="email" name="email" placeholder="Email">
    <input type="tel" name="phone" placeholder="Phone">
    <button type="submit">Send</button>
</form>
```

### 19.4 Touch target sizing for WCAG compliance

```css
/* Minimum touch target: 44 x 44 CSS pixels */
button,
a.button,
input[type="submit"],
input[type="button"],
.touchable {
    min-height: 44px;
    min-width: 44px;
    padding: 12px 16px;
}

/* Touch target spacing: at least 8px between adjacent targets */
.nav-link + .nav-link,
button + button {
    margin-left: 8px;
}
```

### 19.5 Mobile first responsive base

```css
/* Mobile first base */
* {
    box-sizing: border-box;
}

html {
    font-size: 16px;
    line-height: 1.5;
}

body {
    margin: 0;
    padding: 0;
    font-family: system-ui, -apple-system, sans-serif;
    color: #333;
    background: #fff;
}

.container {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 1rem;
}

img, video {
    max-width: 100%;
    height: auto;
}

/* Tablet and up */
@media (min-width: 768px) {
    .container {
        padding: 0 2rem;
    }
}

/* Desktop and up */
@media (min-width: 1024px) {
    body {
        font-size: 18px;
    }
}
```

### 19.6 The WCAG safe viewport (verification snippet)

```bash
#!/bin/bash
# Verify viewport meta tag is WCAG compliant
URL=$1
VIEWPORT=$(curl -s "$URL" | grep -oE 'meta name="viewport"[^>]+' | head -1)

echo "Viewport tag: $VIEWPORT"

if [[ "$VIEWPORT" =~ user-scalable=(no|0) ]]; then
    echo "VIOLATION: user-scalable disabled (WCAG 1.4.4 failure)"
    exit 1
fi

if [[ "$VIEWPORT" =~ maximum-scale=([01]\.?[0-9]*) ]]; then
    SCALE="${BASH_REMATCH[1]}"
    if (( $(echo "$SCALE < 2" | bc -l) )); then
        echo "VIOLATION: maximum-scale $SCALE less than 2 (WCAG 1.4.4 failure)"
        exit 1
    fi
fi

echo "PASS: viewport tag is WCAG compliant"
```

### 19.7 Reduced motion media query

```css
.animated-element {
    animation: slideIn 0.5s ease-out;
}

@media (prefers-reduced-motion: reduce) {
    .animated-element {
        animation: none;
    }

    * {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
    }
}
```

### 19.8 Dark mode aware

```css
:root {
    --bg-primary: #ffffff;
    --bg-secondary: #f5f5f5;
    --fg-primary: #1a1a1a;
    --fg-secondary: #6b7280;
    --accent: #3B7434;
}

@media (prefers-color-scheme: dark) {
    :root {
        --bg-primary: #1a1a1a;
        --bg-secondary: #2a2a2a;
        --fg-primary: #f5f5f5;
        --fg-secondary: #9ca3af;
        --accent: #6B46C1;
    }
}

body {
    background: var(--bg-primary);
    color: var(--fg-primary);
}
```

---

## 20. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles viewport configuration with all supporting CSS.

### 20.1 The Base Template

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Page Title | ThatDeveloperGuy</title>
    <meta name="description" content="...">

    <!-- For Awwwards caliber sites with full bleed: -->
    <!-- <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover"> -->

    <!-- Theme color (browser chrome) -->
    <meta name="theme-color" content="#3B7434">

    <!-- Stylesheets -->
    <link rel="stylesheet" href="/styles.css">
</head>
<body>
    <!-- content -->
    <footer>
        <p>Crafted by ThatDeveloperGuy.com</p>
    </footer>
</body>
</html>
```

### 20.2 The Standard CSS Reset Plus Base

```css
/* /var/www/sites/example.com/styles.css */

/* Box sizing reset */
*,
*::before,
*::after {
    box-sizing: border-box;
}

/* Document base */
html {
    font-size: 16px;
    -webkit-text-size-adjust: 100%;
    text-size-adjust: 100%;
}

body {
    margin: 0;
    padding: 0;
    font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
    font-size: 16px;
    line-height: 1.5;
    color: #1a1a1a;
    background: #fff;
}

/* Prevent iOS auto zoom on form inputs */
input,
textarea,
select {
    font-size: 16px;
}

/* Minimum touch target sizes for accessibility */
button,
a.button,
input[type="submit"],
input[type="button"] {
    min-height: 44px;
    min-width: 44px;
    padding: 12px 16px;
    cursor: pointer;
}

/* Container with mobile first padding */
.container {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 1rem;
}

@media (min-width: 768px) {
    .container {
        padding: 0 2rem;
    }
}

/* Responsive images */
img,
video,
iframe {
    max-width: 100%;
    height: auto;
}

/* Reduced motion support */
@media (prefers-reduced-motion: reduce) {
    *,
    *::before,
    *::after {
        animation-duration: 0.01ms !important;
        animation-iteration-count: 1 !important;
        transition-duration: 0.01ms !important;
    }
}
```

### 20.3 The Audit Script

```bash
#!/bin/bash
# /usr/local/bin/viewport-audit.sh

URL=$1

echo "=== Viewport audit for $URL ==="
echo ""

VIEWPORT=$(curl -s "$URL" | head -c 4000 | grep -oE 'meta name="viewport"[^>]+' | head -1)

if [ -z "$VIEWPORT" ]; then
    echo "FAIL: no viewport meta tag found"
    exit 1
fi

echo "Viewport tag: $VIEWPORT"
echo ""

# Check for canonical pattern
if echo "$VIEWPORT" | grep -q "width=device-width" && \
   echo "$VIEWPORT" | grep -q "initial-scale=1"; then
    echo "OK: width=device-width and initial-scale=1 present"
else
    echo "WARN: viewport does not match canonical pattern"
fi

# Check for WCAG violations
WCAG_VIOLATIONS=0

if [[ "$VIEWPORT" =~ user-scalable=(no|0) ]]; then
    echo "FAIL: user-scalable disabled (WCAG 1.4.4 violation)"
    WCAG_VIOLATIONS=$((WCAG_VIOLATIONS + 1))
fi

if [[ "$VIEWPORT" =~ maximum-scale=([0-9.]+) ]]; then
    SCALE="${BASH_REMATCH[1]}"
    if (( $(echo "$SCALE < 2" | bc -l 2>/dev/null) )); then
        echo "FAIL: maximum-scale $SCALE less than 2 (WCAG 1.4.4 violation)"
        WCAG_VIOLATIONS=$((WCAG_VIOLATIONS + 1))
    fi
fi

if [ $WCAG_VIOLATIONS -eq 0 ]; then
    echo "OK: no WCAG 1.4.4 violations"
fi

echo ""
echo "=== End audit ==="

if [ $WCAG_VIOLATIONS -gt 0 ]; then
    exit 1
fi
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/viewport-audit.sh
viewport-audit.sh https://example.com/
```

After any template change:

```bash
# Reload nginx if static files (most Bubbles client sites)
nginx -t && systemctl reload nginx

# Or restart FastAPI sidecar for templated content
systemctl restart fastapi-sidecar

# Verify
viewport-audit.sh https://example.com/
```

---

## 21. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### Core viewport tag

1. [ ] `<meta name="viewport">` tag present in every HTML page.
2. [ ] Tag placed in `<head>` after `<meta charset>`.
3. [ ] Tag within first 1024 bytes of document (within charset's window).
4. [ ] `width=device-width` set.
5. [ ] `initial-scale=1` set.

### WCAG 1.4.4 compliance (critical for SDVOSB federal work)

6. [ ] **NO `user-scalable=no` anywhere in any template.**
7. [ ] **NO `user-scalable=0` anywhere in any template.**
8. [ ] **NO `maximum-scale=1` (or below 2) anywhere in any template.**
9. [ ] Verified across all templates, not just homepage.
10. [ ] CI pre deploy check rejects WCAG violations.

### Notch and safe area (for full bleed designs)

11. [ ] `viewport-fit=cover` added if design requires full bleed.
12. [ ] CSS uses `env(safe-area-inset-*)` for interactive element positioning.
13. [ ] Tested on actual iPhone X+ device (not just simulator).
14. [ ] Landscape orientation also handled (safe-area-inset-left, safe-area-inset-right).
15. [ ] Fallback values provided in env() functions.

### Mobile first CSS

16. [ ] Base CSS targets smallest screens.
17. [ ] Media queries use `min-width` (not max-width).
18. [ ] `box-sizing: border-box` set globally.
19. [ ] No fixed pixel widths on containers.
20. [ ] Images use `max-width: 100%; height: auto;`.

### Form behavior

21. [ ] Form inputs have `font-size: 16px` minimum (prevents iOS auto zoom).
22. [ ] Form fields visible when keyboard appears (scroll behavior verified).
23. [ ] Submit buttons accessible during keyboard interaction.
24. [ ] `accept-charset="utf-8"` on forms (covered in framework-html-meta-charset.md).

### Touch targets (WCAG 2.5.5)

25. [ ] All interactive elements at least 44 by 44 CSS pixels.
26. [ ] Adjacent touch targets have at least 8px spacing.
27. [ ] Touch targets visible without zoom.

### Text readability (WCAG 1.4.4)

28. [ ] Body text at least 16px (1rem) by default.
29. [ ] Text remains readable when zoomed to 200%.
30. [ ] No horizontal scroll at 200% zoom.

### Mobile usability (Google ranking signal)

31. [ ] Google Search Console > Mobile Usability shows no errors.
32. [ ] Mobile Friendly Test (search.google.com/test/mobile-friendly) passes.
33. [ ] No "Content wider than screen" errors.
34. [ ] No "Text too small to read" errors.
35. [ ] No "Clickable elements too close together" errors.

### Cross device testing

36. [ ] Tested on iPhone (notched, latest iOS).
37. [ ] Tested on Android (Pixel or similar, latest Chrome).
38. [ ] Tested on iPad (portrait and landscape).
39. [ ] Tested on Android tablet.
40. [ ] Tested with browser zoom at 200%, 400%.
41. [ ] Tested with system level large text size enabled.

### Tooling and automation

42. [ ] Lighthouse Mobile audit score 95+.
43. [ ] Lighthouse Accessibility audit score 100.
44. [ ] axe DevTools audit passes for viewport related rules.
45. [ ] `viewport-audit.sh` script available at `/usr/local/bin/`.
46. [ ] Pre deploy check runs viewport audit.

### Documentation and process

47. [ ] Viewport configuration documented in site template.
48. [ ] WCAG compliance statement in accessibility documentation.
49. [ ] Federal client deliverable includes accessibility statement.
50. [ ] Quarterly audit reviews viewport configuration.

A site that passes all 50 has correctly configured viewport for mobile rendering, WCAG accessibility compliance, and Google mobile usability ranking signals.

---

## 22. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: `user-scalable=no` in the viewport tag.**
Symptom: Lighthouse, axe DevTools flag accessibility violation; users with low vision cannot zoom.
Why it breaks: WCAG 1.4.4 violation. Federal Section 508 audit failure.
Fix: remove `user-scalable=no` from the viewport meta tag.

**Pitfall 2: `maximum-scale=1` to "prevent zoom breakage".**
Symptom: same as above; WCAG 1.4.4 violation.
Why it breaks: WCAG requires zoom to at least 200%; 1 is below 2.
Fix: remove `maximum-scale` entirely (browser defaults to ~10x); or set `maximum-scale=2` minimum.

**Pitfall 3: Missing viewport meta tag.**
Symptom: site renders desktop layout shrunk on mobile; tiny text.
Why it breaks: mobile browser assumes ~980px desktop viewport by default.
Fix: add `<meta name="viewport" content="width=device-width, initial-scale=1">`.

**Pitfall 4: Viewport tag placed after large content in `<head>`.**
Symptom: viewport configuration applied late; some browsers ignore.
Why it breaks: viewport tag should be early in head for layout calculation.
Fix: place viewport meta tag immediately after `<meta charset>`.

**Pitfall 5: iOS auto zoom on form input focus.**
Symptom: tapping a form field causes the entire page to zoom in awkwardly.
Why it breaks: iOS auto zooms inputs with font-size less than 16px.
Fix: set `font-size: 16px` on input, textarea, select elements via CSS.

**Pitfall 6: Form fields hidden under keyboard on mobile.**
Symptom: user can't see what they're typing because keyboard covers the field.
Why it breaks: form not scrolling to keep focused input visible.
Fix: use `scroll-margin-bottom` CSS or programmatic scroll on focus.

**Pitfall 7: Horizontal scroll on mobile despite viewport tag.**
Symptom: viewport tag present but page has horizontal scroll on small screens.
Why it breaks: fixed width container or overflowing element wider than viewport.
Fix: audit containers for fixed pixel widths; ensure `max-width: 100%`.

**Pitfall 8: Content cut off by notch on iPhone X+.**
Symptom: header content or buttons obscured by notch.
Why it breaks: design extends to edges but no `env(safe-area-inset-*)` padding.
Fix: add `viewport-fit=cover` AND CSS env() padding on edge positioned elements.

**Pitfall 9: Three.js or WebGL canvas wrong size on mobile.**
Symptom: WebGL canvas appears stretched, distorted, or wrong aspect ratio.
Why it breaks: canvas resize not handled correctly; relies on viewport dimensions.
Fix: handle canvas resize via JavaScript using `visualViewport` API; use `100svh` not `100vh`.

**Pitfall 10: Page jumps when mobile keyboard appears.**
Symptom: visible content suddenly shifts when user taps an input.
Why it breaks: `100vh` height changes when keyboard appears.
Fix: use `100dvh` (dynamic viewport height) or `100svh` (smallest viewport height).

**Pitfall 11: Email client renders broken layout.**
Symptom: email looks fine in browser but breaks in Gmail/Outlook.
Why it breaks: email clients have different viewport handling; meta tag may be ignored.
Fix: use table based layout for emails; rely on inline styles; test in actual email clients.

**Pitfall 12: PWA install prompt shows wrong meta theme color.**
Symptom: PWA installs with default browser color instead of brand color.
Why it breaks: `<meta name="theme-color">` missing or incorrect.
Fix: add `<meta name="theme-color" content="#XXXXXX">` matching brand.

**Pitfall 13: User zoom causes text overlap.**
Symptom: at 200% zoom, text columns overlap or content becomes unreadable.
Why it breaks: layout uses fixed positioning that doesn't reflow.
Fix: use flexible layouts (flexbox, grid); test at 200% and 400% zoom.

**Pitfall 14: Dark mode meta theme color not updating.**
Symptom: browser chrome stays light blue in dark mode.
Why it breaks: only one `theme-color` declared.
Fix:

```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
```

**Pitfall 15: Lighthouse passes but real device fails.**
Symptom: Lighthouse score 100, but actual iPhone displays differently.
Why it breaks: Lighthouse simulates a specific device profile; real device behavior varies.
Fix: test on real devices (iPhone, Android, iPad); do not rely solely on Lighthouse simulation.

---

## 23. DIAGNOSTIC COMMANDS

Reference of every command useful for viewport investigation.

### Inspect viewport meta tag

```bash
# Extract viewport tag from a URL
curl -s https://example.com/ | grep -oE 'meta name="viewport"[^>]+' | head -1

# Check viewport tag is within first 1024 bytes
curl -s https://example.com/ | head -c 1024 | grep -c "meta name=\"viewport\""
# Expected: 1
```

### Check for WCAG violations

```bash
# Find viewport meta tag and check for violations
URL=$1
VIEWPORT=$(curl -s "$URL" | grep -oE 'meta name="viewport"[^>]+' | head -1)
echo "$VIEWPORT"

# Check user-scalable
if [[ "$VIEWPORT" =~ user-scalable=(no|0) ]]; then
    echo "VIOLATION: user-scalable disabled"
fi

# Check maximum-scale
if [[ "$VIEWPORT" =~ maximum-scale=([0-9.]+) ]]; then
    SCALE="${BASH_REMATCH[1]}"
    if (( $(echo "$SCALE < 2" | bc -l 2>/dev/null) )); then
        echo "VIOLATION: maximum-scale $SCALE < 2"
    fi
fi
```

### Bulk audit all client sites

```bash
# Audit every Bubbles client site
for site_dir in /var/www/sites/*/; do
    INDEX="$site_dir/index.html"
    if [ -f "$INDEX" ]; then
        VIEWPORT=$(grep -oE 'meta name="viewport"[^>]+' "$INDEX" | head -1)
        SITE=$(basename "$site_dir")

        if [ -z "$VIEWPORT" ]; then
            echo "MISSING: $SITE has no viewport tag"
        elif [[ "$VIEWPORT" =~ user-scalable=(no|0) ]]; then
            echo "WCAG VIOLATION: $SITE - $VIEWPORT"
        elif [[ "$VIEWPORT" =~ maximum-scale=([01]\.?[0-9]*) ]]; then
            echo "WCAG VIOLATION: $SITE - maximum-scale below 2"
        else
            echo "OK: $SITE"
        fi
    fi
done
```

### Test mobile usability via Lighthouse

```bash
# Install Lighthouse if not present
npm install -g lighthouse

# Run Lighthouse with mobile preset
lighthouse https://example.com/ \
    --preset=desktop \
    --only-categories=accessibility,seo \
    --output=html \
    --output-path=/tmp/lighthouse-mobile.html \
    --chrome-flags="--headless"

# Check viewport specific audit
lighthouse https://example.com/ \
    --only-audits=viewport,meta-viewport \
    --output=json \
    --output-path=/tmp/lighthouse-viewport.json \
    --quiet

python3 -c "
import json
with open('/tmp/lighthouse-viewport.json') as f:
    data = json.load(f)
for audit_id, audit in data.get('audits', {}).items():
    if 'viewport' in audit_id.lower():
        print(f'{audit_id}: {audit.get(\"score\")} - {audit.get(\"title\")}')
"
```

### Test mobile usability via Google's API

```bash
# Mobile Friendly Test API (deprecated as of 2025; use Search Console URL Inspection instead)
# Manual: search.google.com/test/mobile-friendly

# Or use Google PageSpeed Insights API (free tier)
API_KEY="YOUR_KEY"
URL="https://example.com/"
curl -s "https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=$URL&strategy=mobile&key=$API_KEY" | python3 -m json.tool | grep -A 3 "viewport"
```

### Browser DevTools verification

In Chrome DevTools:

1. Open the page.
2. Toggle device toolbar (Cmd/Ctrl + Shift + M).
3. Select a mobile device profile.
4. Check that the page renders at device width, not shrunk.
5. Use Lighthouse panel to run accessibility audit.
6. Use Inspect tool to verify `<meta name="viewport">` in DOM.

### Real device testing

```bash
# iPad/iPhone testing via Safari Web Inspector
# 1. Enable Web Inspector on iOS: Settings > Safari > Advanced > Web Inspector ON
# 2. Connect iOS device to Mac via USB
# 3. Open Safari on Mac: Develop menu > device name > page name
# 4. Inspect viewport via Console:
#    document.documentElement.clientWidth
#    document.documentElement.clientHeight
#    window.innerWidth
#    window.innerHeight
```

### Bulk fix WCAG violations

```bash
# Find and replace user-scalable=no
find /var/www/sites/ -name "*.html" -type f -exec \
    sed -i 's/, *user-scalable=no//g; s/, *user-scalable=0//g; s/user-scalable=no, *//g; s/user-scalable=0, *//g' {} \;

# Find and replace maximum-scale=1
find /var/www/sites/ -name "*.html" -type f -exec \
    sed -i 's/, *maximum-scale=1\([^.]\)/\1/g; s/, *maximum-scale=1\.0\b//g' {} \;

# Verify
for site_dir in /var/www/sites/*/; do
    grep -l "user-scalable=no\|maximum-scale=1\b" "$site_dir"/index.html 2>/dev/null
done
# Should produce no output
```

### After viewport changes

```bash
# Reload nginx if templates served as static files
nginx -t && systemctl reload nginx

# Or restart FastAPI sidecar if templates rendered server side
systemctl restart fastapi-sidecar

# Verify the change
viewport-audit.sh https://example.com/
```

---

## 24. CROSS-REFERENCES

* [framework-html-meta-charset.md](framework-html-meta-charset.md): place the viewport meta tag immediately after `<meta charset="utf-8">` in `<head>`; both must be within the first 1024 bytes.
* [framework-html-meta-robots.md](framework-html-meta-robots.md): the first framework in the HTML signal track; controls indexing.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): X-Robots-Tag HTTP header complements meta robots; not directly related to viewport but part of the same head section signal layer.
* [framework-http-performance-headers.md](framework-http-performance-headers.md) Section 5 (Alt-Svc): performance signals that affect mobile rendering speed.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Mobile usability is a documented ranking factor.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including viewport placement in templates.
* W3C CSS Device Adaptation: https://www.w3.org/TR/css-device-adapt/
* MDN viewport meta tag: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name/viewport
* WCAG 2.1 Success Criterion 1.4.4 (Resize Text): https://www.w3.org/WAI/WCAG21/Understanding/resize-text.html
* WCAG ACT Rule b4f0c3 (Meta viewport allows for zoom): https://www.w3.org/WAI/standards-guidelines/act/rules/b4f0c3/
* Section 508 Standards: https://www.access-board.gov/ict/
* Google Mobile Friendly Test: https://search.google.com/test/mobile-friendly
* Lighthouse accessibility audits: https://web.dev/articles/accessibility-audits-with-lighthouse

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The canonical pattern

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

Place immediately after `<meta charset="utf-8">` in `<head>`.

### The forbidden patterns (WCAG 1.4.4 violations)

```html
<!-- NEVER USE -->
<meta name="viewport" content="..., user-scalable=no">
<meta name="viewport" content="..., maximum-scale=1">
<meta name="viewport" content="..., user-scalable=0">
```

### The full bleed pattern (for designs that extend under notch)

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

Pair with CSS:

```css
.header {
    padding-top: env(safe-area-inset-top);
    padding-left: env(safe-area-inset-left);
    padding-right: env(safe-area-inset-right);
}

.footer {
    padding-bottom: env(safe-area-inset-bottom);
}
```

### Five rules to memorize

1. **The canonical pattern is `width=device-width, initial-scale=1`. That's it for 95% of sites.**
2. **NEVER set `user-scalable=no` or `maximum-scale` below 2 (WCAG 1.4.4).**
3. **Add `viewport-fit=cover` only when design extends under notches.**
4. **Form inputs need `font-size: 16px` minimum (prevents iOS auto zoom).**
5. **Test on real devices, not just Chrome DevTools.**

### Five commands every operator should know

```bash
# 1. Check viewport tag
curl -s URL | grep -oE 'meta name="viewport"[^>]+'

# 2. Check for WCAG violations
curl -s URL | grep -oE 'meta name="viewport"[^>]+' | grep -E "user-scalable=(no|0)|maximum-scale=1\b"

# 3. Lighthouse mobile audit
lighthouse URL --only-categories=accessibility,seo --quiet --output=html --output-path=/tmp/lh.html

# 4. Bulk audit all sites
for site in /var/www/sites/*/index.html; do echo "$(basename $(dirname $site)): $(grep -oE 'meta name="viewport"[^>]+' $site | head -1)"; done

# 5. Apply changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Viewport tag present and canonical
URL=https://example.com/
VP=$(curl -s "$URL" | grep -oE 'meta name="viewport"[^>]+' | head -1)
[[ "$VP" =~ "width=device-width" ]] && [[ "$VP" =~ "initial-scale=1" ]] && echo "OK" || echo "FAIL"

# 2. No WCAG violations
[[ "$VP" =~ user-scalable=(no|0) ]] && echo "FAIL: WCAG violation" || echo "OK: WCAG"

# 3. Google Mobile Friendly Test passes
# (Manual: search.google.com/test/mobile-friendly with URL)
# Or use viewport-audit.sh
viewport-audit.sh "$URL"
```

If all three pass AND no horizontal scroll on real mobile devices AND Lighthouse accessibility score is 100, the viewport layer is correctly wired.

---

End of framework-html-meta-viewport.md.
