# framework-html-app-pwa-meta.md

Comprehensive reference for the App and Progressive Web App (PWA) meta tag family, the `<meta>` tags that control how a website behaves when bookmarked to a mobile home screen, installed as a standalone app, displayed in Windows tile UI (legacy), or surfaced as an installable application in iOS Safari and Chrome on Android. Covers the iOS specific tags (`apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style`, `apple-mobile-web-app-title`), the Android and Chrome modern equivalent (`mobile-web-app-capable`), the cross platform brand identity tag (`application-name`), the Microsoft Windows tile tags (`msapplication-TileColor`, `msapplication-TileImage`, `msapplication-config` with the companion `browserconfig.xml` file), the related apple-touch-icon and apple-touch-startup-image link elements, and the Web App Manifest (`manifest.json` or `manifest.webmanifest`) as the canonical replacement for nearly all of these tags. Covers the critical 2026 deprecation status (Chrome DevTools warns about `apple-mobile-web-app-capable` and recommends `mobile-web-app-capable`; the Web App Manifest standard is the W3C canonical path) versus the practical reality (iOS Safari still uses these meta tags as fallback when the manifest fails to load, and `apple-touch-startup-image` only works when `apple-mobile-web-app-capable` is present), the iOS PWA platform constraints (push notifications added in iOS 16.4 for installed PWAs only, standalone mode removed in the EU under the Digital Markets Act in iOS 17.4, default web app mode in iOS 26, all browsers on iOS forced through WebKit), the Microsoft tile deprecation (Windows 11 dropped Live Tiles, so `msapplication-*` tags are vestigial in 2026 but still parsed by legacy systems), and the Bubbles per client decision framework with the standard minimal pattern (theme-color plus apple-touch-icon plus application-name) as default and the full PWA pattern (manifest plus service worker plus full installable app meta block) only when a client has explicit installable app requirements. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the seventeenth framework in the HTML signal track**, following the meta robots, charset, viewport, description, keywords, author, generator, copyright, theme-color, color-scheme, referrer, content-language, refresh, Open Graph, Twitter Cards, and search engine verification frameworks. Companion to framework-html-meta-theme-color.md (the standalone PWA chrome color signal) and framework-html-favicon-icons.md (the icon family).

Audience: humans implementing installable PWA experiences for client sites, AI assistants generating HTML head sections that balance modern manifest based PWA with iOS Safari fallback meta tags, designers producing the icon and tile asset library, Bubbles operators making the per client PWA versus standard site decision, federal subcontractors evaluating whether a PWA installable experience adds value or risk, and anyone troubleshooting "the iOS home screen icon looks pixelated", "Chrome DevTools shows a deprecation warning about apple-mobile-web-app-capable", "the status bar shows black on white instead of the brand color", "the installed app title shows the full page title instead of the brand name", or "the Windows tile is using a default Microsoft purple instead of our brand color".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters (And Why It Matters Less In 2026)
3. What This Covers
4. The Mental Model: Manifest Canonical, Meta Tag Fallback
5. The 2026 Deprecation Status
6. apple-mobile-web-app-capable
7. apple-mobile-web-app-status-bar-style
8. apple-mobile-web-app-title
9. mobile-web-app-capable
10. application-name
11. msapplication-TileColor
12. msapplication-TileImage
13. msapplication-config And browserconfig.xml
14. The apple-touch-icon Family
15. The apple-touch-startup-image Family
16. The Web App Manifest (Canonical Path)
17. The iOS PWA Platform Constraints In 2026
18. The Microsoft Tile Deprecation In Windows 11
19. The Bubbles Per Client Decision Framework
20. The Standard Minimal Pattern
21. The Full PWA Pattern
22. Implementation Patterns By Stack
23. The Pre Launch Audit Checklist
24. The Common Failures And Fixes
25. The Federal And SDVOSB Context
26. Quick Reference Tag Block
27. References

---

## 1. DEFINITION

The App and PWA meta tag family is a set of `<meta>` tags that control the installable, home screened, and tile based representations of a website across iOS Safari, Chrome on Android, Microsoft Edge on Windows, and (historically) Internet Explorer pinned sites.

```html
<meta name="application-name" content="Heritage Hardwood Floors NWA">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-title" content="Heritage HF">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="mobile-web-app-capable" content="yes">
<meta name="msapplication-TileColor" content="#6B3FA0">
<meta name="msapplication-TileImage" content="/tile-144.png">
<meta name="msapplication-config" content="/browserconfig.xml">
```

These tags collectively answer five questions:

1. What name does the operating system display when the site is bookmarked or installed?
2. Should the site behave as a standalone app (no browser chrome) or as a regular bookmarked URL?
3. What color appears in the iOS status bar when the site runs in standalone mode?
4. What icon and tile assets represent the site on the home screen, Windows Start menu, and lock screen?
5. What color background frames the Microsoft tile and what brand identity is presented?

In 2026, the canonical answer to all of these questions is the **Web App Manifest** (`manifest.json` or `manifest.webmanifest`), a JSON file referenced via `<link rel="manifest">`. The meta tags in this framework are now considered legacy or fallback signals. They remain useful because iOS Safari falls back to them when the manifest cannot be loaded, and `apple-touch-startup-image` for splash screens only works when `apple-mobile-web-app-capable` is also present.

---

## 2. WHY IT MATTERS (AND WHY IT MATTERS LESS IN 2026)

These tags mattered enormously from 2010 to 2018 when iOS Safari did not support the Web App Manifest. They mattered moderately from 2018 to 2023 as Safari added partial manifest support but kept the meta tag fallback chain. They matter less in 2026 because:

* **The Web App Manifest is the canonical standard.** Chrome on Android, Edge, Firefox, and Safari (since iOS 16.4) all use the manifest as primary.
* **Chrome DevTools warns about `apple-mobile-web-app-capable`** as deprecated and recommends `mobile-web-app-capable` instead (which itself is largely redundant when a manifest is present).
* **Microsoft Live Tiles were dropped in Windows 11**, making `msapplication-TileColor`, `msapplication-TileImage`, and `msapplication-config` vestigial.
* **iOS 17.4 removed standalone PWA mode in the EU** under the Digital Markets Act compliance. PWAs in EU countries open in Safari tabs.

Despite the deprecation, the tags still matter because:

* **iOS Safari falls back to these meta tags** when the manifest cannot be loaded.
* **`apple-touch-startup-image` requires `apple-mobile-web-app-capable`** to function; without the meta tag, custom splash screens do not appear.
* **`application-name`** is still used by Chrome on Android, Edge, and Firefox in some bookmark and share contexts.
* **The status bar style on iOS** is still controlled by `apple-mobile-web-app-status-bar-style` even when the manifest is present.

For Bubbles client sites specifically, the App and PWA tags impact:

* **Home screen appearance** when a customer bookmarks the site on iPhone or iPad. The icon, title, and status bar treatment signal whether the brand is professional or unfinished.
* **Brand identity in search results** on mobile (some browsers use `application-name`).
* **Windows Start menu tile** for the small fraction of users who pin sites in Edge on Windows 10 (legacy).
* **Future installable app upgrade path.** If a Bubbles client wants to upgrade to a full PWA (offline support, push notifications, installable), having the foundational tags in place reduces the gap.

For most local service Bubbles clients (Heritage Hardwood Floors NWA, Eureka Bath Works, Greenough's Guide Service, White River Cabins), a full PWA is not warranted. The minimal pattern (covered in Section 20) is sufficient to ensure professional appearance on home screen bookmarks without the operational overhead of a service worker and manifest.

---

## 3. WHAT THIS COVERS

This framework covers:

* The six core App and PWA meta tags requested:
  * `apple-mobile-web-app-capable` (iOS standalone mode, deprecated but still functional).
  * `apple-mobile-web-app-status-bar-style` (iOS status bar treatment in standalone mode).
  * `apple-mobile-web-app-title` (iOS home screen icon label).
  * `mobile-web-app-capable` (Chrome on Android standalone mode, the modern equivalent of the Apple tag).
  * `application-name` (the cross platform brand identity for the site).
  * `msapplication-TileColor` (Windows tile background color, largely deprecated in Windows 11).
  * `msapplication-TileImage` (Windows tile image, largely deprecated in Windows 11).
  * `msapplication-config` (pointer to `browserconfig.xml`, largely deprecated in Windows 11).
* The related and complementary signals:
  * `<link rel="apple-touch-icon">` (iOS home screen icon).
  * `<link rel="apple-touch-startup-image">` (iOS standalone mode splash screen).
  * `<link rel="manifest">` (the Web App Manifest, the canonical replacement).
  * `<meta name="theme-color">` (already covered in framework-html-meta-theme-color.md).
* The 2026 deprecation status of each tag.
* The iOS PWA platform constraints (WebKit monopoly, EU DMA standalone removal, push notification gating, storage quota differences).
* The Microsoft tile deprecation history.
* The Web App Manifest as canonical path with the minimal manifest example.
* The Bubbles per client decision framework.
* The standard minimal pattern (always include these on every Bubbles client).
* The full PWA pattern (only for clients with explicit installable app requirements).

This framework does NOT cover:

* `<meta name="theme-color">` — covered separately in framework-html-meta-theme-color.md.
* `<meta name="color-scheme">` — covered in framework-html-meta-color-scheme.md.
* Service worker implementation, push notification handling, or offline strategy — covered in framework-pwa-service-worker.md.
* Full favicon family (favicon.ico, multiple PNG sizes, SVG, mask icon) — covered in framework-html-favicon-icons.md.

---

## 4. THE MENTAL MODEL: MANIFEST CANONICAL, META TAG FALLBACK

In 2026, the canonical PWA configuration model is:

```
LAYER 1: Web App Manifest (the canonical signal)
   |
   |---> /manifest.webmanifest    (linked via <link rel="manifest">)
         {
           "name": "Full app name",
           "short_name": "Short label",
           "icons": [...],
           "start_url": "/",
           "display": "standalone",
           "theme_color": "#6B3FA0",
           "background_color": "#FFFFFF"
         }

LAYER 2: iOS Safari Fallback Meta Tags (because Safari is partial)
   |
   |---> apple-mobile-web-app-capable          (enables standalone + splash screen)
   |---> apple-mobile-web-app-title            (icon label; manifest short_name preferred but Safari behavior varies)
   |---> apple-mobile-web-app-status-bar-style (iOS specific, no manifest equivalent)
   |---> apple-touch-icon link elements        (icon assets)
   |---> apple-touch-startup-image link elements (splash screens, requires apple-mobile-web-app-capable)

LAYER 3: Chrome Android Modern Equivalent (largely redundant with manifest)
   |
   |---> mobile-web-app-capable                (replaces Apple tag for Chrome warnings)

LAYER 4: Cross Platform Identity
   |
   |---> application-name                      (still used in some browser bookmark UI)

LAYER 5: Microsoft Windows Tile (largely deprecated)
   |
   |---> msapplication-TileColor               (Windows 10 Live Tile background; vestigial in Windows 11)
   |---> msapplication-TileImage               (Windows 10 Live Tile image; vestigial in Windows 11)
   |---> msapplication-config                  (pointer to browserconfig.xml; vestigial)
```

Five rules govern the system:

1. **The Web App Manifest is the primary signal** for any installable PWA experience in 2026.
2. **iOS Safari fallback meta tags must accompany the manifest** because Safari's manifest support is partial and the apple-touch-startup-image splash screen system requires `apple-mobile-web-app-capable`.
3. **`mobile-web-app-capable` is required to silence Chrome DevTools warnings** but provides no functional benefit when a manifest is present.
4. **`application-name` is harmless to include** and provides a brand identity signal across multiple browser bookmark contexts.
5. **Microsoft tile tags are optional in 2026**; they cost almost nothing to include but provide value only on the small Windows 10 Edge legacy surface.

A correctly configured Bubbles client site in 2026 has at minimum the standard minimal pattern (Section 20): `application-name`, `apple-mobile-web-app-title`, `apple-touch-icon`, and `theme-color`. A full PWA client adds the manifest, service worker, and the complete iOS fallback stack.

---

## 5. THE 2026 DEPRECATION STATUS

| Tag | Status | Practical Reality |
|---|---|---|
| `apple-mobile-web-app-capable` | Deprecated (Chrome warning since 2024) | Still required for iOS splash screens; iOS Safari fallback chain. **Keep.** |
| `apple-mobile-web-app-status-bar-style` | Active | No manifest equivalent; iOS only signal for status bar style. **Keep.** |
| `apple-mobile-web-app-title` | Active | Manifest `short_name` preferred but Safari behavior varies. **Keep.** |
| `mobile-web-app-capable` | Active (the modern replacement for the Apple tag) | Largely redundant when manifest is present; silences Chrome DevTools warning. **Include if no manifest.** |
| `application-name` | Active | Still used in browser bookmark UI on some platforms. **Keep.** |
| `msapplication-TileColor` | Deprecated in Windows 11 | Vestigial; harmless. **Keep if Microsoft tile target audience exists, otherwise optional.** |
| `msapplication-TileImage` | Deprecated in Windows 11 | Same as above. **Keep if target audience exists.** |
| `msapplication-config` | Deprecated in Windows 11 | Same. **Keep if target audience exists.** |
| `<link rel="manifest">` | Active (canonical PWA signal) | The W3C standard. **Use as the primary PWA configuration.** |
| `<link rel="apple-touch-icon">` | Active | iOS home screen icon. **Keep.** |
| `<link rel="apple-touch-startup-image">` | Active | iOS splash screen, requires `apple-mobile-web-app-capable`. **Keep for PWA clients.** |

The practical conclusion for Bubbles: include the working subset, omit the strictly vestigial ones unless the client's target audience justifies them.

---

## 6. APPLE-MOBILE-WEB-APP-CAPABLE

The original iOS standalone app mode declaration.

### 6.1 The Tag Format

```html
<meta name="apple-mobile-web-app-capable" content="yes">
```

The only valid `content` value is `yes`. Absence or any other value means the site behaves as a normal bookmarked URL (opens in Safari with browser chrome).

### 6.2 What It Does

When set to `yes` and the user adds the site to their home screen on iOS:

* The site opens in standalone mode (no Safari address bar, no navigation chrome).
* The site behaves like an installed app.
* The `apple-touch-startup-image` splash screens are activated.
* The `apple-mobile-web-app-status-bar-style` setting takes effect.

### 6.3 The Deprecation Warning

Chrome DevTools displays this warning when the tag is present:

> `<meta name="apple-mobile-web-app-capable" content="yes">` is deprecated. Please include `<meta name="mobile-web-app-capable" content="yes">`.

The W3C recommendation: use the Web App Manifest instead of either meta tag.

### 6.4 The Practical Conflict

The deprecation warning is technically correct but operationally incomplete:

* iOS Safari's Web App Manifest support is partial.
* `apple-touch-startup-image` (the splash screen system) only functions when `apple-mobile-web-app-capable` is present, regardless of manifest presence.
* Removing the tag breaks splash screens on iOS.

**For Bubbles convention**: keep the tag and add `mobile-web-app-capable` alongside it to silence the Chrome warning.

### 6.5 The Bubbles Pattern

```html
<!-- Both, for compatibility and warning suppression -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
```

Include on PWA enabled clients. Omit on standard non installable client sites.

---

## 7. APPLE-MOBILE-WEB-APP-STATUS-BAR-STYLE

The iOS status bar style when the site runs in standalone mode.

### 7.1 The Tag Format

```html
<meta name="apple-mobile-web-app-status-bar-style" content="default">
```

Three valid `content` values:

* `default` — white background, black text. The default if the tag is absent.
* `black` — black background, white text. The status bar has a black background.
* `black-translucent` — black translucent overlay. The web content extends behind the status bar, and the status bar appears as a black translucent overlay.

### 7.2 The Common Misunderstanding

The `content` value does NOT set the status bar color to that color. It selects from three predefined styles:

* `default` produces a white status bar.
* `black` produces a black status bar.
* `black-translucent` produces a transparent status bar over the web content.

**There is no way to set the iOS status bar to a brand color via this tag.** For brand color status bar treatment, use `<meta name="theme-color">` (covered in framework-html-meta-theme-color.md), which has more sophisticated control via the Web App Manifest and media query support.

### 7.3 The Selection Decision

* **`default` (white, black text)**: appropriate for sites with light header backgrounds.
* **`black` (black, white text)**: appropriate for sites with dark header backgrounds.
* **`black-translucent`**: appropriate for sites that want to control the full viewport, with the web content extending behind the status bar. Requires the site to add CSS padding to avoid content collision with the status bar.

### 7.4 The Bubbles Pattern

For most Bubbles clients with light header design:

```html
<meta name="apple-mobile-web-app-status-bar-style" content="default">
```

For TCB Fight Factory (purple and black brand) with dark header:

```html
<meta name="apple-mobile-web-app-status-bar-style" content="black">
```

For full bleed designs (rare):

```html
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
```

Then add CSS top padding to the body to prevent content from being obscured by the status bar.

### 7.5 The Manifest Relationship

The Web App Manifest does not have a direct equivalent. The closest is the `theme_color` and `background_color` fields, which Safari uses for the standalone mode chrome but not specifically for status bar style.

**For Bubbles convention**: include this tag explicitly when implementing a PWA. It is the only way to control iOS status bar style.

---

## 8. APPLE-MOBILE-WEB-APP-TITLE

The label displayed under the icon when the site is added to the iOS home screen.

### 8.1 The Tag Format

```html
<meta name="apple-mobile-web-app-title" content="Heritage HF">
```

The `content` value is the label text. Should be short (10 to 12 characters maximum) because iOS truncates longer labels.

### 8.2 What It Does

When a user adds the site to their iOS home screen via Safari's "Add to Home Screen" option:

* iOS uses this value as the icon label.
* If absent, iOS uses the `<title>` element of the page, which is usually too long and ends up truncated.

### 8.3 The Length Constraint

* iOS home screen icon labels truncate at approximately 11 characters.
* Longer labels are truncated with ellipsis.
* For best appearance: 4 to 10 characters.

### 8.4 The Manifest Relationship

The Web App Manifest `short_name` field serves the same purpose. iOS Safari (per Apple's partial manifest support) may use either the meta tag or the manifest `short_name`, with the meta tag taking precedence in some Safari versions.

**For Bubbles convention**: include both the meta tag and the manifest `short_name` with identical values.

### 8.5 The Per Client Examples

| Client | apple-mobile-web-app-title | Reason |
|---|---|---|
| Heritage Hardwood Floors NWA | `Heritage HF` | Full name truncates badly |
| Arkansas Counseling and Wellness Services | `ARC Wellness` | Same |
| Handled Tax and Advisory | `Handled Tax` | Fits |
| ThatDeveloperGuy | `TDG` or `Joseph A` | Initials or first name |
| TCB Fight Factory | `TCB` | Brand acronym |
| White River Cabins | `WR Cabins` | Abbreviated |
| Greenough's Guide Service | `Greenough` | Last name only |
| Eureka Bath Works | `Eureka Bath` | Fits |
| Local Living Real Estate | `Local Living` | Fits |
| Marshallese-Voices | `Voices` | Short label preserves dignity |

### 8.6 The Bubbles Pattern

```html
<meta name="apple-mobile-web-app-title" content="[SHORT_BRAND_LABEL]">
```

On every Bubbles client. Even for clients that are not PWA enabled, the tag ensures professional appearance for users who add the site to their home screen.

---

## 9. MOBILE-WEB-APP-CAPABLE

The modern Chrome on Android equivalent of `apple-mobile-web-app-capable`. Introduced to replace the Apple specific tag and silence the Chrome DevTools deprecation warning.

### 9.1 The Tag Format

```html
<meta name="mobile-web-app-capable" content="yes">
```

The only valid `content` value is `yes`.

### 9.2 What It Does (And Doesn't)

When set to `yes`:

* Chrome on Android may treat the site as installable (though manifest based installability is the canonical path).
* The Chrome DevTools deprecation warning about `apple-mobile-web-app-capable` is silenced (when `mobile-web-app-capable` is added alongside).

What it does NOT do:

* It is not a substitute for the Web App Manifest. Chrome on Android prefers manifest based installability.
* It does not affect iOS Safari behavior (which uses `apple-mobile-web-app-capable`).

### 9.3 The Bubbles Pattern

Include alongside `apple-mobile-web-app-capable` when both are needed:

```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
```

For non installable client sites: omit both.

For full PWA clients with manifest: the manifest is the primary signal. Include both meta tags to suppress warnings and ensure iOS fallback.

---

## 10. APPLICATION-NAME

The site's brand identity name, used in various browser bookmark and share UIs.

### 10.1 The Tag Format

```html
<meta name="application-name" content="Heritage Hardwood Floors NWA">
```

The `content` value is the full application or site name. Unlike `apple-mobile-web-app-title`, no length constraint.

### 10.2 What It Does

* Chrome on Android: used in some bookmark and share contexts.
* Edge on Windows: used in some pinned site contexts.
* Firefox: respected for application name display in some contexts.
* Safari: largely ignored (uses `apple-mobile-web-app-title` and manifest `name`).

### 10.3 The Relationship With `<title>`

`<title>` is the page title (different per page). `application-name` is the site name (same on every page).

```html
<title>Hardwood Floor Refinishing | Bella Vista AR | Heritage Hardwood Floors NWA</title>
<meta name="application-name" content="Heritage Hardwood Floors NWA">
```

### 10.4 The Bubbles Pattern

```html
<meta name="application-name" content="[FULL_CLIENT_BUSINESS_NAME]">
```

On every page of every Bubbles client. The full business name, identical to `og:site_name` and the manifest `name` field.

### 10.5 The Per Client Map

| Client | application-name |
|---|---|
| ThatDeveloperGuy.com | `ThatDeveloperGuy` |
| ThatAIGuy.org | `ThatAIGuy` |
| TCB Fight Factory | `TCB Fight Factory` |
| Arkansas Counseling and Wellness Services | `Arkansas Counseling and Wellness Services` |
| Handled Tax and Advisory | `Handled Tax & Advisory` |
| WeCoverUSA | `WeCoverUSA` |
| Heritage Hardwood Floors NWA | `Heritage Hardwood Floors NWA` |
| Eureka Bath Works | `Eureka Bath Works` |
| White River Cabins | `White River Cabins` |
| Greenough's Guide Service | `Greenough's Guide Service` |
| Marshallese-Voices | `Marshallese Voices` |

---

## 11. MSAPPLICATION-TILECOLOR

The Microsoft Windows Live Tile background color, used when the site is pinned to the Windows 10 Start menu.

### 11.1 The Tag Format

```html
<meta name="msapplication-TileColor" content="#6B3FA0">
```

The `content` value is a hex color code. Should match the brand primary color.

### 11.2 The Deprecation Status

Microsoft removed Live Tiles from Windows 11 (released October 2021). The tag is parsed by Windows 10 Edge legacy and Internet Explorer pinned sites but has no effect on Windows 11.

In 2026, Windows 11 is the dominant version. The tag is vestigial.

### 11.3 The Selection Decision

The brand primary color:

* ThatDeveloperGuy: `#6B3FA0` (brand purple).
* TCB Fight Factory: `#6B3FA0` (purple) or `#000000` (black) per brand guidance.
* Arkansas Counseling: warm brand primary (TBD per client brand book).
* Heritage Hardwood Floors NWA: warm wood tone or brand color.

### 11.4 The Bubbles Pattern

Include for completeness; cost is one tag. Skip if minimizing HTML weight is a priority and Windows 10 audience is negligible.

```html
<meta name="msapplication-TileColor" content="#6B3FA0">
```

---

## 12. MSAPPLICATION-TILEIMAGE

The Microsoft Windows Live Tile image, used when the site is pinned to the Windows 10 Start menu.

### 12.1 The Tag Format

```html
<meta name="msapplication-TileImage" content="/tile-144.png">
```

The `content` value is the URL of a 144x144 pixel PNG image (transparent background recommended).

### 12.2 The Image Specs

* Dimensions: 144x144 pixels.
* Format: PNG with transparent background.
* The image should be the brand logo on a transparent canvas; the tile color comes from `msapplication-TileColor`.

### 12.3 The Deprecation Status

Same as `msapplication-TileColor`: vestigial in Windows 11.

### 12.4 The Bubbles Pattern

If including:

```html
<meta name="msapplication-TileImage" content="/tile-144.png">
```

Include the tile image asset in the client asset library.

---

## 13. MSAPPLICATION-CONFIG AND BROWSERCONFIG.XML

For more advanced Microsoft tile configuration, an XML configuration file at the web root provides multiple tile sizes.

### 13.1 The Tag Format

```html
<meta name="msapplication-config" content="/browserconfig.xml">
```

### 13.2 The browserconfig.xml Format

```xml
<?xml version="1.0" encoding="utf-8"?>
<browserconfig>
    <msapplication>
        <tile>
            <square70x70logo src="/tile-70.png"/>
            <square150x150logo src="/tile-150.png"/>
            <square310x310logo src="/tile-310.png"/>
            <wide310x150logo src="/tile-310x150.png"/>
            <TileColor>#6B3FA0</TileColor>
        </tile>
    </msapplication>
</browserconfig>
```

The XML provides multiple tile sizes for Windows 10's Live Tile layouts (small, medium, large, wide).

### 13.3 The Deprecation Status

Same as the other Microsoft tile tags: vestigial in Windows 11. The XML file is parsed by Windows 10 Edge legacy but has no effect in Windows 11.

### 13.4 The Bubbles Pattern

For maximum completeness on Windows 10 audience:

```html
<meta name="msapplication-config" content="/browserconfig.xml">
```

Plus the `browserconfig.xml` file at the web root.

For Bubbles convention: skip unless Windows 10 audience is meaningful for the client.

---

## 14. THE APPLE-TOUCH-ICON FAMILY

The iOS home screen icon family, declared via `<link rel="apple-touch-icon">` elements (not meta tags, but tightly related to the App and PWA system).

### 14.1 The Basic Pattern

```html
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
```

Default size: 180x180 pixels PNG with no transparency (iOS applies its own rounded corners and effects).

### 14.2 The Sizes Family

```html
<link rel="apple-touch-icon" sizes="57x57" href="/apple-touch-icon-57x57.png">
<link rel="apple-touch-icon" sizes="60x60" href="/apple-touch-icon-60x60.png">
<link rel="apple-touch-icon" sizes="72x72" href="/apple-touch-icon-72x72.png">
<link rel="apple-touch-icon" sizes="76x76" href="/apple-touch-icon-76x76.png">
<link rel="apple-touch-icon" sizes="114x114" href="/apple-touch-icon-114x114.png">
<link rel="apple-touch-icon" sizes="120x120" href="/apple-touch-icon-120x120.png">
<link rel="apple-touch-icon" sizes="144x144" href="/apple-touch-icon-144x144.png">
<link rel="apple-touch-icon" sizes="152x152" href="/apple-touch-icon-152x152.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon-180x180.png">
```

In 2026, iOS uses the 180x180 version on all current devices. The smaller sizes are legacy.

### 14.3 The Minimum Recommended Set

```html
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
```

One file, 180x180 PNG, named `apple-touch-icon.png` at the web root. iOS Safari will find it automatically even without the `<link>` element.

### 14.4 The Bubbles Convention

* Generate a 180x180 PNG for every client.
* Place at `/apple-touch-icon.png` at the web root.
* Declare via `<link rel="apple-touch-icon">` in the `<head>`.

---

## 15. THE APPLE-TOUCH-STARTUP-IMAGE FAMILY

The iOS standalone mode splash screen family. Only functional when `apple-mobile-web-app-capable` is `yes`.

### 15.1 The Pattern

```html
<link rel="apple-touch-startup-image" 
      media="(device-width: 390px) and (device-height: 844px) and (-webkit-device-pixel-ratio: 3)" 
      href="/splash-390x844.png">
```

Each iOS device size requires a separate splash image with a media query targeting that exact dimension.

### 15.2 The Practical Reality

Producing the full set of iOS splash screen sizes requires generating dozens of images (one per current iPhone and iPad screen size). Tools like `pwa-asset-generator` automate this.

### 15.3 The Bubbles Pattern

For PWA enabled clients only. Generate via tooling. Reference via the full media query stack in the HTML head.

For non PWA clients: skip splash screens entirely. The minimal pattern (Section 20) does not include splash images.

---

## 16. THE WEB APP MANIFEST (CANONICAL PATH)

The Web App Manifest is the W3C standard JSON file that replaces nearly all of the App and PWA meta tags in 2026.

### 16.1 The Reference

```html
<link rel="manifest" href="/manifest.webmanifest">
```

Place in the `<head>` of every page (or at least the home page).

### 16.2 The Minimal Manifest

```json
{
  "name": "Heritage Hardwood Floors NWA",
  "short_name": "Heritage HF",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#6B3FA0",
  "background_color": "#FFFFFF",
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

### 16.3 The Field Map

| Manifest field | Meta tag equivalent |
|---|---|
| `name` | `application-name` |
| `short_name` | `apple-mobile-web-app-title` |
| `display: "standalone"` | `apple-mobile-web-app-capable="yes"` + `mobile-web-app-capable="yes"` |
| `theme_color` | `theme-color` meta tag |
| `background_color` | (loosely, msapplication-TileColor) |
| `icons` | apple-touch-icon link elements |
| `start_url` | (no meta tag equivalent) |
| `scope` | (no meta tag equivalent) |

### 16.4 The MIME Type

Serve `manifest.webmanifest` with MIME type `application/manifest+json`. Nginx configuration:

```nginx
location ~ \.webmanifest$ {
    add_header Content-Type "application/manifest+json";
}
```

Or use `manifest.json` with MIME type `application/json`.

### 16.5 The Bubbles Pattern

For full PWA enabled clients:

```html
<link rel="manifest" href="/manifest.webmanifest">
```

Plus the JSON file at the web root.

For non PWA clients: skip the manifest. The iOS Safari fallback meta tags handle basic home screen bookmark behavior without needing a full manifest.

---

## 17. THE IOS PWA PLATFORM CONSTRAINTS IN 2026

Apple's WebKit monopoly on iOS shapes the PWA implementation strategy.

### 17.1 The Constraint Timeline

* **2018**: Safari adds partial Web App Manifest support.
* **2023 (iOS 16.4)**: Web Push API arrives. Push notifications work for installed PWAs only (not in Safari tabs).
* **2024 (iOS 17.4)**: EU only. Apple removes standalone PWA mode under Digital Markets Act compliance. PWAs in EU countries open in Safari tabs without standalone behavior or push.
* **2025 (iOS 18.4)**: Declarative Web Push API.
* **2026 (iOS 26)**: Default web app mode improvements.

### 17.2 The 2026 Capability Map

| Capability | iOS 26 Status |
|---|---|
| Add to Home Screen | Supported |
| Standalone mode | Supported (outside EU) |
| Service workers | Supported with limits |
| Web Push notifications | Supported for installed PWAs only (outside EU) |
| Background sync | Not supported |
| Hardware APIs (Bluetooth, USB, NFC) | Mostly not supported |
| Storage quotas | Tighter than Chrome on Android |
| Offline caching | Supported |
| `manifest.webmanifest` | Partial support |
| `apple-mobile-web-app-capable` fallback | Still used |

### 17.3 The EU Implication

For clients with EU audience, standalone PWA mode is unavailable on iOS. Plan for graceful degradation: the site should function correctly in a regular Safari tab even if standalone mode is the intent.

For Joseph's current Bubbles clients, EU audience is negligible (almost all clients are NWA and SWMO local services). EU DMA constraints do not materially affect implementation.

### 17.4 The WebKit Monopoly

All browsers on iOS (Chrome, Firefox, Edge, Brave) use WebKit. There is no Chromium based browser engine on iOS. PWA features must align with WebKit's capability set regardless of browser brand.

### 17.5 The Strategic Implication For Bubbles

Full PWA implementation on iOS in 2026 is meaningful only when:

* Offline access matters (rare for Bubbles client local services).
* Push notifications are wanted (only relevant for ongoing engagement, rare).
* Home screen install reduces friction for repeat customers.

For most Bubbles clients (one time service providers, local services), the standard minimal pattern is sufficient. Full PWA adds operational complexity without proportional business benefit.

---

## 18. THE MICROSOFT TILE DEPRECATION IN WINDOWS 11

Microsoft Live Tiles were removed from Windows 11 (released October 2021).

### 18.1 The History

* **Windows 8 (2012)**: Live Tiles introduced as the primary Start menu UI.
* **Windows 10 (2015)**: Live Tiles continued; `msapplication-*` tags fully supported.
* **Windows 11 (2021)**: Live Tiles removed. Start menu redesigned without tiles. `msapplication-*` tags no longer have visible effect.

### 18.2 The 2026 Audience Math

Windows 11 has been the dominant Windows version for several years. Windows 10 reaches end of mainstream support in October 2025; extended security updates continue but the user base shifts to Windows 11.

For Bubbles clients in 2026: the Microsoft tile target audience is small and shrinking. Including the tags costs three meta tags plus an optional `browserconfig.xml`; omitting them costs essentially nothing.

### 18.3 The Bubbles Convention

* **Include `msapplication-TileColor` and `msapplication-TileImage`** for completeness on professional client sites (federal SDVOSB context, YMYL).
* **Skip `msapplication-config`** unless the client has explicit Windows 10 tile customization needs (very rare).

---

## 19. THE BUBBLES PER CLIENT DECISION FRAMEWORK

Each client site needs an App and PWA decision.

### 19.1 The Decision Questions

1. **Is the client a candidate for a full installable PWA?** If yes, implement the manifest plus full iOS fallback stack plus service worker.
2. **Does the client benefit from professional home screen bookmark appearance?** If yes, implement the standard minimal pattern.
3. **Is the brand color and dark mode treatment specified?** If yes, set `apple-mobile-web-app-status-bar-style` accordingly.
4. **Is the Windows 10 audience meaningful?** If yes, include the Microsoft tile tags.

### 19.2 The Client Categorization

**Standard local services (no PWA needed):**

Heritage Hardwood Floors NWA, Eureka Bath Works, Greenough's Guide Service, White River Cabins, Avid Pest Control, RAH Construction NWA, Steele Solutions, Pretty Clean NWA, Local Living Real Estate, Diana Undergust, Homes on Grand.

**Standard minimal pattern** (Section 20). No manifest, no service worker.

**YMYL or federal context (no PWA but polished appearance):**

ThatDeveloperGuy.com, WeCoverUSA, Arkansas Counseling and Wellness Services, Handled Tax and Advisory.

**Standard minimal pattern plus Microsoft tile tags** for completeness. Federal subcontracting context benefits from comprehensive meta tag stack.

**PWA candidates (full installable app):**

* ThatAIGuy.org (MEGAMIND interface, AI engagement encourages return visits).
* Marshallese-Voices (community engagement, repeat visits, low data environments may benefit from offline caching).
* TCB Fight Factory (if event app or training schedule app is added).
* Blue Paradise Dairy (if mobile commerce app experience is added).

**Full PWA pattern** (Section 21).

### 19.3 The Documentation Pattern

For each client, document the App and PWA configuration:

```
App and PWA configuration:
  application-name:                       Heritage Hardwood Floors NWA
  apple-mobile-web-app-title:             Heritage HF
  apple-mobile-web-app-capable:           (omitted; non PWA)
  apple-mobile-web-app-status-bar-style:  default
  mobile-web-app-capable:                 (omitted; non PWA)
  msapplication-TileColor:                #6B3FA0
  msapplication-TileImage:                /tile-144.png
  
  apple-touch-icon:                       /apple-touch-icon.png (180x180)
  manifest:                               (omitted; non PWA)
  theme-color:                            #6B3FA0
```

---

## 20. THE STANDARD MINIMAL PATTERN

For Bubbles clients without explicit PWA requirements, the standard minimal pattern provides professional home screen bookmark appearance without operational overhead.

### 20.1 The Tag Stack

```html
<!-- Brand identity -->
<meta name="application-name" content="[FULL_CLIENT_NAME]">
<meta name="apple-mobile-web-app-title" content="[SHORT_CLIENT_LABEL]">

<!-- iOS status bar style (no PWA mode, but still respected for some bookmark contexts) -->
<meta name="apple-mobile-web-app-status-bar-style" content="default">

<!-- Theme color (cross platform brand color) -->
<meta name="theme-color" content="[BRAND_PRIMARY_COLOR]">

<!-- iOS home screen icon -->
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">

<!-- Microsoft tile (vestigial but harmless) -->
<meta name="msapplication-TileColor" content="[BRAND_PRIMARY_COLOR]">
<meta name="msapplication-TileImage" content="/tile-144.png">

<!-- Optional: favicon family (covered in framework-html-favicon-icons.md) -->
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
```

### 20.2 What This Provides

* Polished iOS home screen icon when users bookmark the site.
* Brand color theme on Chrome on Android address bar (via theme-color).
* Brand color Windows 10 Start menu tile (legacy audience).
* Brand identity in browser bookmark UI.

### 20.3 What This Does NOT Provide

* Standalone mode (the site opens in Safari with browser chrome).
* Service worker offline caching.
* Push notifications.
* Web App Manifest features.

This is intentional. For one time service businesses, the operational overhead of full PWA is not warranted.

---

## 21. THE FULL PWA PATTERN

For Bubbles clients that need installable app behavior.

### 21.1 The Tag Stack

```html
<!-- Brand identity -->
<meta name="application-name" content="ThatAIGuy">
<meta name="apple-mobile-web-app-title" content="ThatAIGuy">

<!-- Standalone mode (iOS and Chrome) -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">

<!-- Theme color -->
<meta name="theme-color" content="#6B3FA0">

<!-- iOS home screen icon -->
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">

<!-- iOS splash screens (generated per device size) -->
<link rel="apple-touch-startup-image"
      media="(device-width: 430px) and (device-height: 932px) and (-webkit-device-pixel-ratio: 3)"
      href="/splash-430x932.png">
<!-- (... additional splash screen entries per device size ...) -->

<!-- Web App Manifest (the canonical PWA signal) -->
<link rel="manifest" href="/manifest.webmanifest">

<!-- Microsoft tile (still useful for Edge installable) -->
<meta name="msapplication-TileColor" content="#6B3FA0">
<meta name="msapplication-TileImage" content="/tile-144.png">

<!-- Favicon family -->
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png">
```

### 21.2 The Required Companion Assets

* `/manifest.webmanifest` (the JSON file).
* `/icon-192.png` and `/icon-512.png` (manifest icons).
* `/apple-touch-icon.png` (iOS home screen, 180x180).
* `/tile-144.png` (Windows tile).
* `/favicon-*.png` (favicon family).
* iOS splash screen images per device size.
* Service worker (`/sw.js`) for offline caching and push notifications.
* HTTPS (required for service workers and PWA features).

### 21.3 The Service Worker

Required for installability on Chrome and for offline behavior. Implementation covered in framework-pwa-service-worker.md (separate from this framework).

### 21.4 The Validation

After deployment, validate the PWA setup:

* Chrome DevTools → Application tab → Manifest. Confirms manifest is loaded and valid.
* Chrome DevTools → Application tab → Service Workers. Confirms service worker is registered.
* Chrome DevTools → Lighthouse → PWA audit. Targets 90+ score.
* iOS Safari → Add to Home Screen. Confirms standalone mode and splash screen render correctly.

---

## 22. IMPLEMENTATION PATTERNS BY STACK

### 22.1 Static HTML (Bubbles Default)

Direct in the `<head>` of every page, or via a shared include.

```html
<!doctype html>
<html lang="en-US">
<head>
    <meta charset="utf-8">
    <title>Page Title</title>

    <!-- App and PWA -->
    <meta name="application-name" content="Heritage Hardwood Floors NWA">
    <meta name="apple-mobile-web-app-title" content="Heritage HF">
    <meta name="apple-mobile-web-app-status-bar-style" content="default">
    <meta name="theme-color" content="#6B3FA0">
    <meta name="msapplication-TileColor" content="#6B3FA0">
    <meta name="msapplication-TileImage" content="/tile-144.png">
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
</head>
```

### 22.2 Next.js 14 App Router

```typescript
// app/layout.tsx
export const metadata = {
  applicationName: 'Heritage Hardwood Floors NWA',
  appleWebApp: {
    title: 'Heritage HF',
    statusBarStyle: 'default',
    capable: false,  // omit for non PWA
  },
  themeColor: '#6B3FA0',
  manifest: '/manifest.webmanifest',  // only for full PWA
  icons: {
    apple: [{ url: '/apple-touch-icon.png', sizes: '180x180' }],
  },
  other: {
    'msapplication-TileColor': '#6B3FA0',
    'msapplication-TileImage': '/tile-144.png',
  },
};
```

### 22.3 SvelteKit

```svelte
<!-- src/app.html -->
<head>
    <meta name="application-name" content="Heritage Hardwood Floors NWA">
    <meta name="apple-mobile-web-app-title" content="Heritage HF">
    <meta name="apple-mobile-web-app-status-bar-style" content="default">
    <meta name="theme-color" content="#6B3FA0">
    <meta name="msapplication-TileColor" content="#6B3FA0">
    <meta name="msapplication-TileImage" content="/tile-144.png">
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
</head>
```

### 22.4 Astro

```astro
---
const brand = {
  name: 'Heritage Hardwood Floors NWA',
  shortName: 'Heritage HF',
  themeColor: '#6B3FA0',
};
---
<head>
    <meta name="application-name" content={brand.name} />
    <meta name="apple-mobile-web-app-title" content={brand.shortName} />
    <meta name="apple-mobile-web-app-status-bar-style" content="default" />
    <meta name="theme-color" content={brand.themeColor} />
    <meta name="msapplication-TileColor" content={brand.themeColor} />
    <meta name="msapplication-TileImage" content="/tile-144.png" />
    <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
</head>
```

### 22.5 The Bubbles Asset Pipeline

For each client:

```bash
# Generate the icon family from a single high resolution source
# Source: /client-brand/icon-source-1024.png (or SVG)

# Generate apple-touch-icon
convert icon-source.png -resize 180x180 apple-touch-icon.png

# Generate Microsoft tile
convert icon-source.png -resize 144x144 -background transparent tile-144.png

# Generate manifest icons (full PWA only)
convert icon-source.png -resize 192x192 icon-192.png
convert icon-source.png -resize 512x512 icon-512.png

# Generate favicon family (covered in framework-html-favicon-icons.md)
convert icon-source.png -resize 32x32 favicon-32x32.png
convert icon-source.png -resize 16x16 favicon-16x16.png
```

Place all assets at the web root. Deploy. Run:

```bash
nginx -t && systemctl reload nginx
```

---

## 23. THE PRE LAUNCH AUDIT CHECKLIST

Before a Bubbles client site goes live, verify the App and PWA stack.

### 23.1 The Checklist (Standard Minimal Pattern)

* [ ] `application-name` present with full business name.
* [ ] `apple-mobile-web-app-title` present with short label (4 to 10 characters).
* [ ] `apple-mobile-web-app-status-bar-style` set appropriately for header design.
* [ ] `theme-color` matches brand primary color.
* [ ] `apple-touch-icon` 180x180 PNG exists at web root and is referenced.
* [ ] `msapplication-TileColor` matches brand primary.
* [ ] `msapplication-TileImage` 144x144 PNG exists at web root and is referenced.
* [ ] Favicon family present (separate framework).
* [ ] iOS Safari "Add to Home Screen" test produces clean icon and label.

### 23.2 The Checklist (Full PWA Pattern)

All of the above plus:

* [ ] `manifest.webmanifest` file at web root with all required fields.
* [ ] `manifest.webmanifest` served with MIME type `application/manifest+json`.
* [ ] `<link rel="manifest">` references the manifest file.
* [ ] `apple-mobile-web-app-capable` set to `yes`.
* [ ] `mobile-web-app-capable` set to `yes`.
* [ ] iOS splash screens generated for current device sizes.
* [ ] Service worker registered and active.
* [ ] HTTPS enabled (required for PWA).
* [ ] Chrome DevTools Lighthouse PWA audit scores 90+.
* [ ] iOS Add to Home Screen produces standalone mode with custom splash.

### 23.3 The curl Verification

```bash
curl -s https://example.com/ | grep -iE 'name="(application-name|apple-mobile-web-app|mobile-web-app-capable|msapplication-Tile|theme-color)"'
```

Returns each tag. Confirms server side rendering.

---

## 24. THE COMMON FAILURES AND FIXES

### 24.1 "Chrome DevTools shows a deprecation warning about apple-mobile-web-app-capable"

Cause: The Apple tag is present without the modern `mobile-web-app-capable` counterpart.
Fix: Add `<meta name="mobile-web-app-capable" content="yes">` alongside the Apple tag.

### 24.2 "The iOS home screen icon looks pixelated"

Cause 1: The `apple-touch-icon.png` is smaller than 180x180.
Fix: Provide 180x180 PNG.

Cause 2: iOS applied transparency or rounded corners awkwardly to the source.
Fix: Use a solid square PNG (no transparency); iOS applies its own rounded corners.

Cause 3: The image is being scaled up from a low resolution source.
Fix: Generate from a high resolution source (1024x1024 or larger).

### 24.3 "The home screen icon label is truncated"

Cause: `apple-mobile-web-app-title` is too long, or the `<title>` is being used as fallback.
Fix: Shorten to 4 to 10 characters.

### 24.4 "The iOS status bar shows the wrong style in standalone mode"

Cause: `apple-mobile-web-app-status-bar-style` is set to a value that conflicts with the design.
Fix: Choose the appropriate value (`default`, `black`, or `black-translucent`).

### 24.5 "The Windows tile is using the default Microsoft purple instead of the brand color"

Cause: `msapplication-TileColor` is missing or set incorrectly.
Fix: Set the hex color matching the brand.

Cause 2: Windows 11 audience; tiles no longer render.
Fix: Acknowledge the deprecation; no effective fix.

### 24.6 "The iOS splash screen does not appear"

Cause 1: `apple-mobile-web-app-capable` is missing or not set to `yes`.
Fix: Add the tag (required for splash screens even when manifest is present).

Cause 2: The splash image URL is wrong or returns 404.
Fix: Verify image exists and the media query matches the device.

Cause 3: The media query targets the wrong device dimensions.
Fix: Regenerate splash screens for current iPhone and iPad sizes.

### 24.7 "The manifest is not being recognized by Chrome"

Cause 1: `<link rel="manifest">` is missing.
Fix: Add the link element.

Cause 2: The manifest file returns wrong MIME type.
Fix: Configure nginx to serve `.webmanifest` with `application/manifest+json`.

Cause 3: The manifest JSON has syntax errors.
Fix: Validate with `jq` or an online manifest validator.

### 24.8 "The PWA installs on Chrome but not iOS"

Cause: iOS does not support automatic install prompts. Users must manually "Add to Home Screen" from the Share menu.
Fix: Add an in app prompt instructing iOS users to use Share → Add to Home Screen.

### 24.9 "The PWA worked yesterday but standalone mode broke today"

Cause 1: A theme update removed `apple-mobile-web-app-capable`.
Fix: Restore the tag.

Cause 2: User is in the EU and updated to iOS 17.4+ which removed standalone mode under DMA.
Fix: Acknowledge the platform constraint; no fix available.

---

## 25. THE FEDERAL AND SDVOSB CONTEXT

For Joseph's SDVOSB federal subcontracting work, the App and PWA tags carry minimal direct weight but contribute to overall professional appearance.

### 25.1 The Federal Audience Mobile Behavior

Federal procurement officers and contracting officers occasionally bookmark vendor sites for reference. A polished home screen icon and clean status bar treatment signal operational maturity.

### 25.2 The Federal Convention

For ThatDeveloperGuy.com and WeCoverUSA:

```html
<meta name="application-name" content="ThatDeveloperGuy">
<meta name="apple-mobile-web-app-title" content="TDG">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="theme-color" content="#6B3FA0">
<meta name="msapplication-TileColor" content="#6B3FA0">
<meta name="msapplication-TileImage" content="/tile-144.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
```

Comprehensive minimal stack. No full PWA (operational overhead not justified for federal vendor site).

### 25.3 The Documentation

For federal context, document the icon asset library and the meta tag stack alongside other compliance documentation (SAM.gov registration, SDVOSB certification, site verification per framework-html-search-engine-verification.md).

---

## 26. QUICK REFERENCE TAG BLOCK

For copy paste into any Bubbles client page `<head>`:

### 26.1 Standard Minimal Pattern (Most Clients)

```html
<!-- App and PWA (minimal) -->
<meta name="application-name" content="FULL_CLIENT_NAME">
<meta name="apple-mobile-web-app-title" content="SHORT_LABEL">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="theme-color" content="#BRAND_HEX">
<meta name="msapplication-TileColor" content="#BRAND_HEX">
<meta name="msapplication-TileImage" content="/tile-144.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
```

### 26.2 Full PWA Pattern (PWA Enabled Clients)

```html
<!-- App and PWA (full) -->
<meta name="application-name" content="FULL_CLIENT_NAME">
<meta name="apple-mobile-web-app-title" content="SHORT_LABEL">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="theme-color" content="#BRAND_HEX">
<meta name="msapplication-TileColor" content="#BRAND_HEX">
<meta name="msapplication-TileImage" content="/tile-144.png">
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png">
<link rel="manifest" href="/manifest.webmanifest">

<!-- iOS splash screens (one entry per device size; generate via tooling) -->
<link rel="apple-touch-startup-image"
      media="(device-width: 430px) and (device-height: 932px) and (-webkit-device-pixel-ratio: 3)"
      href="/splash-430x932.png">
<!-- ... etc ... -->
```

### 26.3 Minimal manifest.webmanifest

```json
{
  "name": "FULL_CLIENT_NAME",
  "short_name": "SHORT_LABEL",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#BRAND_HEX",
  "background_color": "#FFFFFF",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### 26.4 nginx MIME Type Snippet

```nginx
location ~ \.webmanifest$ {
    add_header Content-Type "application/manifest+json";
}
```

End every nginx change with:

```bash
nginx -t && systemctl reload nginx
```

---

## 27. REFERENCES

* Apple Configuring Web Applications: https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariWebContent/ConfiguringWebApplications/ConfiguringWebApplications.html
* W3C Web App Manifest specification: https://www.w3.org/TR/appmanifest/
* MDN Web App Manifest reference: https://developer.mozilla.org/en-US/docs/Web/Manifest
* MDN apple-mobile-web-app-capable: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name
* Microsoft browser configuration schema reference (legacy): https://learn.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/ie11-deprecated-features/dn455106(v=vs.85)
* PWA on iOS compatibility reference: https://firt.dev/notes/pwa-ios/

**Companion frameworks in the HTML signal track:**

* framework-html-meta-robots.md
* framework-html-meta-charset.md
* framework-html-meta-viewport.md
* framework-html-meta-description.md
* framework-html-meta-keywords.md
* framework-html-meta-author.md
* framework-html-meta-generator.md
* framework-html-meta-copyright.md
* framework-html-meta-theme-color.md (direct companion)
* framework-html-meta-color-scheme.md
* framework-html-meta-referrer.md
* framework-html-content-language.md
* framework-html-meta-refresh.md
* framework-html-open-graph.md
* framework-html-twitter-cards.md
* framework-html-search-engine-verification.md

**Forward references:**

* framework-html-favicon-icons.md (the full icon family, including manifest icons).
* framework-pwa-service-worker.md (service worker implementation for full PWA).
* framework-pwa-manifest.md (deeper manifest field reference).

**Parent reference:** UNIVERSAL-RANKING-FRAMEWORK.md (the 27 section master document).

---

*End of framework-html-app-pwa-meta.md.*
