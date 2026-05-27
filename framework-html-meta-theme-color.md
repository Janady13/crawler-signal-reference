# framework-html-meta-theme-color.md

Comprehensive reference for HTML `<meta name="theme-color">`, the mobile browser chrome and operating system UI color declaration. Covers the standard pattern, the critical 2025-2026 development (iOS 26 Safari dropped support for the meta tag, now uses body/html background color or a fixed element workaround), the light/dark mode media query pattern with `media="(prefers-color-scheme: ...)"`, the Apple restricted color palette (Safari does not honor pure black, some reds, yellows, greens), the PWA manifest integration (`theme_color` key in site.webmanifest), the related iOS specific tags (`apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style`), the Windows specific tags (`msapplication-TileColor`, `msapplication-navbutton-color`), the `<meta name="color-scheme">` CSS meta tag distinction, the brand color strategy (ThatDeveloperGuy purple/green per Joseph's standing brand rule; TCB Fight Factory purple/black), and the Bubbles per client color decisions. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the ninth framework in the HTML signal track**, following meta robots, charset, viewport, description, keywords, author, generator, and copyright. Companion to the 12 wire layer frameworks.

Audience: humans configuring mobile browser chrome to match brand colors, AI assistants generating HTML head sections with appropriate theme color handling for Chrome AND iOS 26 Safari, designers implementing dark mode that affects browser chrome, PWA developers reconciling meta tag with manifest.json theme_color, and anyone troubleshooting "theme color works on Chrome but not iOS Safari", "Safari shows the wrong chrome color despite my meta tag", "browser chrome turns black in dark mode", or "the theme color flashes on page load".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Theme Color Mental Model (read this first)
5. The iOS 26 Safari Change (CRITICAL UPDATE)
6. The Standard Chrome And Other Browser Pattern
7. The Light And Dark Mode Media Query Pattern
8. The Apple Restricted Color Palette
9. The PWA Manifest Integration
10. iOS Specific Meta Tags (apple-mobile-web-app-*)
11. Windows Specific Meta Tags (msapplication-*)
12. The color-scheme CSS Meta Tag (distinct from theme-color)
13. Brand Color Strategy
14. The Bubbles Per Client Color Decisions
15. The iOS 26 Workaround Patterns
16. Asset Class And Use Case Recipes
17. Bubbles Standard Pattern (paste ready)
18. Audit Checklist
19. Common Pitfalls
20. Diagnostic Commands
21. Cross-References

---

## 1. DEFINITION

`<meta name="theme-color">` is the HTML directive declaring a color for mobile browser chrome (address bar, status bar area, tab indicator) and operating system UI elements when the page is loaded as a PWA. Defined in W3C specifications and HTML living standard.

```html
<head>
    <meta name="theme-color" content="#6B46C1">
</head>
```

Three structural facts shape how the directive works in 2026:

* **Browser support varies critically by vendor.** Chrome Android, Chrome Desktop, Firefox (limited), Brave, Edge support theme-color. iOS Safari historically supported it; **iOS 26 Safari dropped support** and now uses body/html background color instead.
* **It supports media query variants.** `media="(prefers-color-scheme: light)"` and `media="(prefers-color-scheme: dark)"` provide automatic dark mode handling. Available since October 2022 across major browsers (the HTMLMetaElement.media property is baseline widely available).
* **Apple has a restricted color palette.** Safari (when it did support theme-color, and via background color now) does not honor pure black (#000000), some reds, yellows, and greens. This is a documented Apple choice.

For Bubbles client sites in 2026, the convention is: include `<meta name="theme-color">` with the brand primary color, include dark mode variant via media query, plus apply matching `body { background-color }` CSS to cover iOS 26 Safari behavior. Joseph's ThatDeveloperGuy brand colors are purple and green; TCB Fight Factory brand colors are purple and black.

---

## 2. WHY IT MATTERS

Eight independent considerations push correct theme color usage from "nice to have" to "actively managed signal" in 2025 and forward.

**It is the most visible polish on mobile sites.** When a user opens a site on Chrome Android, the entire top of the screen (address bar plus status bar area) takes the theme-color. Sites with no theme-color get default gray; sites with theme-color matching the brand look intentional and polished.

**iOS 26 Safari behavior changed in 2025-2026.** Per Apple's iOS 26 release notes and subsequent developer reports, Safari 26 stopped honoring the theme-color meta tag. Apple now derives the browser chrome color from the body/html background color or from a fixed element at the top of the viewport. This breaks sites that relied on theme-color for Safari chrome coloring.

**The dark mode media query pattern is essential for modern sites.** Without the media variant, dark mode users see brand colors against a dark system context, which can produce poor contrast. The pattern is:

```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
```

This produces appropriate chrome in both modes.

**Apple's restricted color palette blocks some brand colors.** Per documented Safari behavior: pure black (#000000), certain reds, yellows, and greens are not honored. The Roger Lee Safari color checker tool maps out which colors work. For brands using these prohibited colors, fallback strategy is needed.

**PWA manifest theme_color is the cross device fallback.** When a site is installed as a PWA, the `theme_color` value in the web manifest controls the splash screen and chrome. Aligning meta theme-color with manifest theme_color provides consistent appearance across launches.

**iOS PWA specific tags still matter.** `apple-mobile-web-app-status-bar-style` controls the iOS status bar appearance in PWA mode. `apple-mobile-web-app-capable` enables PWA mode. These are iOS specific and predate the broader theme-color spec.

**Windows tile color is for legacy tile pinning.** `msapplication-TileColor` and similar Windows specific tags affect how the site appears when pinned to a Windows tile. Less critical in 2026 (tile pinning is rarely used) but still part of complete browser chrome coverage.

**Color choice impacts accessibility.** Chrome rendering for theme-color includes text overlay (URL bar text, status bar icons). Some colors produce poor contrast with these UI elements. Testing is required.

**Cost of getting it wrong.** Incorrect theme color handling produces measurable polish and brand consistency damage. Real examples:

* Bubbles client launched with `<meta name="theme-color" content="#000000">`. Chrome rendered it correctly. iOS Safari (pre 26) refused to render it because pure black is on Apple's prohibited list. The site had black chrome in Chrome and gray in Safari. Fix: use #0a0a0a (very dark gray, accepted by Safari) instead of pure black.
* Site launched in early 2026 worked perfectly on Chrome but Safari 26 ignored the theme-color tag entirely. Investigation revealed the iOS 26 behavior change. Fix: set `body { background-color: #6B46C1 }` to match the theme-color, allowing Safari 26 to pick up the color from body background.
* Real estate client had dark theme-color (#1a1a1a) without dark mode variant. Users on dark mode systems saw double-dark (chrome and system both dark), making URL nearly unreadable. Fix: add light theme-color variant via media query.
* TCB Fight Factory had `<meta name="theme-color" content="#FF0000">` for an accent moment in their brand. Safari refused (prohibited red shade). Brand colors are purple and black; red was used incorrectly. Fix: switch to brand purple #6B46C1 which Safari accepts.
* WordPress site (pre Bubbles migration) had two conflicting theme-color tags from Yoast and a different plugin. Browser used the first one; designer expected the second. Fix: removed both during migration; added single canonical theme-color.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The meta theme-color tag plus its full operational context:

1. **The iOS 26 critical change**: what happened, what to do.
2. **The standard pattern**: for Chrome and other supporting browsers.
3. **The dark mode pattern**: media query variants.
4. **The Apple restricted color palette**: which colors don't work.
5. **The PWA manifest integration**: theme_color key.
6. **The related iOS and Windows tags**: full browser chrome coverage.
7. **The color-scheme CSS meta tag distinction**: different purpose.
8. **The Bubbles brand colors**: ThatDeveloperGuy and TCB conventions.

Section 5 is the critical 2026 update for iOS 26 Safari. Section 15 is the workaround pattern.

---

## 4. THE THEME COLOR MENTAL MODEL (READ THIS FIRST)

A browser opens your page. It checks for theme-color metadata to color the browser chrome:

```
Browser checks for theme color signal:
   |
   v
==================== PER BROWSER BEHAVIOR ====================
   |
   |---> Chrome Android: reads <meta name="theme-color">. Honors it.
   |     Picks media query variant matching system color scheme.
   |     Applies to address bar AND status bar.
   |
   |---> Chrome Desktop: reads <meta name="theme-color">. Limited honor.
   |     Affects tab color in some configurations.
   |
   |---> iOS 26 Safari (2025+): IGNORES <meta name="theme-color">.
   |     Uses body/html background color OR fixed element near top of viewport.
   |
   |---> iOS pre-26 Safari: read <meta name="theme-color"> with restrictions.
   |     Apple's prohibited color list applied.
   |     Only worked if "Show color in compact tab bar" enabled in Safari prefs.
   |
   |---> Firefox: limited support; treats theme-color informally.
   |
   |---> Brave, Edge: typically follows Chrome behavior.
   |
   |---> PWA mode (any OS): manifest.json theme_color preferred.
   |     Meta tag supplements but manifest is canonical.

==================== THE DECISION TREE ====================
   |
   v
For each Bubbles client site:
   |
   |---> 1. Define brand primary color.
   |
   |---> 2. Add <meta name="theme-color" content="#brand">.
   |
   |---> 3. Add dark mode variant if site supports dark theme.
   |
   |---> 4. Set body { background-color: #brand } in CSS (for iOS 26).
   |
   |---> 5. Add manifest.json theme_color if PWA capable.
   |
   |---> 6. Add iOS specific tags if PWA install enabled.
   |
   |---> 7. Verify against Apple's prohibited color list.
   |
   |---> 8. Test on real Chrome Android AND real iOS 26 Safari.
```

Six rules govern the system:

1. **Include the standard tag for Chrome and most browsers.**
2. **Include the dark mode media variant for dark mode support.**
3. **Match body background color in CSS for iOS 26 Safari coverage.**
4. **Check Apple's prohibited color list (no pure black, some reds/yellows/greens).**
5. **Align with PWA manifest.json theme_color if applicable.**
6. **Test on multiple real devices, not just simulators.**

A correctly configured Bubbles client site has a working theme-color tag for Chrome AND body background CSS matching for iOS 26 Safari AND a dark mode variant, with PWA manifest alignment if installable.

---

## 5. THE IOS 26 SAFARI CHANGE (CRITICAL UPDATE)

In 2025, with iOS 26 (the "Safari 26" version), Apple changed how Safari determines the browser chrome color. This is a breaking change for sites that relied on theme-color.

### 5.1 The Old Behavior (Pre iOS 26)

Safari read `<meta name="theme-color">` and applied it to the browser chrome (the area at top of screen showing URL bar, plus the status bar area on notched devices). Apple's prohibited color list applied; Safari preferences had to enable "Show color in compact tab bar" for the effect to be visible.

### 5.2 The New Behavior (iOS 26 and forward)

Safari 26 ignores the meta theme-color tag entirely. The browser chrome color is now determined by:

1. **The background color of the element at the top of the viewport.** As the user scrolls, the chrome color may update to match.
2. **Falls back to system default** if no background color is detected at the top.

This means:

```html
<!-- This used to work in Safari; now ignored in Safari 26 -->
<meta name="theme-color" content="#6B46C1">

<!-- This is what Safari 26 looks at -->
<body style="background-color: #6B46C1;">
    <!-- ... content ... -->
</body>
```

The change is documented across:
- Apple Developer Forums (thread 801239 from 2026).
- Reddit r/webdev community discussions.
- CSS Tricks updated coverage.
- Ben Frain's iOS 26 Safari analysis (November 2025).
- grooovinger.com (February 2026).

### 5.3 The Workaround Pattern

For sites that need consistent chrome color across Chrome and iOS 26 Safari:

```html
<head>
    <!-- Standard theme-color for Chrome and others -->
    <meta name="theme-color" content="#6B46C1">

    <!-- Dark mode variant -->
    <meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
</head>
<body style="background-color: #6B46C1;">
    <!-- The body background-color is what iOS 26 Safari uses -->
    <!-- It must match the theme-color above -->

    <main>
        <!-- Inner content can have any background; iOS 26 picks color from top of viewport -->
    </main>
</body>
```

Or with CSS file:

```css
/* /var/www/sites/example.com/styles.css */

body {
    background-color: #6B46C1;
    /* This is what iOS 26 Safari uses for chrome color */
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #1a1a1a;
    }
}

main {
    background-color: white;
    /* Main content background can differ; iOS 26 picks from top of viewport */
}
```

### 5.4 The Fixed Element Trick

For sites where the body background color cannot match the desired chrome color (e.g., the main content area should be white), use a fixed positioned element at the very top:

```html
<head>
    <meta name="theme-color" content="#6B46C1">
</head>
<body>
    <!-- Fixed element provides theme color for iOS 26 Safari -->
    <div class="ios-26-theme-color" aria-hidden="true"></div>

    <!-- Actual content -->
    <main>
        <!-- ... -->
    </main>
</body>

<style>
.ios-26-theme-color {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 1px;
    background-color: #6B46C1;
    z-index: -1;
    pointer-events: none;
}
</style>
```

Caveat: per Ben Frain's analysis, the fixed element approach can break when the page is reloaded with the user scrolled away from the top. The body background approach is more reliable.

### 5.5 The Bubbles Recommendation

For Bubbles client sites in 2026:

1. **Always set body background-color in CSS** matching the desired theme-color.
2. **Always include the standard meta theme-color** for Chrome and other browsers.
3. **Always include the dark mode media variant**.
4. **Document the iOS 26 behavior** in client documentation so the client understands the workaround.

This three layer approach maximizes coverage across browsers.

### 5.6 The Future Direction

Apple has not (as of early 2026) restored meta theme-color support. The body background color approach appears to be Apple's intended direction. Future iOS releases may further refine this; check Apple Developer documentation when launching new sites.

The PWA manifest theme_color does NOT replace the meta tag in this regard. For Safari 26, the manifest theme_color affects PWA mode only; browsing mode uses body background.

---

## 6. THE STANDARD CHROME AND OTHER BROWSER PATTERN

For Chrome Android, Chrome Desktop, Firefox, Brave, Edge, and other supporting browsers, the standard meta theme-color works as expected.

### 6.1 The Basic Pattern

```html
<head>
    <meta name="theme-color" content="#6B46C1">
</head>
```

The color value is any valid CSS color:

```html
<!-- Hex -->
<meta name="theme-color" content="#6B46C1">

<!-- RGB -->
<meta name="theme-color" content="rgb(107, 70, 193)">

<!-- HSL -->
<meta name="theme-color" content="hsl(258, 51%, 52%)">

<!-- Named color (limited use) -->
<meta name="theme-color" content="purple">
```

Hex is the convention; most consistently parsed.

### 6.2 The Chrome Android Effect

When viewed on Chrome Android, the chrome (URL bar) takes the theme color:

```
[Notch / Status bar area = theme-color]
[URL bar = theme-color]
[Page content = whatever the page renders]
```

The status bar area at the top of the screen (where time, battery, signal display) also takes the theme color on most Android devices.

### 6.3 The Chrome Desktop Effect

On Chrome Desktop, theme-color has limited effect:

* Pinned tab color may be affected.
* PWA installed apps use theme-color for window chrome.
* General browsing: no obvious effect.

### 6.4 The Color Choice

For brand consistency:

* **Primary brand color**: usually the right choice for theme-color.
* **Secondary or accent color**: avoid; reserve for content.
* **Neutral color** (white, light gray): acceptable for light brands.

For Joseph's ThatDeveloperGuy:

```html
<meta name="theme-color" content="#6B46C1">
<!-- ThatDeveloperGuy purple, per standing brand rule -->
```

(Note: Joseph's standing rule specifies purple and green as the ThatDeveloperGuy palette; purple is the conventional primary for chrome.)

For TCB Fight Factory:

```html
<meta name="theme-color" content="#6B46C1">
<!-- TCB purple; their brand is purple and black -->
```

(Pure black #000000 is on Apple's prohibited list; purple is the safe brand primary.)

---

## 7. THE LIGHT AND DARK MODE MEDIA QUERY PATTERN

The HTMLMetaElement.media property enables different theme colors for light and dark mode. Widely available since October 2022.

### 7.1 The Standard Pattern

```html
<head>
    <meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
    <meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
</head>
```

The browser selects the variant matching the user's system color scheme.

### 7.2 The Default Pattern

For sites with single theme-color (no dark mode):

```html
<meta name="theme-color" content="#6B46C1">
<!-- No media query; applies in all color schemes -->
```

### 7.3 The Combined Pattern

For sites with default plus dark mode override:

```html
<meta name="theme-color" content="#6B46C1">
<!-- Default for any mode -->

<meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
<!-- Dark mode override -->
```

Browsers using the media variant apply the dark variant; others fall back to default.

### 7.4 The CSS Coordination

Pair with CSS to ensure body background matches theme-color across modes:

```css
body {
    background-color: #6B46C1;
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #1a1a1a;
    }
}
```

This handles both:
- Chrome and others: read media variant of theme-color.
- iOS 26 Safari: read body background color (which matches via CSS).

### 7.5 The Browser Coverage

| Browser | Single theme-color | Light/dark media variants |
|---|---|---|
| Chrome Android | Yes | Yes |
| Chrome Desktop | Partial | Yes |
| Firefox | Limited | Limited |
| Brave | Yes | Yes |
| Edge | Yes | Yes |
| iOS 26 Safari | **IGNORED** | Ignored (uses body bg) |
| iOS pre 26 Safari | Yes (with restrictions) | Yes |

### 7.6 The Bubbles Convention

Always include the dark mode variant when the site supports dark mode in CSS. Skip if site is light only.

---

## 8. THE APPLE RESTRICTED COLOR PALETTE

Apple (when Safari did honor theme-color, and presumably still in PWA context) maintains a list of prohibited colors. This applies to manifest theme_color as well.

### 8.1 The Prohibited Categories

Per documented Apple behavior:

* **Pure black** (#000000): rejected. Use #0a0a0a or similar near black.
* **Pure white** (#ffffff): accepted but rendered as default chrome.
* **Certain reds**: bright pure reds rejected. Acceptable: darker reds, brand specific.
* **Certain yellows**: high saturation yellows rejected.
* **Certain greens**: bright pure greens rejected.

### 8.2 The Tool For Verification

Roger Lee maintains a Safari Theme Color Tester:
- https://www.bram.us/2022/05/03/safari-theme-color-tool/ (or similar; URL may have moved)

You can enter a color and see if Safari accepts it. For unprohibited colors, the chrome takes the color. For prohibited colors, Safari falls back to gray.

For 2026, with iOS 26 using body background instead, the prohibited list may still apply (body background colors that are prohibited get adjusted) but verification needs to happen on iOS 26 directly.

### 8.3 The Safe Replacement Strategy

For brands with prohibited colors in their palette:

* **Brand uses pure black**: replace with #0a0a0a, #111111, or #1a1a1a. Visually nearly identical.
* **Brand uses bright red**: use a slightly desaturated or darker variant.
* **Brand uses bright yellow**: use a more muted yellow.
* **Brand uses bright green**: use a darker or more muted green.

### 8.4 The Bubbles Implication

For TCB Fight Factory (purple and black brand):

* Purple is safe.
* Pure black is prohibited.
* Use #0a0a0a or similar dark gray as "black" substitute for theme-color.

```html
<!-- TCB pattern -->
<meta name="theme-color" content="#6B46C1">
<!-- Primary: purple -->

<meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
<!-- Dark mode: near black (not pure black) -->
```

For ThatDeveloperGuy (purple and green):

* Both colors safe in standard variants.
* Use brand purple primary.

```html
<!-- ThatDeveloperGuy pattern -->
<meta name="theme-color" content="#6B46C1">
```

---

## 9. THE PWA MANIFEST INTEGRATION

For sites that are installable as Progressive Web Apps, the manifest.json `theme_color` provides the canonical chrome color for installed app contexts.

### 9.1 The Manifest Pattern

```json
{
    "name": "ThatDeveloperGuy",
    "short_name": "TDG",
    "start_url": "/",
    "display": "standalone",
    "theme_color": "#6B46C1",
    "background_color": "#1a1a1a",
    "icons": [
        {
            "src": "/icon-192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "/icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ]
}
```

Two color properties:

* **`theme_color`**: the chrome color for the installed PWA window. Matches the meta tag.
* **`background_color`**: the splash screen background color when the PWA launches.

### 9.2 The Linking

```html
<head>
    <meta name="theme-color" content="#6B46C1">
    <link rel="manifest" href="/site.webmanifest">
</head>
```

The manifest is linked from the head; the browser fetches it and uses it for PWA install behavior.

### 9.3 The Alignment Rule

`<meta name="theme-color">` content and manifest.json `theme_color` should match. Mismatches produce inconsistent appearance between:

* Standard browsing (reads meta tag).
* PWA mode (reads manifest).

For Bubbles convention: same value.

### 9.4 The Dark Mode Manifest

Web Manifest supports dark mode variants:

```json
{
    "theme_color": "#6B46C1",
    "background_color": "#1a1a1a",
    "user_preferences": {
        "color_scheme_dark": {
            "theme_color": "#1a1a1a",
            "background_color": "#000000"
        }
    }
}
```

Browser support varies; the standard is evolving. For now: provide light mode in manifest; dark mode via CSS media queries.

### 9.5 The PWA Setup Decision

For Bubbles client sites:

* **PWA capable** (the client wants installable app behavior): include manifest with theme_color.
* **Not PWA**: still include meta theme-color for browser chrome; skip manifest.

For ThatDeveloperGuy: PWA capable; manifest included.

For most client sites: PWA optional; usually not configured unless requested.

---

## 10. IOS SPECIFIC META TAGS (APPLE-MOBILE-WEB-APP-*)

Apple introduced PWA capability for iOS before the W3C web manifest standardization. The legacy meta tags are still used.

### 10.1 The Core iOS PWA Tags

```html
<head>
    <!-- Enable iOS PWA mode (full screen, no browser chrome) -->
    <meta name="apple-mobile-web-app-capable" content="yes">

    <!-- Status bar appearance in PWA mode -->
    <meta name="apple-mobile-web-app-status-bar-style" content="default">

    <!-- App title when added to home screen -->
    <meta name="apple-mobile-web-app-title" content="ThatDeveloperGuy">

    <!-- App icons -->
    <link rel="apple-touch-icon" href="/apple-touch-icon.png">
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon-180.png">
</head>
```

### 10.2 The status-bar-style Values

* **`default`**: white text on dark background (the typical default appearance).
* **`black`**: black background with white text.
* **`black-translucent`**: status bar overlays the content; transparent. Content extends behind status bar.

For full bleed designs (covered in framework-html-meta-viewport.md Section 10), `black-translucent` is often combined with `viewport-fit=cover`.

### 10.3 The iOS 26 Note

Per iOS 26 changes documented by Ben Frain:

> "You no longer need to have a site.webmanifest file at the root of the website, or the `<meta name='apple-mobile-web-app-capable'>` tag in your head."

iOS 26 introduces simplified "Add to Home Screen" that doesn't require these tags. However, including them ensures backwards compatibility with iOS 25 and earlier users.

### 10.4 The Full iOS PWA Set

For sites that want comprehensive iOS PWA support:

```html
<head>
    <!-- iOS PWA mode -->
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="default">
    <meta name="apple-mobile-web-app-title" content="ThatDeveloperGuy">

    <!-- iOS icons (multiple sizes) -->
    <link rel="apple-touch-icon" href="/apple-touch-icon.png">
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon-180.png">
    <link rel="apple-touch-icon" sizes="152x152" href="/apple-touch-icon-152.png">

    <!-- iOS splash screens (per device size) -->
    <link rel="apple-touch-startup-image" href="/splash-1242x2208.png" media="...">
    <!-- ... per device specs ... -->
</head>
```

### 10.5 The Bubbles Convention

For Bubbles client sites with PWA capability:

* Include `apple-mobile-web-app-capable`.
* Include `apple-mobile-web-app-status-bar-style` matching brand color decision.
* Include `apple-mobile-web-app-title` matching site name.
* Include apple-touch-icon (multiple sizes for best appearance).

For sites without PWA: skip these tags; meta theme-color is sufficient.

---

## 11. WINDOWS SPECIFIC META TAGS (MSAPPLICATION-*)

Microsoft Windows tile pinning uses meta tags. Less critical in 2026 but part of complete coverage.

### 11.1 The Tile Color

```html
<meta name="msapplication-TileColor" content="#6B46C1">
<meta name="msapplication-TileImage" content="/mstile-144x144.png">
```

When a Windows user pins the site to their Start menu, the tile uses these colors and images.

### 11.2 The Browser Config XML

For more comprehensive Windows configuration:

```html
<meta name="msapplication-config" content="/browserconfig.xml">
```

```xml
<!-- /browserconfig.xml -->
<?xml version="1.0" encoding="utf-8"?>
<browserconfig>
    <msapplication>
        <tile>
            <square150x150logo src="/mstile-150x150.png"/>
            <TileColor>#6B46C1</TileColor>
        </tile>
    </msapplication>
</browserconfig>
```

### 11.3 The 2026 Relevance

Windows tile pinning is rarely used in 2026. Most users don't pin web sites to Start. The full Windows configuration is optional polish.

For Bubbles: include the tile color and image for completeness; skip the browserconfig.xml unless a client specifically requests Windows tile coverage.

### 11.4 The Legacy navbutton-color

```html
<!-- LEGACY; IE9-11 specific -->
<meta name="msapplication-navbutton-color" content="#6B46C1">
```

This affected IE's navigation button color. IE is no longer supported (Edge replaced it). Omit this tag in modern sites.

---

## 12. THE COLOR-SCHEME CSS META TAG (DISTINCT FROM THEME-COLOR)

A different meta tag with a related but distinct purpose: `<meta name="color-scheme">`.

### 12.1 The Pattern

```html
<head>
    <meta name="color-scheme" content="light dark">
</head>
```

### 12.2 What It Does

The `color-scheme` meta tag tells the browser which color schemes the page supports. Values:

* **`light`**: page supports light mode only.
* **`dark`**: page supports dark mode only.
* **`light dark`**: page supports both (browser uses system preference).
* **`only light`**: page supports light only; force light even in dark mode system.
* **`only dark`**: page supports dark only.

### 12.3 The Effect

When a browser knows the page supports both schemes, it can:

* Use appropriate default colors for form elements (date pickers, scroll bars, etc.).
* Apply native dark mode styling where appropriate.
* Avoid the flash of light content when system is dark.

### 12.4 The Distinction From theme-color

* **`theme-color`**: declares a specific color for browser chrome.
* **`color-scheme`**: declares which color schemes the page supports.

Both can coexist; they serve different purposes.

```html
<head>
    <meta name="color-scheme" content="light dark">
    <meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
    <meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
</head>
```

### 12.5 The CSS Alternative

The same can be declared via CSS:

```css
:root {
    color-scheme: light dark;
}
```

Equivalent to the meta tag. CSS is the more modern convention.

### 12.6 The Bubbles Convention

Include `<meta name="color-scheme">` or the CSS equivalent on sites supporting both color schemes. For light only sites, omit (or use `only light`).

---

## 13. BRAND COLOR STRATEGY

The theme-color choice should reflect the brand's identity. For Bubbles, Joseph maintains specific brand color rules.

### 13.1 The ThatDeveloperGuy Brand

Per Joseph's standing rule: ThatDeveloperGuy brand colors are **purple and green**.

* **Primary purple**: typical hex around #6B46C1 (Tailwind purple-700 family) or similar.
* **Secondary green**: a complementary green for accents.

For theme-color:

```html
<meta name="theme-color" content="#6B46C1">
<!-- Primary brand purple -->
```

Dark mode:

```html
<meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
<!-- Dark mode: near black instead of purple (lighter on dark gives bad contrast for chrome) -->
```

Or for brand consistent dark mode:

```html
<meta name="theme-color" content="#4a2c8a" media="(prefers-color-scheme: dark)">
<!-- Darker purple variant for dark mode -->
```

### 13.2 The TCB Fight Factory Brand

Per Joseph's standing rule: TCB brand colors are **purple and black only**.

* **Primary purple**: same brand purple as TDG (Joseph runs the brand).
* **Secondary black**: pure black for body text and accents.

For theme-color:

```html
<meta name="theme-color" content="#6B46C1">
<!-- TCB purple primary -->
```

Dark mode (with the prohibited color note):

```html
<meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
<!-- Near black (Safari prohibits pure black) -->
```

For body background to match (iOS 26):

```css
body {
    background-color: #6B46C1;
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #0a0a0a;
    }
}
```

### 13.3 The Local Business Client Patterns

Each Bubbles local business client has their own brand colors:

* **Heritage Hardwood Floors NWA**: wood tones (warm browns).
* **Arkansas Counseling and Wellness Services**: calming blues or greens.
* **White River Cabins**: nature greens or earth tones.
* **Avid Pest Control**: bold accent colors.
* **RAH Construction NWA**: industrial blues or grays.

For each client: define brand primary; use as theme-color; verify against Apple prohibited list; apply CSS body background to match.

### 13.4 The Client Decision Question

When onboarding a new client, ask:

1. What is your brand primary color?
2. Do you have a secondary color?
3. Do you want dark mode support?
4. Should the browser chrome match brand colors?

Document the answers; configure theme-color accordingly.

### 13.5 The Accessibility Verification

Browser chrome includes UI elements (URL text, status icons) that need contrast against the theme color. Test:

* Brand color with white text overlay: sufficient contrast?
* Brand color with dark text overlay: sufficient contrast?

For most brand purples: white text overlay has good contrast.
For light brand colors: dark text overlay is better but Chrome doesn't always use it.

Use the WebAIM Contrast Checker (https://webaim.org/resources/contrastchecker/) to verify.

---

## 14. THE BUBBLES PER CLIENT COLOR DECISIONS

Each client site needs a documented theme color decision.

### 14.1 The Decision Framework

For each client:

1. **What is the brand primary color?**
2. **Is the color on Apple's prohibited list?**
3. **Does the site support dark mode?**
4. **Is the site PWA installable?**
5. **What is the body background color in CSS?**

### 14.2 The Per Client Decisions Table

| Client | Primary | Dark mode variant | Body background | PWA |
|---|---|---|---|---|
| ThatDeveloperGuy.com | #6B46C1 (purple) | #1a1a1a | matches primary | Yes |
| TCB Fight Factory | #6B46C1 (purple) | #0a0a0a (not pure black) | matches primary | Optional |
| Heritage Hardwood Floors NWA | brown brand color | dark warm tone | matches primary | No |
| Arkansas Counseling and Wellness | calming blue/green | matching dark | matches primary | No |
| Handled Tax (Amanda Emerdinger) | client decision | matching dark | matches primary | No |
| White River Cabins | nature green | dark green | matches primary | No |
| Greenough's Guide Service | brand color | matching dark | matches primary | No |
| Diana Undergust (Lake Homes Realty) | real estate brand | matching dark | matches primary | No |
| WeCoverUSA (federal) | corporate brand | matching dark | matches primary | Yes (federal app) |

(Specific hex values are placeholders; document actual brand colors per client.)

### 14.3 The Documentation Pattern

For each client, document in the project notes:

```
Brand colors:
  Primary: #6B46C1 (purple)
  Secondary: #1a1a1a (near black)
  Accent: #88c70a (green for ThatDeveloperGuy)

Theme color implementation:
  <meta name="theme-color" content="#6B46C1">
  <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">

  body { background-color: #6B46C1; }
  @media (prefers-color-scheme: dark) { body { background-color: #0a0a0a; } }

PWA manifest:
  theme_color: #6B46C1
  background_color: #1a1a1a

Apple prohibited check: PASSED.
```

---

## 15. THE IOS 26 WORKAROUND PATTERNS

Three patterns for handling iOS 26 Safari behavior.

### 15.1 Pattern A: Body Background Match (Recommended)

The simplest and most reliable:

```html
<head>
    <meta name="theme-color" content="#6B46C1">
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
</head>
<body>
    <main>
        <!-- ... content ... -->
    </main>
</body>

<style>
body {
    background-color: #6B46C1;
    margin: 0;
    padding: 0;
}

main {
    background-color: white;
    /* Main content can have different background; iOS 26 picks from top */
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #0a0a0a;
    }

    main {
        background-color: #1a1a1a;
        color: white;
    }
}
</style>
```

Pros: works reliably; simple; matches Apple's intended behavior.

Cons: requires body background to be the chrome color, which may interfere with full bleed designs.

### 15.2 Pattern B: Fixed Element Trick

For sites where body background needs to be different from chrome color:

```html
<head>
    <meta name="theme-color" content="#6B46C1">
</head>
<body>
    <div class="ios-chrome-color" aria-hidden="true"></div>
    <main>
        <!-- ... content with white background ... -->
    </main>
</body>

<style>
.ios-chrome-color {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 1px;
    background-color: #6B46C1;
    z-index: -1;
    pointer-events: none;
}

body {
    background-color: white; /* Or whatever the main content background is */
}

main {
    background-color: white;
}
</style>
```

Pros: body can have any background.

Cons: per Ben Frain's analysis, breaks when page reloads at non zero scroll position.

### 15.3 Pattern C: Scroll Aware (Advanced)

For sites that need to change chrome color based on scroll position:

```html
<head>
    <meta name="theme-color" content="#6B46C1" id="theme-color-meta">
</head>
<body>
    <!-- ... content ... -->
</body>

<script>
const themeColorMeta = document.getElementById('theme-color-meta');

window.addEventListener('scroll', () => {
    const scrollPos = window.scrollY;

    if (scrollPos < 100) {
        themeColorMeta.setAttribute('content', '#6B46C1');
        document.body.style.backgroundColor = '#6B46C1';
    } else {
        themeColorMeta.setAttribute('content', '#ffffff');
        document.body.style.backgroundColor = '#ffffff';
    }
});
</script>
```

For sites with hero sections that should have different chrome than the rest of the page.

Pros: dynamic; matches scrolling context.

Cons: JavaScript dependent; may flash on initial load; complex.

### 15.4 The Bubbles Default

Use Pattern A (body background match). Simplest, most reliable, lowest maintenance. Document in project notes per client.

---

## 16. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 16.1 Canonical Bubbles head with theme-color

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Page Title | ThatDeveloperGuy</title>
    <meta name="description" content="...">

    <!-- Theme color (light mode default) -->
    <meta name="theme-color" content="#6B46C1">

    <!-- Theme color (dark mode variant) -->
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">

    <!-- Color scheme support declaration -->
    <meta name="color-scheme" content="light dark">

    <!-- ... -->
</head>
<body>
    <!-- ... -->
</body>

<style>
/* For iOS 26 Safari coverage */
body {
    background-color: #6B46C1;
    margin: 0;
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #0a0a0a;
    }
}
</style>
```

### 16.2 With full PWA configuration

```html
<head>
    <!-- Standard meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
    <title>ThatDeveloperGuy</title>

    <!-- Theme color -->
    <meta name="theme-color" content="#6B46C1">
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">

    <!-- Color scheme -->
    <meta name="color-scheme" content="light dark">

    <!-- iOS PWA -->
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="default">
    <meta name="apple-mobile-web-app-title" content="TDG">

    <!-- iOS icons -->
    <link rel="apple-touch-icon" href="/apple-touch-icon.png">

    <!-- PWA manifest -->
    <link rel="manifest" href="/site.webmanifest">

    <!-- Windows -->
    <meta name="msapplication-TileColor" content="#6B46C1">
    <meta name="msapplication-TileImage" content="/mstile-144x144.png">
</head>
```

### 16.3 The site.webmanifest

```json
{
    "name": "ThatDeveloperGuy.com",
    "short_name": "TDG",
    "description": "Hand coded websites for Northwest Arkansas.",
    "start_url": "/",
    "display": "standalone",
    "theme_color": "#6B46C1",
    "background_color": "#1a1a1a",
    "icons": [
        {
            "src": "/icon-192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "/icon-512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ]
}
```

### 16.4 TCB Fight Factory pattern (purple and black, with prohibited color workaround)

```html
<head>
    <meta name="theme-color" content="#6B46C1">
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
    <!-- Note: #0a0a0a not #000000 (Safari prohibits pure black) -->
</head>

<style>
body {
    background-color: #6B46C1;
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #0a0a0a;
    }
}
</style>
```

### 16.5 Local business client pattern (Heritage Hardwood Floors NWA, warm brown)

```html
<head>
    <meta name="theme-color" content="#8B4513">
    <!-- Brand: saddle brown for hardwood -->
    <meta name="theme-color" content="#2c1810" media="(prefers-color-scheme: dark)">
</head>

<style>
body {
    background-color: #8B4513;
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #2c1810;
    }
}
</style>
```

### 16.6 White background brand (light/airy)

```html
<head>
    <meta name="theme-color" content="#ffffff">
    <meta name="theme-color" content="#1a1a1a" media="(prefers-color-scheme: dark)">
</head>

<style>
body {
    background-color: #ffffff;
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #1a1a1a;
    }
}
</style>
```

### 16.7 Light theme + body match (full bleed compatible)

```html
<head>
    <meta name="theme-color" content="#6B46C1">
</head>
<body>
    <!-- Main content with white background -->
    <main style="background: white; min-height: 100vh;">
        <!-- ... content ... -->
    </main>
</body>

<style>
body {
    background-color: #6B46C1;
    margin: 0;
}
</style>
```

The chrome color is purple (theme-color and body match). Main content area is white.

### 16.8 The verification script

```bash
#!/bin/bash
# /usr/local/bin/theme-color-check.sh

URL=$1

echo "=== Theme color check for $URL ==="

# Get meta theme-color
META=$(curl -s "$URL" | grep -oE 'meta name="theme-color" [^>]+' | head -3)
echo "Meta theme-color:"
echo "$META"

# Check for media variant
DARK=$(echo "$META" | grep "prefers-color-scheme: dark")
if [ -n "$DARK" ]; then
    echo "  Dark mode variant: present"
else
    echo "  Dark mode variant: MISSING"
fi

# Check body CSS for iOS 26 coverage
BG=$(curl -s "$URL" | grep -oE 'background-color: *#[0-9a-fA-F]+' | head -3)
echo "Body background-color hints:"
echo "$BG"

# Check for prohibited colors
if echo "$META" | grep -qE '#000000\|#000[^0-9a-fA-F]'; then
    echo "WARNING: pure black detected; Safari may not honor"
fi
```

---

## 17. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles configuration for theme color.

### 17.1 The Template Default

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- Brand color from client config -->
    <meta name="theme-color" content="{{ brand_primary_color }}">
    <meta name="theme-color" content="{{ brand_dark_color }}" media="(prefers-color-scheme: dark)">

    <!-- Color scheme support -->
    <meta name="color-scheme" content="light dark">

    <!-- ... -->
</head>
<body class="brand-bg">
    <!-- ... -->
</body>
```

### 17.2 The Per Client Config

```python
# /opt/bubbles/services/example.com/client_config.py

CLIENT_BRAND_COLORS = {
    "thatdeveloperguy": {
        "primary": "#6B46C1",
        "dark_mode": "#0a0a0a",
        "secondary": "#88c70a",  # ThatDeveloperGuy green
        "manifest_bg": "#1a1a1a",
    },
    "tcb_fight_factory": {
        "primary": "#6B46C1",
        "dark_mode": "#0a0a0a",  # not pure black; Safari prohibits
        "manifest_bg": "#0a0a0a",
    },
    "heritage_hardwood": {
        "primary": "#8B4513",  # saddle brown
        "dark_mode": "#2c1810",  # darker warm tone
        "manifest_bg": "#2c1810",
    },
    # ... per client
}

def get_theme_colors(client_key):
    return CLIENT_BRAND_COLORS.get(client_key, {
        "primary": "#6B46C1",
        "dark_mode": "#0a0a0a",
    })
```

### 17.3 The CSS Template

```css
/* /var/www/sites/example.com/brand.css */

:root {
    --brand-primary: #6B46C1;
    --brand-dark: #0a0a0a;
}

body.brand-bg {
    background-color: var(--brand-primary);
    margin: 0;
}

@media (prefers-color-scheme: dark) {
    body.brand-bg {
        background-color: var(--brand-dark);
    }
}

main {
    background-color: white;
    min-height: 100vh;
}

@media (prefers-color-scheme: dark) {
    main {
        background-color: #1a1a1a;
        color: white;
    }
}
```

### 17.4 The Audit Workflow

```bash
#!/bin/bash
# /usr/local/bin/theme-color-audit.sh

echo "=== Theme color audit ==="

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"

    if [ -f "$INDEX" ]; then
        META=$(grep -oE 'meta name="theme-color"[^>]+' "$INDEX" | head -3)

        if [ -z "$META" ]; then
            echo "MISSING: $SITE has no theme-color"
        else
            # Check for dark variant
            if ! echo "$META" | grep -q "prefers-color-scheme: dark"; then
                echo "NO DARK VARIANT: $SITE"
            fi

            # Check for prohibited colors
            if echo "$META" | grep -qE '#000000\|#000[^0-9a-fA-F]'; then
                echo "PROHIBITED COLOR: $SITE uses pure black"
                echo "  $META"
            fi
        fi

        # Check body background
        if ! grep -q "background-color:" "$INDEX"; then
            STYLE_FILE="$site_dir/styles.css"
            if [ -f "$STYLE_FILE" ]; then
                if ! grep -q "body" "$STYLE_FILE" | grep -q "background-color"; then
                    echo "NO BODY BG: $SITE (iOS 26 won't have chrome color)"
                fi
            fi
        fi
    fi
done

echo "=== End audit ==="
```

### 17.5 After Configuration Changes

```bash
# Reload nginx if static files changed
nginx -t && systemctl reload nginx

# Or restart FastAPI sidecar for templated content
systemctl restart fastapi-sidecar

# Verify on a sample URL
theme-color-check.sh https://example.com/
```

---

## 18. AUDIT CHECKLIST

Run through these 45 items for production deployment.

### Core theme color tag

1. [ ] `<meta name="theme-color">` present on every page.
2. [ ] Color value is valid CSS color format (hex preferred).
3. [ ] Color matches brand primary.

### Dark mode variant

4. [ ] Dark mode variant present via `media="(prefers-color-scheme: dark)"`.
5. [ ] Dark mode variant color is appropriate (good contrast).
6. [ ] Dark mode variant tested on real dark mode device.

### iOS 26 Safari coverage

7. [ ] Body background-color matches theme-color in CSS.
8. [ ] Body background-color has dark mode variant matching dark theme-color.
9. [ ] Tested on real iOS 26 Safari device.
10. [ ] Documentation notes iOS 26 behavior change.

### Apple prohibited color list

11. [ ] No pure black (#000000) in any theme-color.
12. [ ] No bright unsuitable reds checked.
13. [ ] No bright unsuitable yellows checked.
14. [ ] No bright unsuitable greens checked.

### PWA manifest (if applicable)

15. [ ] manifest.json includes theme_color.
16. [ ] manifest theme_color matches meta theme-color.
17. [ ] manifest includes background_color (splash screen).
18. [ ] manifest icons provided (multiple sizes).

### iOS specific tags (if PWA)

19. [ ] `apple-mobile-web-app-capable` present.
20. [ ] `apple-mobile-web-app-status-bar-style` configured.
21. [ ] `apple-mobile-web-app-title` set.
22. [ ] `apple-touch-icon` link present.

### Windows specific tags (optional)

23. [ ] `msapplication-TileColor` if Windows tile important.
24. [ ] `msapplication-TileImage` if Windows tile important.

### Color scheme support

25. [ ] `<meta name="color-scheme">` set to match site capability.
26. [ ] Or CSS `color-scheme: light dark;` equivalent.

### Cross browser verification

27. [ ] Tested on Chrome Android.
28. [ ] Tested on Chrome Desktop.
29. [ ] Tested on iOS 26 Safari.
30. [ ] Tested on iOS pre 26 Safari (legacy).
31. [ ] Tested on Firefox.
32. [ ] Tested on Brave or Edge.

### Brand consistency

33. [ ] Theme color matches documented brand primary.
34. [ ] All pages on a site use same theme color (unless intentionally differentiated).
35. [ ] No conflicting theme-color tags (Yoast or similar plugins).

### Bubbles per client documentation

36. [ ] Brand primary documented per client.
37. [ ] Dark mode variant documented per client.
38. [ ] Apple prohibited color check documented.
39. [ ] PWA manifest decision documented.

### Accessibility

40. [ ] Theme color provides good contrast for chrome UI text.
41. [ ] No accessibility concerns flagged by audit tools.

### Cross cutting

42. [ ] No duplicate theme-color tags (multiple in one page).
43. [ ] Theme-color placement appropriate in head (after charset, viewport).

### CMS migration cleanup

44. [ ] Migrated sites have appropriate brand theme-color (not leftover platform color).
45. [ ] WordPress sites that auto generated theme-color have it audited.

A site that passes all 45 has correctly configured theme color for current 2026 browser coverage including iOS 26 Safari workaround.

---

## 19. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: theme-color works on Chrome but not iOS 26 Safari.**
Symptom: Chrome shows brand color in chrome; iOS 26 Safari shows gray.
Why it breaks: iOS 26 Safari dropped theme-color support.
Fix: also set `body { background-color: <brand color>; }` in CSS.

**Pitfall 2: Pure black theme-color ignored by Safari.**
Symptom: `<meta name="theme-color" content="#000000">` doesn't work in Safari.
Why it breaks: Apple's prohibited color list includes pure black.
Fix: use #0a0a0a or #1a1a1a instead.

**Pitfall 3: Bright red theme-color ignored by Safari.**
Symptom: `<meta name="theme-color" content="#FF0000">` doesn't work in Safari.
Why it breaks: Apple's prohibited color list includes bright pure red.
Fix: use a darker or desaturated red, or test with the Safari color checker tool.

**Pitfall 4: Dark mode produces bad contrast.**
Symptom: dark mode user sees brand purple chrome with light system colors; URL hard to read.
Why it breaks: brand color may not have sufficient contrast against UI text in dark mode.
Fix: add dark mode media variant with appropriate dark color.

**Pitfall 5: Multiple conflicting theme-color tags.**
Symptom: page has theme-color from Yoast AND from custom template; browser uses one.
Why it breaks: plugins or templates duplicate the tag.
Fix: remove duplicates; ensure single canonical theme-color.

**Pitfall 6: theme_color in manifest doesn't match meta tag.**
Symptom: standard browsing shows one color; PWA mode shows different color.
Why it breaks: mismatch between meta tag and manifest.
Fix: align both to same value.

**Pitfall 7: Body background doesn't match theme-color (iOS 26 issue).**
Symptom: chrome color set via meta tag; iOS 26 ignores and uses body background.
Why it breaks: body has different background than theme-color.
Fix: set body background-color to match theme-color value.

**Pitfall 8: Fixed element trick breaks on page reload at scroll position.**
Symptom: fixed element provides theme color when at top of page; doesn't work after reload deeper.
Why it breaks: per Ben Frain's analysis; fixed element approach is fragile.
Fix: use body background-color approach instead.

**Pitfall 9: Apple Mobile Web App Capable tag missing.**
Symptom: iOS users add site to home screen; opens in Safari instead of PWA mode.
Why it breaks: iOS PWA mode requires the tag (pre iOS 26; iOS 26 has simplified add to home).
Fix: include `<meta name="apple-mobile-web-app-capable" content="yes">`.

**Pitfall 10: WordPress site auto generated theme-color from plugin.**
Symptom: page has theme-color set to a non brand color.
Why it breaks: SEO or theme plugin set it automatically.
Fix: remove plugin auto generation; add hand crafted theme-color.

**Pitfall 11: theme-color hex with shorthand vs full.**
Symptom: `#FFF` vs `#FFFFFF`; some browsers parse one but not the other.
Why it breaks: hex shorthand support varies (most browsers accept; consistency matters).
Fix: use full 6 character hex format for consistency.

**Pitfall 12: theme-color is rgba() with transparency.**
Symptom: `rgba(107, 70, 193, 0.5)` doesn't work; browsers use opaque only.
Why it breaks: theme-color requires opaque color.
Fix: use opaque hex or rgb without alpha.

**Pitfall 13: msapplication-TileColor and theme-color mismatch.**
Symptom: Windows tile shows different color than mobile chrome.
Why it breaks: separate tags configured independently.
Fix: align both to brand primary.

**Pitfall 14: Brand color renders too dark for chrome text overlay.**
Symptom: very dark brand color makes URL text hard to read against chrome.
Why it breaks: insufficient contrast.
Fix: choose slightly lighter brand variant for theme-color, or use opposite mode pattern.

**Pitfall 15: theme-color not set for the body background, but iOS works.**
Symptom: iOS 26 Safari shows brand color despite no body background set.
Why it breaks: iOS may pick color from another element near top (header, hero).
Fix: explicitly set body background-color for predictable behavior.

---

## 20. DIAGNOSTIC COMMANDS

Reference of commands useful for theme color investigation.

### Inspect a single URL

```bash
# Get all theme-color tags
curl -s https://example.com/ | grep -oE 'meta name="theme-color"[^>]+'

# Get dark mode variant specifically
curl -s https://example.com/ | grep -oE 'meta name="theme-color" content="[^"]+" media="\(prefers-color-scheme: dark\)"'

# Check for body background-color
curl -s https://example.com/ | grep -oE 'body[^{]*\{[^}]*background-color[^}]*\}' | head -3
```

### Get PWA manifest theme color

```bash
# Find manifest link
MANIFEST_URL=$(curl -s https://example.com/ | grep -oE 'link rel="manifest" href="[^"]+"' | sed 's/.*href="\([^"]*\)".*/\1/')

# Download manifest
curl -s "https://example.com${MANIFEST_URL}" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(f\"theme_color: {data.get('theme_color', 'not set')}\")
print(f\"background_color: {data.get('background_color', 'not set')}\")
"
```

### Bulk audit Bubbles client sites

```bash
theme-color-audit.sh
```

### Check for Apple prohibited colors

```bash
URL=$1
TC=$(curl -s "$URL" | grep -oE 'meta name="theme-color" content="[^"]+"' | head -3 | sed 's/.*content="\([^"]*\)".*/\1/')

for color in $TC; do
    # Check for pure black
    if [ "$color" = "#000000" ] || [ "$color" = "#000" ]; then
        echo "WARNING: pure black ($color) detected; Safari prohibits"
    fi
    # Check for bright pure red, green, yellow
    case "$color" in
        "#ff0000"|"#FF0000"|"#f00"|"#F00"|"#ffff00"|"#FFFF00"|"#ff0"|"#FF0"|"#00ff00"|"#00FF00"|"#0f0"|"#0F0")
            echo "WARNING: bright pure color ($color) likely on Apple prohibited list"
            ;;
    esac
done
```

### Visual inspection

In browser:

1. **Chrome Android**: open page; look at top of screen (URL bar + status area).
2. **iOS 26 Safari**: open page; look at top of screen (compact tab bar).
3. **Toggle dark mode**: verify color changes appropriately.
4. **Inspect element**: check `<meta name="theme-color">` and body background.

In Lighthouse:

```bash
lighthouse https://example.com/ \
    --only-audits=themed-omnibox \
    --output=json --output-path=/tmp/lh.json --quiet
```

Lighthouse `themed-omnibox` audit checks for theme-color presence and validity.

### Real device testing

Required for iOS 26 verification:
- iPhone or iPad running iOS 26+.
- Open Safari (not Chrome on iOS; same engine but different behavior).
- Browse to the site.
- Observe compact tab bar color.
- Toggle Safari's "Show color in compact tab bar" if needed.

### After changes

```bash
# Reload nginx if static
nginx -t && systemctl reload nginx

# Or restart FastAPI
systemctl restart fastapi-sidecar

# Verify
theme-color-check.sh https://example.com/
```

---

## 21. CROSS-REFERENCES

* [framework-html-meta-viewport.md](framework-html-meta-viewport.md): viewport-fit=cover for notched displays; combined with theme-color for full bleed mobile experience.
* [framework-html-meta-charset.md](framework-html-meta-charset.md): theme-color placement after charset.
* [framework-html-meta-description.md](framework-html-meta-description.md): aligns with meta description in head order.
* [framework-html-meta-generator.md](framework-html-meta-generator.md): the "Hand coded by ThatDeveloperGuy.com" generator tag pairs with brand color theme-color for full brand reinforcement.
* [framework-http-security-headers.md](framework-http-security-headers.md): the Permissions-Policy header can affect color rendering of certain APIs (rare).
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): theme color not a ranking signal but polish affects user behavior signals.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including brand color configuration.
* W3C theme-color spec: https://www.w3.org/TR/appmanifest/#theme_color-member
* MDN meta name=theme-color: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name/theme-color
* MDN HTMLMetaElement.media: https://developer.mozilla.org/en-US/docs/Web/API/HTMLMetaElement/media
* Web App Manifest theme_color: https://developer.mozilla.org/en-US/docs/Web/Manifest/theme_color
* CSS Tricks Meta Theme Color and Trickery: https://css-tricks.com/meta-theme-color-and-trickery/
* Apple Developer Forums on iOS 26 theme-color: https://developer.apple.com/forums/thread/801239
* Ben Frain iOS 26 Safari theme-color analysis: https://benfrain.com/ios26-safari-theme-color-tab-tinting-with-fixed-position-elements/
* grooovinger Define the Theme Color for Safari 26: https://grooovinger.com/notes/2026-02-27-safari-26-header-background
* WebAIM Contrast Checker: https://webaim.org/resources/contrastchecker/
* Lighthouse themed-omnibox audit: https://web.dev/articles/themed-omnibox

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Include theme-color for Chrome. Match body background for iOS 26 Safari. Include dark mode variant.**

### The canonical pattern

```html
<head>
    <meta name="theme-color" content="#6B46C1">
    <meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
    <meta name="color-scheme" content="light dark">
</head>

<style>
body {
    background-color: #6B46C1;
}

@media (prefers-color-scheme: dark) {
    body {
        background-color: #0a0a0a;
    }
}
</style>
```

### The critical 2026 note

iOS 26 Safari dropped support for `<meta name="theme-color">`. It now reads body/html background color. The body background-color CSS is REQUIRED for iOS 26 coverage.

### The Apple prohibited list

* NO pure black (#000000); use #0a0a0a.
* NO bright pure reds, yellows, greens; use darker variants.

### Bubbles brand colors

* **ThatDeveloperGuy**: purple (#6B46C1) and green; primary purple.
* **TCB Fight Factory**: purple and black; use #6B46C1 primary, #0a0a0a (not pure black).
* Each local client: per client brand color decision documented.

### Five rules to memorize

1. **Include both meta theme-color AND body background CSS (iOS 26 coverage).**
2. **Add dark mode media variant for dark mode users.**
3. **Avoid Apple's prohibited colors (especially pure black).**
4. **Align manifest.json theme_color with meta theme-color if PWA.**
5. **Test on real Chrome Android AND real iOS 26 Safari (simulators may differ).**

### Five commands every operator should know

```bash
# 1. Get theme-color tags from a URL
curl -s URL | grep -oE 'meta name="theme-color"[^>]+'

# 2. Check for prohibited pure black
curl -s URL | grep -E 'theme-color.*#000000\|theme-color.*#000"'

# 3. Audit across all Bubbles client sites
theme-color-audit.sh

# 4. Get PWA manifest theme_color
MANIFEST=$(curl -s URL | grep -oE 'link rel="manifest" href="[^"]+"' | sed 's/.*href="\([^"]*\)".*/\1/')
curl -s "URL${MANIFEST}" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('theme_color'))"

# 5. Apply CSS or template changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. theme-color present
URL=https://example.com/
HAS_TC=$(curl -s "$URL" | grep -c 'meta name="theme-color"')
[ "$HAS_TC" -ge "1" ] && echo "OK: theme-color present" || echo "FAIL"

# 2. Body background-color matches (iOS 26 coverage)
TC_COLOR=$(curl -s "$URL" | grep -oE 'meta name="theme-color" content="[^"]+"' | head -1 | sed 's/.*content="\([^"]*\)".*/\1/')
BODY_BG=$(curl -s "$URL" | grep -oE 'body[^{]*\{[^}]*background-color: *[^;]+' | head -1)
echo "theme-color: $TC_COLOR"
echo "body bg: $BODY_BG"

# 3. No Apple prohibited colors
PROHIBITED=$(curl -s "$URL" | grep -E 'theme-color.*content="(#000|#000000|#FF0000|#FFFF00|#00FF00)"')
[ -z "$PROHIBITED" ] && echo "OK: no prohibited colors" || echo "FAIL: $PROHIBITED"
```

If all three pass AND the chrome color displays correctly on real Chrome Android AND real iOS 26 Safari devices, the theme color layer is correctly wired across the 2026 browser landscape.

---

End of framework-html-meta-theme-color.md.
