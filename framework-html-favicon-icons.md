# framework-html-favicon-icons.md

Comprehensive reference for the favicon and icon family, the set of HTML `<link>` elements and associated image files that control how a website is represented as a small icon across browser tabs, bookmarks, search results, mobile home screens, Windows tiles, Android launchers, PWA install prompts, RSS reader sidebars, AI assistant link unfurls, and operating system level pinned shortcuts. Covers the modern 2026 minimal stack consensus (one ICO file, one SVG file, one Apple Touch Icon, optional 192x192 and 512x512 PNGs in the Web App Manifest icons array), the legacy stack (16x16, 32x32, 48x48, 60x60, 70x70, 72x72, 76x76, 96x96, 114x114, 120x120, 144x144, 152x152, 180x180, 192x192, 256x256, 310x310, 384x384, 512x512), the `<link rel="icon" href="/favicon.ico">` legacy fallback role (still essential because RSS readers and feed crawlers fetch `/favicon.ico` directly without parsing HTML), the `<link rel="icon" type="image/svg+xml" href="/icon.svg">` modern primary role (supported in Chrome, Firefox, Edge, and Safari since 2024) with the dark mode pattern (`@media (prefers-color-scheme: dark)` inside the SVG `<style>` tag), the `<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">` iOS home screen role, the `<link rel="mask-icon" href="/icon.svg" color="#...">` Safari pinned tab role (mostly redundant in 2026 because modern Safari uses the SVG favicon directly), the `<link rel="manifest" href="/manifest.webmanifest">` reference and the manifest icons array with sizes, type, and the `purpose: "maskable"` field for Android adaptive icons, the PNG fallback family (favicon-16x16.png and favicon-32x32.png declared via `<link rel="icon" type="image/png" sizes="...">`), the asset generation pipeline using ImageMagick `convert` or libvips, the aggressive browser caching that makes favicon updates frustrating (cache busting query parameter or filename versioning required), the Bubbles per client decision framework with the standard modern minimal stack as default and the federal SDVOSB SVG dark mode pattern for ThatDeveloperGuy. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the nineteenth framework in the HTML signal track**, following the meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, color-scheme, referrer, content-language, refresh, Open Graph, Twitter Cards, search engine verification, App and PWA, and canonical/alternate frameworks. Direct companion to framework-html-app-pwa-meta.md (which covered `apple-touch-icon` and `<link rel="manifest">` from the PWA angle) and framework-html-meta-theme-color.md (which controls the chrome color that frames the icons in PWA mode).

Audience: humans implementing the modern favicon stack for client sites, AI assistants generating HTML head sections with the correct icon link family, designers producing the source SVG and PNG icon assets, Bubbles operators generating the icon family from a single source via ImageMagick or libvips, federal subcontractors maintaining brand identity across browser tabs and procurement officer LinkedIn previews, and anyone troubleshooting "the favicon looks pixelated in the browser tab", "Google search results show a generic icon instead of our brand", "the dark mode browser tab shows our dark logo invisibly on dark background", "the iOS home screen icon has a white square behind it", or "the new favicon won't update no matter how many times we refresh".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Mental Model: Modern Minimal Stack
5. The 2026 Consensus On The Minimal Stack
6. The Legacy Stack History (And Why To Skip It)
7. favicon.ico (Legacy Fallback)
8. The SVG Favicon (Modern Primary)
9. The Dark Mode SVG Favicon Pattern
10. The PNG Family (16x16 And 32x32)
11. apple-touch-icon (iOS Home Screen)
12. mask-icon (Safari Pinned Tab)
13. The Manifest Link Reference
14. The Web App Manifest Icons Array
15. The Maskable Icon Pattern (Android Adaptive Icons)
16. The Sizes And Type Attributes
17. The Asset Generation Pipeline
18. The Caching And Versioning Discipline
19. The Bubbles Per Client Decision Framework
20. The Server Side File Placement Requirement
21. Implementation Patterns By Stack
22. The Pre Launch Audit Checklist
23. The Common Failures And Fixes
24. The Federal And SDVOSB Context
25. Quick Reference Tag Block
26. References

---

## 1. DEFINITION

The favicon and icon family consists of HTML `<link>` elements in the `<head>` that point to image files representing the site at various sizes and in various platform specific formats.

```html
<head>
    <link rel="icon" href="/favicon.ico" sizes="32x32">
    <link rel="icon" type="image/svg+xml" href="/icon.svg">
    <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
    <link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
    <link rel="mask-icon" href="/safari-pinned-tab.svg" color="#6B3FA0">
    <link rel="manifest" href="/manifest.webmanifest">
</head>
```

The image files themselves live at the web root or in a sibling assets directory. The HTML head declares which files exist and what their sizes and types are; browsers, operating systems, and tools pick the appropriate asset based on context.

Five files cover virtually every modern use case in 2026:

1. **`/favicon.ico`** — legacy fallback. Required because RSS readers, feed crawlers, AI assistant unfurlers, and many tools fetch `/favicon.ico` directly without parsing HTML.
2. **`/icon.svg`** — modern primary. Renders crisply at any size, supports dark mode via embedded CSS media queries.
3. **`/apple-touch-icon.png`** — iOS home screen icon, 180x180 PNG.
4. **`/icon-192.png`** — Android home screen icon, declared in the manifest.
5. **`/icon-512.png`** — Android splash screen and PWA install dialog, declared in the manifest.

Plus the optional `favicon-16x16.png` and `favicon-32x32.png` for Google search results (which specifically look for PNG favicons) and platforms with imperfect SVG handling.

---

## 2. WHY IT MATTERS

The favicon is the smallest brand asset on the site but appears in more places than any other:

* **Browser tabs**: every visitor sees the favicon every time they have the site open.
* **Bookmarks and history**: stored alongside the URL forever.
* **Browser tab grouping** (Chrome, Edge): the favicon is the primary visual identifier.
* **Google search results**: a 16x16 favicon appears next to the title on mobile and in some desktop layouts.
* **Bing search results**: same.
* **iOS home screen** (when bookmarked or installed as PWA): the apple-touch-icon at 180x180.
* **Android home screen** (when installed as PWA): the 192x192 manifest icon.
* **Android splash screen** (PWA install): the 512x512 manifest icon.
* **PWA install prompts**: the 512x512 manifest icon.
* **Windows Start menu tiles** (legacy, Windows 10 Edge): the msapplication-TileImage (covered in framework-html-app-pwa-meta.md Section 12).
* **RSS reader sidebars** (Feedly, Inoreader, NetNewsWire): typically fetch `/favicon.ico` directly.
* **AI assistant link unfurls** (ChatGPT, Claude, Perplexity, Gemini): increasingly fetch the favicon to render alongside the unfurled link card.
* **Slack and Discord link unfurls**: often use the favicon as the small icon next to the unfurled card.
* **Email signatures and corporate directory tools**: some systems extract the favicon for branding.

For Bubbles client sites specifically, the favicon impacts:

* **Brand consistency across every visit.** A polished favicon signals operational maturity; a generic globe icon signals an unfinished site.
* **Google search results CTR.** A recognizable brand favicon next to the result improves click through. Studies place the CTR lift at 5 to 15% versus a missing or generic favicon.
* **Federal procurement officer visual recognition.** SAM.gov listings, LinkedIn shares, and email forwards all carry the favicon. Consistency matters for SDVOSB credibility.
* **Dark mode users.** Modern browsers (Chrome, Firefox, Safari) honor `prefers-color-scheme: dark`. Without a dark mode favicon, a dark colored logo disappears against dark browser tabs.
* **The thatdeveloperguy.com fleet identity.** 130+ client sites with proper favicons signal the agency's standard of craftsmanship.

The favicon is a one time setup with permanent brand value. Skipping it is unprofessional.

---

## 3. WHAT THIS COVERS

This framework covers:

* The five core icon link elements requested:
  * `<link rel="icon" href="/favicon.ico">` (legacy fallback).
  * `<link rel="icon" type="image/svg+xml" href="/icon.svg">` (modern SVG primary).
  * `<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">` (iOS home screen).
  * `<link rel="mask-icon" href="/icon.svg" color="#...">` (Safari pinned tab, mostly redundant in 2026).
  * `<link rel="manifest" href="/manifest.webmanifest">` (Web App Manifest reference).
* The PNG fallback family: `favicon-16x16.png` and `favicon-32x32.png` declared via `<link rel="icon" type="image/png" sizes="...">`.
* The Web App Manifest icons array with sizes, type, and `purpose: "maskable"` for Android adaptive icons.
* The dark mode SVG favicon pattern using `@media (prefers-color-scheme: dark)` inside the SVG `<style>` tag.
* The Safari pinned tab `mask-icon` mechanism and its 2026 status (mostly redundant because modern Safari uses the SVG favicon directly; still useful as a defensive belt and suspenders signal).
* The asset generation pipeline: starting from a high resolution source (1024x1024 SVG or PNG), generating the full icon family via ImageMagick `convert` or libvips.
* The aggressive browser caching that makes favicon updates frustrating, and the two reliable cache busting strategies (filename versioning and query parameter versioning).
* The Bubbles per client decision framework with the modern minimal stack as default.
* The federal and SDVOSB context with consistent brand favicon across the fleet.

This framework does NOT cover:

* The `apple-touch-icon` size family in depth (covered in framework-html-app-pwa-meta.md Section 14).
* The full Web App Manifest field reference (only the `icons` array; full reference in framework-pwa-manifest.md if separately developed).
* `<meta name="theme-color">` (covered in framework-html-meta-theme-color.md).
* `msapplication-TileImage` and the Microsoft tile family (covered in framework-html-app-pwa-meta.md Sections 11 through 13).
* PNG, SVG, ICO file format internals.
* Service worker icon registration (not relevant; service workers do not register icons).

---

## 4. THE MENTAL MODEL: MODERN MINIMAL STACK

In 2026, the modern minimal stack is five files plus a manifest reference:

```
THE FIVE FILES
   |
   |---> /favicon.ico                      (legacy fallback, multi size, ~10 KB)
   |---> /icon.svg                         (modern primary, scalable, ~2 KB)
   |---> /apple-touch-icon.png             (iOS home screen, 180x180, ~20 KB)
   |---> /icon-192.png                     (Android home screen, 192x192, referenced in manifest)
   |---> /icon-512.png                     (Android splash and install, 512x512, referenced in manifest)

THE HTML HEAD DECLARATIONS
   |
   |---> <link rel="icon" href="/favicon.ico" sizes="32x32">
   |---> <link rel="icon" type="image/svg+xml" href="/icon.svg">
   |---> <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
   |---> <link rel="manifest" href="/manifest.webmanifest">

THE MANIFEST
   |
   |---> {
            "icons": [
              { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
              { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
            ]
          }
```

Seven rules govern the system:

1. **Always include `/favicon.ico`** at the web root, regardless of how modern the rest of the stack is. RSS readers and tools fetch it directly.
2. **Use SVG as the modern primary** when possible. Crisp at any size, supports dark mode, small file size.
3. **Always include the 180x180 `apple-touch-icon.png`** for iOS home screen bookmarks.
4. **Include 192x192 and 512x512 PNG icons in the manifest** for Android home screen and PWA install prompts.
5. **Design the icon for legibility at 16x16.** If it falls apart at 16x16, redesign.
6. **Test in dark mode.** A dark logo on dark browser tabs is invisible.
7. **Version the filenames or query parameters** when updating; browsers cache favicons aggressively.

A correctly configured Bubbles client site has the four declarations in the `<head>`, the five files at the web root, the manifest with the icons array, and a dark mode aware SVG when the brand mark warrants it.

---

## 5. THE 2026 CONSENSUS ON THE MINIMAL STACK

The industry consensus in 2026 (per Favicon.io, FaviconStudio, Evil Martians, JW Tool Box, and Wux Web Tools published guidance) converges on:

| File | Purpose | Required? |
|---|---|---|
| `/favicon.ico` | Legacy fallback; RSS readers, AI unfurlers, tools | Yes |
| `/icon.svg` (or `/favicon.svg`) | Modern primary; dark mode support | Recommended |
| `/apple-touch-icon.png` (180x180) | iOS home screen | Yes |
| `/icon-192.png` | Android home screen (manifest) | Yes (PWA) or Recommended (non PWA) |
| `/icon-512.png` | Android splash + PWA install (manifest) | Yes (PWA) or Recommended (non PWA) |
| `/favicon-16x16.png` | Google search results, Safari fallback | Optional |
| `/favicon-32x32.png` | High DPI browser tabs, search results | Optional |
| `/safari-pinned-tab.svg` (mask icon) | Safari pinned tab (mostly redundant in 2026) | Optional |

The two competing schools of thought are:

* **Three files school** (Evil Martians, minimalist): favicon.ico, icon.svg, apple-touch-icon.png. Skip the 16x16 and 32x32 PNGs (the SVG handles those sizes). Skip the 192 and 512 unless explicitly going PWA.
* **Five files school** (most generator tools): include the 16x16 and 32x32 PNGs for maximum compatibility with platforms that don't yet handle SVG well (Google search results sometimes prefer PNG; some older RSS readers).

**For Bubbles convention**: the five files school. The cost of including two extra PNG files is trivial; the cost of debugging a missing icon in Google search results or a niche RSS reader is high.

---

## 6. THE LEGACY STACK HISTORY (AND WHY TO SKIP IT)

From approximately 2010 to 2018, generator tools (RealFaviconGenerator and others) produced massive icon sets:

```html
<!-- The historical exhaustive stack (DO NOT USE in 2026) -->
<link rel="apple-touch-icon" sizes="57x57" href="/apple-touch-icon-57x57.png">
<link rel="apple-touch-icon" sizes="60x60" href="/apple-touch-icon-60x60.png">
<link rel="apple-touch-icon" sizes="72x72" href="/apple-touch-icon-72x72.png">
<link rel="apple-touch-icon" sizes="76x76" href="/apple-touch-icon-76x76.png">
<link rel="apple-touch-icon" sizes="114x114" href="/apple-touch-icon-114x114.png">
<link rel="apple-touch-icon" sizes="120x120" href="/apple-touch-icon-120x120.png">
<link rel="apple-touch-icon" sizes="144x144" href="/apple-touch-icon-144x144.png">
<link rel="apple-touch-icon" sizes="152x152" href="/apple-touch-icon-152x152.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon-180x180.png">
<link rel="icon" type="image/png" sizes="192x192" href="/android-chrome-192x192.png">
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="96x96" href="/favicon-96x96.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
<link rel="manifest" href="/manifest.json">
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#6B3FA0">
<link rel="shortcut icon" href="/favicon.ico">
<!-- ...and more... -->
```

15+ link elements, 15+ image files at the web root, no functional benefit beyond what 5 files provide in 2026.

### 6.1 Why The Legacy Stack Existed

* iOS supported a wide range of older device resolutions, each with its own preferred icon size.
* SVG favicons were not supported by major browsers until 2019 (Chrome) and 2024 (Safari).
* PWA install dialogs had platform specific size requirements.

### 6.2 Why The Legacy Stack Is Obsolete

* iOS now uses 180x180 for all current devices; smaller sizes are rendered by downscaling.
* SVG favicons render crisply at any size in all major modern browsers.
* The Web App Manifest centralized PWA icon declarations into a single JSON file.

### 6.3 The Bubbles Convention

Skip the legacy stack. Use the modern minimal stack (Section 5). If migrating a client site that has the legacy stack, the cleanup is harmless (the extra files do not affect performance materially); but new builds should use the minimal stack from day one.

---

## 7. FAVICON.ICO (LEGACY FALLBACK)

The `/favicon.ico` file at the web root is the legacy fallback that every site should provide.

### 7.1 The Tag Format

```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
```

The `sizes="32x32"` attribute is technically not required for ICO files (ICO is a multi size container), but declaring it helps modern browsers prioritize the SVG over the ICO when the SVG is also declared.

### 7.2 The File Format

ICO is a Microsoft format that contains multiple image sizes in a single file. For the modern stack, two sizes inside the ICO:

* 16x16 (browser tab on standard DPI displays).
* 32x32 (browser tab on high DPI displays).

Older browsers and tools sometimes look for 48x48 too; include if generated from a single source costs nothing extra.

### 7.3 Why It Is Still Required In 2026

Despite SVG favicons being widely supported, `/favicon.ico` remains essential because:

* **RSS readers** (Feedly, Inoreader, NetNewsWire, NewsBlur, Reeder) fetch `/favicon.ico` directly without parsing the HTML.
* **AI assistant link unfurlers** fetch `/favicon.ico` for the small icon next to the unfurled card.
* **Browser bookmark imports** from older browsers expect ICO.
* **Email clients** (Outlook, Apple Mail) and some Slack and Discord unfurlers do the same.
* **Search engine crawlers** sometimes fetch `/favicon.ico` independently of the HTML.

If `/favicon.ico` is missing, these tools either show a generic icon or no icon at all.

### 7.4 The Browser Auto Discovery

Even if no `<link rel="icon">` declaration exists in the HTML, browsers automatically request `/favicon.ico` from the site root. This means:

* Placing the file at `/favicon.ico` is sufficient for legacy browsers.
* The `<link>` declaration is for explicit clarity and to declare additional sizes or formats.

### 7.5 The Bubbles Pattern

Always generate and place `/favicon.ico` at the web root for every client. No exceptions.

---

## 8. THE SVG FAVICON (MODERN PRIMARY)

The `<link rel="icon" type="image/svg+xml" href="/icon.svg">` is the modern primary favicon declaration.

### 8.1 The Tag Format

```html
<link rel="icon" type="image/svg+xml" href="/icon.svg">
```

The `type="image/svg+xml"` attribute is required; without it, browsers may not recognize the file as SVG.

### 8.2 Browser Support In 2026

* **Chrome**: supported since version 80 (2020).
* **Firefox**: supported since version 41 (2015).
* **Edge**: supported since the Chromium based rebuild (2020).
* **Safari**: supported since iOS 17 / macOS 14 (2023).
* **iOS Safari**: supported since iOS 17 (2023).

In 2026, SVG favicons work on essentially every modern browser. The remaining holdouts (legacy IE, ancient Android browsers) fall back to `/favicon.ico`.

### 8.3 The Advantages Of SVG

* **Scales perfectly to any size.** A single 1 KB SVG renders crisply at 16x16, 32x32, 48x48, and any higher density. No blurry upscaling.
* **Dark mode support via embedded CSS.** The SVG can change colors based on `prefers-color-scheme` (Section 9).
* **Smaller file size than equivalent PNG.** Most logo style favicons are 1 to 3 KB as SVG versus 5 to 20 KB as PNG.
* **Future proof.** New display densities arrive every year; SVG handles them all.

### 8.4 The Disadvantages

* **Complex SVGs render slowly at very small sizes.** A logo with fine line work may look fuzzy at 16x16.
* **Some platforms still prefer PNG.** Google search results sometimes display PNGs more reliably than SVGs.
* **Some browser caching quirks.** SVG favicon caching has had bugs historically.

### 8.5 The Design Constraint

Design the SVG for legibility at 16x16. If the brand mark has fine detail, simplify the favicon version. A square logomark, monogram, or single recognizable shape works better than a complex wordmark at small sizes.

### 8.6 The Minimal SVG

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100">
    <rect width="100" height="100" fill="#6B3FA0"/>
    <text x="50" y="68" font-family="Inter, sans-serif" font-size="60" font-weight="700" 
          text-anchor="middle" fill="#FFFFFF">T</text>
</svg>
```

A purple square with the letter "T" centered. Around 250 bytes. Renders crisply at any size.

### 8.7 The Bubbles Pattern

For every client, generate an SVG favicon from the brand source. Place at `/icon.svg` (or `/favicon.svg`; either name works). Declare in the HTML head.

---

## 9. THE DARK MODE SVG FAVICON PATTERN

SVG favicons support dark mode via embedded CSS media queries.

### 9.1 The Pattern

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100">
    <style>
        .fg { fill: #1A1A1A; }
        .bg { fill: #FFFFFF; }
        @media (prefers-color-scheme: dark) {
            .fg { fill: #FFFFFF; }
            .bg { fill: #1A1A1A; }
        }
    </style>
    <rect class="bg" width="100" height="100"/>
    <path class="fg" d="..."/>
</svg>
```

When the user's OS is in dark mode, the SVG swaps colors. Browser tabs in dark mode render the dark mode version automatically.

### 9.2 The Inversion Pattern

For a black on white logo:

* Light mode: black foreground, white background.
* Dark mode: white foreground, black background.

For a brand color logo (e.g., purple):

* Light mode: brand color foreground, white background.
* Dark mode: brand color foreground, dark background. (The brand color may need lightening for legibility on dark backgrounds.)

### 9.3 The Per Client Application

* **ThatDeveloperGuy**: brand purple on white in light mode, brand purple on dark in dark mode (purple lightened slightly for dark mode legibility).
* **TCB Fight Factory**: purple on black always (the brand mandates this).
* **Arkansas Counseling and Wellness**: warm brand color, with dark mode variant tested for legibility.
* **Handled Tax and Advisory**: professional brand color, dark mode variant.
* **WeCoverUSA**: brand identity, dark mode variant.

### 9.4 The Test

Open the SVG in a browser tab. Use the OS dark mode toggle (Settings → Appearance on macOS, Settings → Personalization → Colors on Windows). The favicon should remain legible in both modes.

### 9.5 The Bubbles Discipline

For every Bubbles client with a brand mark that risks visibility issues in dark mode (any black mark, any white mark, any low contrast mark): implement the dark mode pattern.

For brand marks that work equally well on light and dark backgrounds (high contrast color logos): the dark mode pattern is optional but harmless.

---

## 10. THE PNG FAMILY (16X16 AND 32X32)

The PNG fallback family covers platforms with imperfect SVG handling.

### 10.1 The Pattern

```html
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
```

Two PNG files. Two link elements.

### 10.2 Why Include Them If SVG Is The Primary

* **Google search results**: PNG favicons are more reliably picked up than SVG, especially on mobile result pages where the 16x16 favicon appears next to the page title.
* **Bing search results**: similar behavior.
* **Some Safari versions and older Android browsers**: imperfect SVG handling that gracefully falls back to PNG.
* **Belt and suspenders**: the cost is two small files; the protection covers edge cases.

### 10.3 The File Specs

* **favicon-16x16.png**: 16x16 pixels, PNG, ~500 bytes.
* **favicon-32x32.png**: 32x32 pixels, PNG, ~1 KB.

Both should be exact pixel renderings of the favicon at those sizes, not downscaled from a higher resolution.

### 10.4 The Generation Approach

For maximum visual quality:

* Design the favicon natively at 32x32 in a vector tool (Figma, Sketch, Inkscape).
* Export 16x16 and 32x32 PNGs from that source.
* The 16x16 export may need manual tweaking to remain legible at that size.

For acceptable visual quality:

* Generate from the SVG source using ImageMagick or libvips.
* Accept some loss of crispness at 16x16.

### 10.5 The Bubbles Pattern

Always include both PNG sizes alongside the SVG. The five file school (Section 5).

---

## 11. APPLE-TOUCH-ICON (IOS HOME SCREEN)

The 180x180 PNG icon used when a user adds the site to their iOS home screen.

### 11.1 The Tag Format

```html
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
```

The `sizes="180x180"` attribute is technically optional (iOS only uses 180x180 in modern versions) but explicit is better.

### 11.2 The File Specs

* **Dimensions**: 180x180 pixels exactly.
* **Format**: PNG.
* **Transparency**: avoid; use a solid color background. iOS applies its own rounded corners and effects.
* **File size**: ~10 to 30 KB.

### 11.3 The Filename Convention

`/apple-touch-icon.png` at the web root.

iOS Safari also auto discovers `/apple-touch-icon.png` even without the `<link>` declaration. Both belt and suspenders.

### 11.4 The Design Considerations

* Solid background (iOS adds rounded corners on top).
* Brand logo or monogram centered, with reasonable padding (around 10 to 15 pixels of padding on a 180x180 canvas).
* High contrast (the icon must be recognizable on both light and dark iOS home screens).

### 11.5 The Bubbles Pattern

Always include for every client. Even non PWA clients benefit: users sometimes bookmark sites to their iOS home screen for repeat access (Arkansas Counseling appointment booking, Handled Tax client portal entry, Heritage Hardwood Floors NWA quote request).

---

## 12. MASK-ICON (SAFARI PINNED TAB)

The `<link rel="mask-icon">` declaration provides a monochrome SVG for Safari's pinned tab feature on macOS.

### 12.1 The Tag Format

```html
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#6B3FA0">
```

Two attributes:

* `href`: a monochrome SVG (black silhouette on transparent background).
* `color`: the hex color Safari applies to the silhouette when the tab is pinned.

### 12.2 The Mechanism

When a user pins a tab in Safari on macOS:

* The pinned tab strip shows a small monochrome icon.
* Safari uses the `mask-icon` SVG silhouette, colored with the `color` attribute value.
* Active tab uses the brand color; inactive uses muted color.

### 12.3 The File Specs

The SVG should be:

* **A single color silhouette** (typically black or a single dark color).
* **Transparent background**.
* **Simple shape** that renders well at small sizes.

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16">
    <path fill="#000000" d="..."/>
</svg>
```

### 12.4 The 2026 Status

`mask-icon` is **mostly redundant in 2026** because:

* Safari since version 12 (macOS Mojave, 2018) gracefully falls back to the standard favicon for pinned tabs when `mask-icon` is missing.
* Safari since iOS 17 / macOS 14 (2023) uses the SVG favicon directly for pinned tabs (the dark mode aware SVG handles it).
* Pinned tabs are a less prominent feature in modern Safari workflows.

It is still useful as a defensive signal but not strictly necessary.

### 12.5 The Bubbles Decision

**For most Bubbles clients**: skip `mask-icon`. The modern SVG favicon handles Safari pinned tab rendering.

**For federal SDVOSB context** (ThatDeveloperGuy, WeCoverUSA) where comprehensive defensive coverage signals operational maturity: include `mask-icon` as belt and suspenders.

```html
<!-- Optional defensive declaration -->
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#6B3FA0">
```

### 12.6 The Reciprocal Compatibility Note

If `mask-icon` is included, ensure the `color` attribute matches the brand primary color. A mismatch produces a brand color violation on Safari pinned tabs.

---

## 13. THE MANIFEST LINK REFERENCE

The `<link rel="manifest" href="/manifest.webmanifest">` declaration points to the Web App Manifest file.

### 13.1 The Tag Format

```html
<link rel="manifest" href="/manifest.webmanifest">
```

The href can be `/manifest.webmanifest` (newer convention) or `/manifest.json` (older convention). Both work; pick one and be consistent across the site.

### 13.2 The MIME Type Requirement

Serve the manifest with MIME type `application/manifest+json`:

```nginx
location ~ \.webmanifest$ {
    add_header Content-Type "application/manifest+json";
}
```

For `/manifest.json`:

```nginx
location = /manifest.json {
    add_header Content-Type "application/manifest+json";
}
```

End every nginx change with:

```bash
nginx -t && systemctl reload nginx
```

### 13.3 The Relationship With This Framework

The full Web App Manifest field reference (`name`, `short_name`, `start_url`, `display`, `theme_color`, `background_color`, `scope`, etc.) was touched on in framework-html-app-pwa-meta.md Section 16. **This framework focuses specifically on the `icons` array** within the manifest (Section 14), which is the icon related portion of the manifest.

### 13.4 The Required Versus Optional Decision

* **For PWA enabled clients**: the manifest is required.
* **For non PWA clients**: the manifest is recommended but optional. Including it costs little and provides forward compatibility if PWA features are added later.

For Bubbles convention: include the manifest reference on all paying client sites. Even for non PWA clients, a minimal manifest with name and icons provides install metadata in some browser contexts.

---

## 14. THE WEB APP MANIFEST ICONS ARRAY

The `icons` array in the Web App Manifest declares the icons for Android home screens, PWA install prompts, and splash screens.

### 14.1 The Minimal Icons Array

```json
{
  "name": "Heritage Hardwood Floors NWA",
  "short_name": "Heritage HF",
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

Two icons: 192x192 and 512x512.

* **192x192**: Android home screen.
* **512x512**: Android splash screen and PWA install prompt.

### 14.2 The Maskable Icon Pattern

Modern Android uses adaptive icons that crop the icon to a system defined shape (circle, rounded square, squircle, etc.). The `purpose: "maskable"` field tells the OS the icon includes enough padding for safe cropping.

```json
{
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
    },
    {
      "src": "/icon-maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

The maskable icon is a separate file with the logo placed inside a "safe zone" centered in the canvas with ample padding around the edges. The OS may crop the outer 20% of the icon depending on the device's adaptive icon shape.

### 14.3 The Maskable Icon Safe Zone

For a 512x512 maskable icon:

* The "safe zone" is approximately the inner 80% (a circle of diameter 408 pixels centered in the canvas).
* The outer 20% may be cropped by the OS.
* The logo should fit entirely within the safe zone.

Maskable icon generators (like maskable.app) help validate the safe zone.

### 14.4 The Any Plus Maskable Pattern

Best practice in 2026 is to provide both `any` (default) and `maskable` icons:

```json
{
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icon-maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

* `purpose: "any"` icons are used when no cropping is applied.
* `purpose: "maskable"` icons are used by adaptive icon systems.
* If `purpose` is absent, the icon is treated as `any` by default.

### 14.5 The SVG In The Icons Array

The Web App Manifest can reference SVG icons:

```json
{
  "icons": [
    {
      "src": "/icon.svg",
      "sizes": "any",
      "type": "image/svg+xml"
    },
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    }
  ]
}
```

The `sizes: "any"` value indicates the SVG works at any size. Some Android systems prefer PNG anyway; including both is the comprehensive approach.

### 14.6 The Bubbles Manifest Icons Pattern

For full PWA clients:

```json
{
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

For non PWA clients with minimal manifest:

```json
{
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

---

## 15. THE MASKABLE ICON PATTERN (ANDROID ADAPTIVE ICONS)

A more detailed look at the maskable icon system.

### 15.1 The Problem It Solves

Android 8.0 (Oreo) and later use adaptive icons that crop app icons to a system defined shape. Without maskable awareness:

* Pixel devices may show the icon cropped to a circle, removing the corners.
* Samsung devices may show it cropped to a squircle.
* OnePlus devices may show a teardrop shape.

A non maskable icon may have its logo edges cropped, producing an unprofessional appearance on those devices.

### 15.2 The Solution

Provide a separate `purpose: "maskable"` icon with:

* The brand logo centered inside the canvas.
* At least 40 pixels of padding on all sides of a 512x512 canvas.
* A solid color background filling the entire canvas (the cropped corners show this color).

### 15.3 The Visual Example

```
Standard "any" icon (fills the entire 512x512 canvas):
+--------+
|  LOGO  |
+--------+

Maskable icon (logo inside safe zone, background fills edges):
+--------+
|  ssss  |   <- 's' is safe zone, padding fills corners
| sLOGOs |
|  ssss  |
+--------+
```

When Android crops the maskable version to a circle:

* The corners (filled with the background color) get cropped.
* The logo remains visible and intact in the center.

### 15.4 The Maskable.app Validator

Maskable.app is a free tool that validates maskable icons against the standard mask shapes. Upload the icon and verify the logo remains visible after all common mask shapes are applied.

### 15.5 The Bubbles Generation Approach

For PWA enabled Bubbles clients:

1. Generate `/icon-512.png` (standard, logo fills canvas).
2. Generate `/icon-maskable-512.png` (logo centered with 40 to 50 pixel padding, background color fills edges).
3. Declare both in the manifest icons array.

For non PWA clients: skip the maskable variant. The standard 192 and 512 icons are sufficient.

---

## 16. THE SIZES AND TYPE ATTRIBUTES

The `sizes` and `type` attributes on icon link elements help browsers select the right asset.

### 16.1 The sizes Attribute

```html
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
```

* **Single size**: `sizes="32x32"`.
* **Multiple sizes**: `sizes="16x16 32x32 48x48"` (space separated).
* **Any size (for SVG)**: `sizes="any"`.

The browser uses the sizes attribute to pick the most appropriate icon for the current rendering context.

### 16.2 The type Attribute

```html
<link rel="icon" type="image/svg+xml" href="/icon.svg">
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/x-icon" href="/favicon.ico">
```

Valid type values:

* `image/svg+xml` for SVG.
* `image/png` for PNG.
* `image/x-icon` or `image/vnd.microsoft.icon` for ICO (often omitted; the file extension communicates the type).
* `image/gif`, `image/jpeg` (rarely used for favicons).

### 16.3 The Priority Cascade

When multiple icon declarations exist, modern browsers select in this priority order:

1. SVG (if declared and supported).
2. PNG matching the requested size exactly.
3. PNG closest to the requested size (upscaled or downscaled).
4. ICO file.

Declaring all sizes explicitly lets the browser pick the best match without guessing.

### 16.4 The Bubbles Convention

Always include `sizes` and `type` on every icon link element. The cost is a few attributes; the benefit is deterministic icon selection.

---

## 17. THE ASSET GENERATION PIPELINE

For Bubbles, generate the icon family from a single high resolution source.

### 17.1 The Source Asset

Start from one of:

* A 1024x1024 PNG (square aspect ratio, transparent or solid background per brand).
* A vector SVG that can be exported at any size.

The source should be the highest quality available; downsampling produces good icons, upsampling produces blurry ones.

### 17.2 The ImageMagick Pipeline

```bash
#!/bin/bash
# generate-icons.sh
# Generate the full favicon family from a single source

SOURCE="icon-source-1024.png"
OUT_DIR="/var/www/[client]/html"

# SVG (if source is vector, copy directly; otherwise skip and use brand SVG)
# cp icon-source.svg "$OUT_DIR/icon.svg"

# favicon.ico (multi size: 16x16 and 32x32)
convert "$SOURCE" \
    \( -clone 0 -resize 16x16 \) \
    \( -clone 0 -resize 32x32 \) \
    -delete 0 \
    "$OUT_DIR/favicon.ico"

# PNG family for browser tabs and search results
convert "$SOURCE" -resize 16x16 "$OUT_DIR/favicon-16x16.png"
convert "$SOURCE" -resize 32x32 "$OUT_DIR/favicon-32x32.png"

# Apple touch icon (iOS home screen)
convert "$SOURCE" -resize 180x180 "$OUT_DIR/apple-touch-icon.png"

# Manifest icons (Android home screen and PWA install)
convert "$SOURCE" -resize 192x192 "$OUT_DIR/icon-192.png"
convert "$SOURCE" -resize 512x512 "$OUT_DIR/icon-512.png"

# Microsoft tile (legacy; covered in framework-html-app-pwa-meta.md)
convert "$SOURCE" -resize 144x144 "$OUT_DIR/tile-144.png"

# Maskable icon (with padding for adaptive icon safe zone)
# Logo at ~60% canvas size, brand color background filling edges
convert "$SOURCE" -resize 410x410 -background "#6B3FA0" -gravity center -extent 512x512 \
    "$OUT_DIR/icon-maskable-512.png"

echo "Icons generated in $OUT_DIR"
ls -la "$OUT_DIR"/*.png "$OUT_DIR"/*.ico
```

Per Joseph's preference: bash and shell over Python for HTML and asset generation. ImageMagick `convert` is the standard tool.

### 17.3 The libvips Alternative (Faster)

For high volume icon generation across the 130+ Bubbles fleet, libvips is significantly faster than ImageMagick:

```bash
# libvips equivalent
vips thumbnail "$SOURCE" "$OUT_DIR/favicon-32x32.png" 32
vips thumbnail "$SOURCE" "$OUT_DIR/apple-touch-icon.png" 180
vips thumbnail "$SOURCE" "$OUT_DIR/icon-192.png" 192
vips thumbnail "$SOURCE" "$OUT_DIR/icon-512.png" 512
```

Install on Bubbles:

```bash
sudo apt install libvips-tools
```

### 17.4 The SVG Source Handling

If the source is SVG, the ICO and PNG files are still rasterized from it. The SVG itself is copied directly to `/icon.svg`:

```bash
cp icon-source.svg "$OUT_DIR/icon.svg"

# Then generate the PNG family from a high resolution PNG rasterization
rsvg-convert -h 1024 icon-source.svg > /tmp/source.png
convert /tmp/source.png -resize 16x16 "$OUT_DIR/favicon-16x16.png"
# ... etc
```

### 17.5 The Deployment Step

After generation:

```bash
# Verify files exist
ls -la "$OUT_DIR"/{favicon.ico,icon.svg,favicon-*.png,apple-touch-icon.png,icon-192.png,icon-512.png,icon-maskable-512.png,tile-144.png}

# Validate nginx config (if any MIME type rules were added)
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx

# Test from outside Bubbles
curl -I https://[client-domain]/favicon.ico
curl -I https://[client-domain]/icon.svg
curl -I https://[client-domain]/apple-touch-icon.png
```

---

## 18. THE CACHING AND VERSIONING DISCIPLINE

Browsers cache favicons aggressively. An updated favicon may take days or weeks to appear in users' browsers without explicit cache busting.

### 18.1 The Problem

* Chrome caches favicons for the lifetime of the browser process and beyond.
* Firefox caches favicons indefinitely until explicit refresh.
* Safari behavior is the most aggressive; favicons can persist across browser updates.
* Edge: similar to Chrome.

When a Bubbles client updates their brand and the favicon changes, users see the old favicon until the cache evicts.

### 18.2 The Two Cache Busting Strategies

**Strategy 1: Query parameter versioning**

```html
<link rel="icon" href="/favicon.ico?v=2">
<link rel="icon" type="image/svg+xml" href="/icon.svg?v=2">
```

Pros: simple, requires only HTML change.
Cons: some tools (especially RSS readers) fetch `/favicon.ico` without query parameters and miss the version.

**Strategy 2: Filename versioning**

```html
<link rel="icon" href="/favicon-v2.ico">
<link rel="icon" type="image/svg+xml" href="/icon-v2.svg">
```

Plus keep `/favicon.ico` (without version) at the web root for legacy tools.

Pros: more reliable cache busting; tools that fetch `/favicon.ico` get the new one.
Cons: requires file rename and nginx update.

### 18.3 The Bubbles Recommendation

For initial deployment: no versioning needed (no prior cached version exists).

For favicon updates after initial deployment: use **query parameter versioning** as the default. Bump the version number in the HTML declarations. Update `/favicon.ico` and `/icon.svg` files in place.

For high stakes brand refreshes where every user must see the new icon promptly (federal context, major brand relaunch): use **filename versioning** combined with the unversioned legacy fallback.

### 18.4 The Cache Busting Documentation Pattern

For each client, document favicon cache history:

```
Favicon history:
  Initial deploy:  2025-08-15, /favicon.ico, /icon.svg, /apple-touch-icon.png
  Brand refresh:   2026-03-10, bumped to v=2 (?v=2 query parameters)
```

---

## 19. THE BUBBLES PER CLIENT DECISION FRAMEWORK

Each client needs favicon decisions.

### 19.1 The Decision Questions

1. **What is the brand mark?** Logo, monogram, wordmark, abstract symbol.
2. **Is the brand mark legible at 16x16?** If not, design a simplified favicon mark.
3. **What is the brand primary color?** Used for `mask-icon`, the favicon background, and the manifest theme color.
4. **Is dark mode awareness needed?** If the brand mark has visibility issues on dark backgrounds, implement the dark mode SVG pattern.
5. **Is the client a full PWA?** If yes, include `icon-maskable-512.png` and the manifest icons array with maskable purpose.

### 19.2 The Standard Modern Minimal Stack (Default For Every Client)

```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" type="image/svg+xml" href="/icon.svg">
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="manifest" href="/manifest.webmanifest">
```

Five files plus the manifest. Every Bubbles paying client.

### 19.3 The Per Client Recommendations

**ThatDeveloperGuy.com:**

Standard stack plus:

* Dark mode SVG (brand purple on white in light mode, brand purple lightened slightly on dark in dark mode).
* `mask-icon` for federal SDVOSB defensive coverage.

**ThatAIGuy.org:**

Standard stack plus dark mode SVG (AI/tech audience uses dark mode heavily).

**TCB Fight Factory:**

Standard stack. Brand purple and black mark on transparent SVG. Brand mandates purple and black only; the favicon reflects that.

**Arkansas Counseling and Wellness Services:**

Standard stack. Warm brand palette. Dark mode SVG for OS dark mode users.

**Handled Tax and Advisory:**

Standard stack. Professional financial palette.

**WeCoverUSA:**

Standard stack plus `mask-icon` for federal subcontractor defensive coverage.

**Eureka Bath Works:**

Standard stack.

**White River Cabins:**

Standard stack.

**Greenough's Guide Service:**

Standard stack.

**Heritage Hardwood Floors NWA:**

Standard stack. Wood tone brand color.

**Local Living Real Estate, Diana Undergust, Homes on Grand:**

Standard stack.

**Marshallese-Voices:**

Standard stack. Brand mark consistent across English and Marshallese versions; the favicon does not need language variants.

**Avid Pest Control, RAH Construction NWA, Steele Solutions, Blue Paradise Dairy, Pretty Clean NWA:**

Standard stack.

### 19.4 The PWA Enabled Client Addition

For clients identified as full PWA candidates (ThatAIGuy.org, Marshallese-Voices, potential TCB Fight Factory event app, potential Blue Paradise Dairy mobile commerce):

Add to the manifest:

```json
{
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

Plus generate `/icon-maskable-512.png` with proper safe zone padding.

### 19.5 The Documentation Pattern

For each client:

```
Favicon configuration:
  Source asset:      /var/www/[client]/source/icon-source-1024.png
  Brand color:       #6B3FA0
  Dark mode SVG:     [yes/no]
  PWA maskable:      [yes/no]
  
Files at web root:
  /favicon.ico
  /icon.svg
  /favicon-16x16.png
  /favicon-32x32.png
  /apple-touch-icon.png
  /icon-192.png
  /icon-512.png
  /tile-144.png  (Microsoft, per framework-html-app-pwa-meta.md)
  /icon-maskable-512.png  (PWA clients only)
  /manifest.webmanifest

Generation pipeline:
  /var/www/[client]/scripts/generate-icons.sh
  
Last regenerated:    [date]
```

---

## 20. THE SERVER SIDE FILE PLACEMENT REQUIREMENT

Unlike most HTML signals in this track, the favicon family is primarily a file placement concern, not a server side HTML rendering concern.

### 20.1 The File Placement

All icon files must be physically present at the web root (or wherever the `<link>` declarations point):

```
/var/www/[client]/html/
├── favicon.ico
├── icon.svg
├── favicon-16x16.png
├── favicon-32x32.png
├── apple-touch-icon.png
├── icon-192.png
├── icon-512.png
├── tile-144.png
├── icon-maskable-512.png  (PWA clients only)
├── manifest.webmanifest
└── ...
```

### 20.2 The HTTP Accessibility Requirement

Each file must be publicly accessible via HTTPS. Test:

```bash
curl -I https://[client-domain]/favicon.ico
curl -I https://[client-domain]/icon.svg
curl -I https://[client-domain]/apple-touch-icon.png
curl -I https://[client-domain]/manifest.webmanifest
```

Each should return HTTP 200 with appropriate Content-Type.

### 20.3 The MIME Type Configuration

nginx serves PNG, ICO, and SVG with correct MIME types by default. The manifest requires explicit configuration:

```nginx
location ~ \.webmanifest$ {
    add_header Content-Type "application/manifest+json";
}
```

### 20.4 The HTML Head Reference

After files are placed and accessible, the HTML head declarations reference them. The declarations are not strictly required for `/favicon.ico` (browsers auto discover it) but are required for everything else.

### 20.5 The Static HTML Versus Framework Stacks

* **Static HTML (Bubbles default)**: place files at the web root, reference in `<head>` directly. Trivial.
* **Next.js App Router**: place files in `/public/` directory; Next.js serves them from the root. Reference via the `metadata.icons` API.
* **SvelteKit**: place in `/static/` directory; same root serving behavior.
* **Astro**: place in `/public/` directory; same behavior.

In all cases, the files end up accessible at HTTPS root URLs.

---

## 21. IMPLEMENTATION PATTERNS BY STACK

### 21.1 Static HTML (Bubbles Default)

Direct in the `<head>`:

```html
<head>
    <meta charset="utf-8">
    <title>Page Title</title>

    <!-- Favicon and icons -->
    <link rel="icon" href="/favicon.ico" sizes="32x32">
    <link rel="icon" type="image/svg+xml" href="/icon.svg">
    <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
    <link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
    <link rel="manifest" href="/manifest.webmanifest">
</head>
```

### 21.2 Next.js 14 App Router

```typescript
// app/layout.tsx
export const metadata = {
  manifest: '/manifest.webmanifest',
  icons: {
    icon: [
      { url: '/favicon.ico', sizes: '32x32' },
      { url: '/icon.svg', type: 'image/svg+xml' },
      { url: '/favicon-32x32.png', type: 'image/png', sizes: '32x32' },
      { url: '/favicon-16x16.png', type: 'image/png', sizes: '16x16' },
    ],
    apple: [{ url: '/apple-touch-icon.png', sizes: '180x180', type: 'image/png' }],
  },
};
```

### 21.3 SvelteKit

```svelte
<!-- src/app.html -->
<head>
    <link rel="icon" href="/favicon.ico" sizes="32x32" />
    <link rel="icon" type="image/svg+xml" href="/icon.svg" />
    <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png" />
    <link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png" />
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
    <link rel="manifest" href="/manifest.webmanifest" />
</head>
```

### 21.4 Astro

```astro
<head>
    <link rel="icon" href="/favicon.ico" sizes="32x32" />
    <link rel="icon" type="image/svg+xml" href="/icon.svg" />
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
    <link rel="manifest" href="/manifest.webmanifest" />
</head>
```

### 21.5 The nginx MIME Type Configuration

```nginx
# Web App Manifest
location ~ \.webmanifest$ {
    add_header Content-Type "application/manifest+json";
}

# Long cache headers for icons (cache busting via versioning when updating)
location ~* \.(ico|png|svg)$ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

End with:

```bash
nginx -t && systemctl reload nginx
```

---

## 22. THE PRE LAUNCH AUDIT CHECKLIST

Before a Bubbles client site goes live.

### 22.1 The Checklist

* [ ] `/favicon.ico` exists at web root, accessible via HTTPS, returns 200.
* [ ] `/icon.svg` exists at web root, accessible via HTTPS, returns 200.
* [ ] `/favicon-16x16.png` exists at web root.
* [ ] `/favicon-32x32.png` exists at web root.
* [ ] `/apple-touch-icon.png` exists at web root, is 180x180 PNG.
* [ ] `/icon-192.png` exists at web root, is 192x192 PNG.
* [ ] `/icon-512.png` exists at web root, is 512x512 PNG.
* [ ] `/manifest.webmanifest` exists, validates as JSON, includes icons array.
* [ ] Manifest served with `application/manifest+json` MIME type.
* [ ] All icon files served with appropriate MIME types (image/x-icon, image/svg+xml, image/png).
* [ ] HTML head includes the full link element block.
* [ ] Browser tab favicon renders correctly in Chrome, Firefox, Safari, Edge.
* [ ] iOS Safari "Add to Home Screen" produces clean 180x180 icon.
* [ ] Android Chrome "Add to Home Screen" produces clean 192x192 icon.
* [ ] Dark mode SVG renders correctly (if implemented) in dark mode browser tabs.
* [ ] Favicon is legible at 16x16 (not blurred, not collapsed).
* [ ] Brand color in the favicon matches the brand primary.
* [ ] Maskable icon (if implemented) validates on maskable.app.
* [ ] Google search results test (Google Search Console URL Inspection) shows expected favicon.

### 22.2 The curl Verification

```bash
# Check each file
curl -I https://example.com/favicon.ico
curl -I https://example.com/icon.svg
curl -I https://example.com/apple-touch-icon.png
curl -I https://example.com/icon-192.png
curl -I https://example.com/icon-512.png
curl -I https://example.com/manifest.webmanifest

# Confirm manifest MIME type
curl -I https://example.com/manifest.webmanifest | grep -i content-type

# Confirm HTML head references all files
curl -s https://example.com/ | grep -iE '<link rel="(icon|apple-touch-icon|mask-icon|manifest)"'
```

### 22.3 The Browser Test

* Open https://example.com/ in Chrome, Firefox, Safari, Edge.
* Verify the favicon appears in the browser tab.
* Toggle OS dark mode and verify the favicon remains legible.
* On macOS Safari, try pinning the tab; verify the pinned tab icon renders correctly.
* On iOS Safari, add to Home Screen; verify the icon and label appear correctly.
* On Android Chrome, add to Home Screen; verify the maskable icon renders correctly.

---

## 23. THE COMMON FAILURES AND FIXES

### 23.1 "The favicon doesn't appear in the browser tab"

Cause 1: No `/favicon.ico` at the web root.
Fix: Generate and place the file.

Cause 2: `/favicon.ico` returns 404 due to nginx misconfiguration.
Fix: Verify accessibility via curl.

Cause 3: Browser caching an old failed request (no icon).
Fix: Hard refresh (Ctrl+Shift+R or Cmd+Shift+R) or open in private browsing window.

### 23.2 "The favicon looks pixelated or blurry"

Cause 1: Generated from a low resolution source.
Fix: Regenerate from a 1024x1024 or larger source.

Cause 2: The design has too much fine detail for 16x16.
Fix: Simplify the favicon design (use a monogram, single shape, or simplified mark).

### 23.3 "Google search results show a generic globe icon instead of our favicon"

Cause 1: Google has not yet recrawled the site since the favicon was added.
Fix: Submit the URL for re crawl in Search Console.

Cause 2: Google specifically requires PNG favicons (16x16 or 32x32) for search results.
Fix: Ensure `/favicon-32x32.png` is declared via `<link rel="icon" type="image/png" sizes="32x32">`.

Cause 3: The favicon URL is in robots.txt disallow.
Fix: Allow `/favicon.ico` and all icon files in robots.txt.

### 23.4 "The iOS home screen icon shows a white square around the logo"

Cause: The `apple-touch-icon.png` has transparency.
Fix: Add a solid color background (brand color or white). iOS applies its own rounded corners.

### 23.5 "The favicon won't update no matter how many times we refresh"

Cause: Aggressive browser caching.
Fix 1: Cache busting query parameter (`?v=2`) in the HTML declarations.
Fix 2: Filename versioning (`/icon-v2.svg`).
Fix 3: Hard refresh and clear browser cache. Test in private browsing.

### 23.6 "The dark mode browser tab shows the logo invisibly on dark background"

Cause: Static SVG without dark mode awareness; the brand mark is dark on light.
Fix: Implement the dark mode SVG pattern (Section 9) with `@media (prefers-color-scheme: dark)` style.

### 23.7 "The Android home screen icon is showing cropped or has white edges"

Cause: No maskable icon declared.
Fix: Generate `/icon-maskable-512.png` with proper safe zone padding and add to manifest with `purpose: "maskable"`.

### 23.8 "The manifest is not loading; PWA install prompt does not appear"

Cause 1: `<link rel="manifest">` is missing.
Fix: Add the link element.

Cause 2: Manifest MIME type is wrong.
Fix: Configure nginx to serve `.webmanifest` with `application/manifest+json`.

Cause 3: Manifest JSON has syntax errors.
Fix: Validate with `jq` or an online manifest validator.

Cause 4: Manifest icons array is missing required sizes.
Fix: Include 192x192 and 512x512 at minimum.

### 23.9 "Safari pinned tab is showing the wrong color"

Cause 1: `mask-icon` color attribute is wrong.
Fix: Set color to brand primary hex.

Cause 2: `mask-icon` is missing entirely; Safari falls back to standard favicon.
Fix: Optionally add `mask-icon` for explicit control.

### 23.10 "AI assistants and RSS readers show no favicon"

Cause: Tools fetch `/favicon.ico` directly without parsing HTML; the file is missing.
Fix: Always include `/favicon.ico` at the web root.

---

## 24. THE FEDERAL AND SDVOSB CONTEXT

For Joseph's SDVOSB federal subcontracting work, favicon discipline reinforces brand consistency.

### 24.1 The Federal Audience Recognition

Federal procurement officers and contracting officers see the favicon in:

* Browser tabs when reviewing vendor sites.
* LinkedIn share previews (the favicon appears as the small icon next to the unfurled card).
* Email signatures and corporate directory tools.
* SAM.gov related searches.

Consistent brand mark across every surface signals operational maturity.

### 24.2 The ThatDeveloperGuy SDVOSB Pattern

```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" type="image/svg+xml" href="/icon.svg">
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#6B3FA0">
<link rel="manifest" href="/manifest.webmanifest">
```

Full defensive stack including mask-icon. Brand purple consistent across every declared file.

Dark mode SVG implementation ensures professional appearance in dark mode terminal and IDE environments where federal technical evaluators may be reviewing the site.

### 24.3 The WeCoverUSA Federal Pattern

Same full defensive stack. National scope brand identity.

### 24.4 The Documentation Pattern

For federal context client work, document the favicon asset library alongside other compliance documentation:

```
Federal vendor visual identity:
  Primary brand color:     #6B3FA0
  Favicon source:          /var/www/thatdeveloperguy.com/source/icon-source-1024.png
  Dark mode SVG:           yes (brand purple lightens to #8E63C7 in dark mode)
  mask-icon:               yes (Safari pinned tab defensive coverage)
  SAM.gov consistency:     URL and brand mark cross referenced
  SDVOSB certification:    [details]
```

---

## 25. QUICK REFERENCE TAG BLOCK

For copy paste into any Bubbles client page `<head>`:

### 25.1 Standard Modern Minimal Stack (Every Client)

```html
<!-- Favicon and icons -->
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" type="image/svg+xml" href="/icon.svg">
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="manifest" href="/manifest.webmanifest">
```

### 25.2 With Federal SDVOSB Defensive Coverage

```html
<!-- Federal defensive: includes mask-icon -->
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" type="image/svg+xml" href="/icon.svg">
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#BRAND_HEX">
<link rel="manifest" href="/manifest.webmanifest">
```

### 25.3 Minimal Manifest.webmanifest (icons array only)

```json
{
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### 25.4 PWA Enabled Manifest.webmanifest

```json
{
  "name": "Client Full Name",
  "short_name": "Short Label",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#BRAND_HEX",
  "background_color": "#FFFFFF",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

### 25.5 Dark Mode SVG Favicon Template

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100">
    <style>
        .bg { fill: #FFFFFF; }
        .fg { fill: #6B3FA0; }
        @media (prefers-color-scheme: dark) {
            .bg { fill: #0F0F0F; }
            .fg { fill: #8E63C7; }
        }
    </style>
    <rect class="bg" width="100" height="100"/>
    <text class="fg" x="50" y="70" font-family="Inter, sans-serif" font-size="60" 
          font-weight="800" text-anchor="middle">T</text>
</svg>
```

### 25.6 The nginx Snippet

```nginx
# Web App Manifest MIME type
location ~ \.webmanifest$ {
    add_header Content-Type "application/manifest+json";
}

# Icon caching
location ~* \.(ico|png|svg)$ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

End with:

```bash
nginx -t && systemctl reload nginx
```

---

## 26. REFERENCES

* W3C Web App Manifest specification (icons section): https://www.w3.org/TR/appmanifest/#icons-member
* MDN apple-touch-icon: https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/apple-touch-icon
* MDN mask-icon: https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/mask-icon
* Maskable icon validator: https://maskable.app
* Favicon.io generator: https://favicon.io
* Evil Martians How to Favicon in 2026: https://evilmartians.com/chronicles/how-to-favicon-in-2021-six-files-that-fit-most-needs
* Google Search Central favicon guidelines: https://developers.google.com/search/docs/appearance/favicon-in-search

**Companion frameworks in the HTML signal track:**

* framework-html-meta-robots.md
* framework-html-meta-charset.md
* framework-html-meta-viewport.md
* framework-html-meta-description.md
* framework-html-meta-keywords.md
* framework-html-meta-author.md
* framework-html-meta-generator.md
* framework-html-meta-copyright.md
* framework-html-meta-theme-color.md (the chrome color that frames the icons)
* framework-html-meta-color-scheme.md
* framework-html-meta-referrer.md
* framework-html-content-language.md
* framework-html-meta-refresh.md
* framework-html-open-graph.md
* framework-html-twitter-cards.md
* framework-html-search-engine-verification.md
* framework-html-app-pwa-meta.md (direct companion: apple-touch-icon and manifest reference covered there too)
* framework-html-canonical-alternate.md

**Forward references:**

* framework-pwa-manifest.md (deeper Web App Manifest field reference if separately developed).
* framework-pwa-service-worker.md (service worker layer for full PWA).

**Parent reference:** UNIVERSAL-RANKING-FRAMEWORK.md (the 27 section master document).

---

*End of framework-html-favicon-icons.md.*
