# framework-html-meta-copyright.md

Comprehensive reference for HTML `<meta name="copyright">`, the page copyright declaration tag. Covers the legal reality (copyright is automatic upon creation; the tag merely declares what already exists), the current operational status (Google ignores for ranking; visible footer notice is the primary legal signal), the modern alternatives (Schema.org `copyrightHolder` / `copyrightYear` / `copyrightNotice` / `license` properties; `<link rel="license">` for Creative Commons style explicit licensing; visible footer text for legal notice), the year update problem (hard coded years that become stale; JavaScript auto year fallback handling), the client vs developer copyright attribution distinction (the client owns content copyright; Joseph's "Crafted by ThatDeveloperGuy.com" footer is developer attribution, not copyright claim), the Creative Commons license pattern for shareable content, and the Bubbles decision framework (omit the meta tag; use Schema.org plus visible footer plus rel="license" where applicable). Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the eighth framework in the HTML signal track**, following meta robots, charset, viewport, description, keywords, author, and generator. Companion to the 12 wire layer frameworks.

Audience: humans configuring legally accurate copyright notices on client sites, AI assistants generating HTML head sections with appropriate (or absent) copyright metadata, developers working with editorial content where authorship and copyright differ, federal subcontractors balancing client copyright with government public domain considerations, anyone troubleshooting "should we include meta copyright on every page", "our footer still says 2019 in 2026", "do we need to register copyright before declaring it", or "how do we license content for sharing".

---

## TABLE OF CONTENTS

1. Definition
2. Why It (Mostly Doesn't) Matter
3. What This Covers
4. The Copyright Mental Model (read this first)
5. The Legal Reality: Copyright Is Automatic
6. Meta Copyright vs Schema.org vs Visible Footer vs rel="license"
7. The Schema.org Copyright Properties
8. The Visible Footer Convention (Bubbles standard)
9. The rel="license" Link Approach
10. The Creative Commons Pattern
11. The Year Update Problem
12. The Client vs Developer Attribution Distinction
13. The Federal Public Domain Consideration
14. Asset Class And Use Case Recipes
15. Bubbles Standard Pattern (paste ready)
16. Audit Checklist
17. Common Pitfalls
18. Diagnostic Commands
19. Cross-References

---

## 1. DEFINITION

`<meta name="copyright">` is the HTML directive that declares page copyright information. It is informational metadata rooted in Dublin Core conventions from the 1990s. The tag is not formally part of the HTML5 spec but is widely recognized and parsed.

```html
<head>
    <meta name="copyright" content="Copyright 2026 ThatDeveloperGuy.com. All rights reserved.">
</head>
```

Three structural facts shape how the directive works:

* **It is informational, not legal.** Copyright exists from the moment a work is created. The tag merely declares the copyright holder; it does not establish copyright (creation does that automatically under Berne Convention).
* **The visible footer is the primary signal.** Courts, users, and crawlers all weight the visible page footer copyright notice more heavily than the meta tag. The meta tag is supplementary.
* **Modern Schema.org alternatives are richer.** `Article.copyrightHolder`, `Article.copyrightYear`, `Article.copyrightNotice`, and `Article.license` provide structured copyright information that search engines and indexing systems can use.

For Bubbles client sites in 2026, the convention is to omit the meta copyright tag and rely on visible footer notices plus Schema.org structured data. The meta tag adds no operational value over these alternatives.

---

## 2. WHY IT (MOSTLY DOESN'T) MATTER

Seven independent considerations clarify the decision to omit meta copyright in favor of alternatives.

**Copyright is automatic; the tag does not establish it.** Per the Berne Convention (ratified by 181 countries including the United States) and US Copyright Act, copyright exists from the moment of creation. No notice is required. The visible "© 2026 Author Name" or meta tag are courtesy declarations; legal copyright exists regardless.

**Google ignores meta copyright for ranking.** The tag does not influence search rankings. Like meta keywords, it provides no SEO benefit. Removing it has no negative impact.

**The visible footer is what matters legally.** When copyright disputes arise, courts and users look at the visible page footer for copyright notice. The meta tag rarely surfaces in legal analysis. For DMCA notices and similar, the visible notice is the standard reference.

**Schema.org provides richer structured data.** Schema.org Article and CreativeWork types support `copyrightHolder` (Person or Organization), `copyrightYear` (date), `copyrightNotice` (text), and `license` (URL). This structured data is consumed by indexers, AI assistants, and other parsers that ignore the simpler meta tag.

**Stale year problem is common.** Sites with hard coded "Copyright 2019" in the meta tag (and footer) signal abandonment or low maintenance to users. Visitors who see outdated copyright assume the site isn't maintained. Automated year handling (JavaScript or template variable) is the fix; many sites don't implement it.

**Open license patterns require `rel="license"`.** Creative Commons, Apache, MIT, and other explicit licensing schemes use `<link rel="license" href="...">` to point to the license URL. The meta copyright tag does not convey license information.

**Federal subcontracting has public domain considerations.** Work created for the federal government may be in public domain (US works of the federal government are not copyrightable per 17 U.S.C. § 105). Client sites for federal subcontractors need careful copyright handling that the meta tag cannot adequately express.

**Cost of getting it wrong.** Misconfigured copyright metadata produces measurable signals. Real examples:

* Bubbles client site had hard coded "© 2018 Client Business" in both meta tag and footer. Site visited in 2026; visitors noticed the 8 year old date and assumed the site was abandoned. Bounce rate increased. Fix: dynamic year via template variable; remove meta tag.
* Federal subcontractor site had `<meta name="copyright" content="Copyright 2024 Government Agency. All rights reserved.">` on pages containing federal works. Federal works of the US government are not copyrightable under 17 U.S.C. § 105. The notice was technically incorrect. Fix: remove copyright claim from federal work pages; add public domain notice where appropriate.
* Editorial site used Creative Commons content but never declared the license. Users couldn't determine reuse permissions. Search engines couldn't classify the content. Adding `<link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">` plus Schema.org `license` property resolved the ambiguity.
* Joseph's editorial blog originally had meta copyright crediting "ThatDeveloperGuy" but the content was attributed to Joseph individually. Mismatch between copyright holder and visible author. Aligned to Joseph Anady as copyright holder for editorial content; ThatDeveloperGuy as copyright holder for marketing pages.
* Client migration from WordPress to Bubbles preserved a 2017 copyright meta tag. New Bubbles site appeared to be 9 years old based on copyright year. Removed during cleanup.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The meta copyright tag plus its full operational context:

1. **The legal reality**: copyright is automatic; declarations are courtesy.
2. **The four signal channels**: meta tag, Schema.org, visible footer, rel="license".
3. **The year update problem**: keeping dates current.
4. **The client vs developer distinction**: who owns what.
5. **The federal public domain consideration**: SDVOSB context.
6. **The Bubbles decision**: omit meta tag; use alternatives.

Section 8 covers the visible footer pattern in detail (the Bubbles standard). Section 12 addresses the client vs developer attribution.

---

## 4. THE COPYRIGHT MENTAL MODEL (READ THIS FIRST)

Copyright is a legal status that exists by default. Declarations are signals to users about who owns what.

```
A work is created (article, photo, code, design).
   |
   v
Copyright exists automatically (Berne Convention, US Copyright Act).
   |
   v
The creator does NOT need to:
   - Register copyright (though registration provides legal advantages).
   - Display a copyright notice.
   - Use the © symbol.
   - Specify a year.
   - File any paperwork.
   |
   v
The creator MAY:
   - Display copyright notice for clarity.
   - Specify what users may do (license).
   - Provide contact for permissions.
   - Declare copyright holder identity (for editorial content where author != site owner).

==================== THE FOUR SIGNAL CHANNELS ====================

   |
   |---> Visible footer text (PRIMARY)
   |     "© 2026 ThatDeveloperGuy.com. All rights reserved."
   |     What users see; what courts reference; what crawlers parse from body.
   |
   |---> Schema.org structured data
   |     copyrightHolder, copyrightYear, copyrightNotice, license
   |     Rich, structured, consumed by indexers and AI.
   |
   |---> rel="license" link
   |     <link rel="license" href="https://...">
   |     Explicit license URL for Creative Commons etc.
   |
   |---> Meta copyright tag (LEGACY)
   |     <meta name="copyright" content="...">
   |     Informational; ignored by Google; superseded by alternatives.

==================== THE BUBBLES DECISION ====================

   |
   v
   Meta copyright tag: OMIT.
   |
   Visible footer: INCLUDE on every page.
   |
   Schema.org copyright properties: INCLUDE for editorial content.
   |
   rel="license": INCLUDE if explicit license (CC, etc.) applies.
```

Six rules govern the system:

1. **Copyright is automatic.** Notices are courtesy and clarity, not legal requirement.
2. **The visible footer is the primary signal.** Prioritize getting this right.
3. **Use Schema.org for structured copyright data.** Richer than meta tag.
4. **Use rel="license" for explicit licensing.** Creative Commons, MIT, Apache, etc.
5. **Update the year dynamically.** Don't hard code dates that become stale.
6. **Distinguish copyright holder from developer attribution.** Client owns content; Joseph is credited as builder.

A correctly configured site has accurate visible footer copyright, Schema.org structured copyright data, optional rel="license" for licensed content, and NO meta copyright tag.

---

## 5. THE LEGAL REALITY: COPYRIGHT IS AUTOMATIC

Understanding copyright law clarifies why the meta tag matters less than alternatives.

### 5.1 The Berne Convention

The Berne Convention for the Protection of Literary and Artistic Works (1886, revised multiple times) established the principle that copyright is automatic upon creation. The US joined in 1989. 181 countries are parties.

Key principles:

* Copyright exists from the moment of creation.
* No notice is required for protection.
* No registration is required for protection.
* The © symbol is not required.

Registration with the US Copyright Office (or equivalent) provides advantages:

* Public record of ownership.
* Required to file infringement lawsuits in US federal court.
* Statutory damages available for registered works.

But the absence of registration does not invalidate copyright; it just limits remedies.

### 5.2 What The Meta Tag Does (And Doesn't)

The meta tag:

* Declares the copyright holder.
* Provides a date reference (usually).
* Provides notice to readers.

The meta tag does NOT:

* Establish copyright (creation does).
* Provide legal protection.
* Substitute for registration.
* Convey license terms (rel="license" or Schema.org does).

### 5.3 The Notice Pattern

A standard copyright notice contains:

* The © symbol (or the word "Copyright").
* The year of first publication.
* The name of the copyright holder.

```
© 2026 ThatDeveloperGuy.com
```

The "All rights reserved" phrase is a holdover from the Buenos Aires Convention (1910), historically required in some Latin American countries. Modern Berne Convention treatment makes it unnecessary, but it remains common.

### 5.4 The Update Rule

Copyright year refers to the year of first publication, not the current year. Strict accuracy:

```
© 2024 ThatDeveloperGuy.com  (if first published in 2024)
```

Common practice (slightly looser):

```
© 2024-2026 ThatDeveloperGuy.com  (covering first publication through current year)
```

The range form acknowledges ongoing modification while preserving the original publication date.

### 5.5 The Bubbles Implication

For Bubbles client sites:

* Add visible footer copyright for clarity.
* Use accurate year (or year range).
* Use dynamic year for the upper bound.
* Skip the meta tag (Schema.org and footer are sufficient).

---

## 6. META COPYRIGHT VS SCHEMA.ORG VS VISIBLE FOOTER VS REL="LICENSE"

Four ways to convey copyright information. Each plays a different role.

### 6.1 The Visible Footer (Primary)

```html
<footer>
    <p>© 2026 ThatDeveloperGuy.com. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

What users see. What courts examine. What crawlers parse from body content. The strongest signal.

### 6.2 Schema.org (Structured)

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "Article",
    "copyrightHolder": {
        "@type": "Person",
        "name": "Joseph Anady",
        "url": "https://thatdeveloperguy.com/about/"
    },
    "copyrightYear": "2026",
    "copyrightNotice": "© 2026 Joseph Anady. All rights reserved.",
    "license": "https://creativecommons.org/licenses/by-sa/4.0/"
}
</script>
```

Structured, machine readable, supports rich author/publisher information.

### 6.3 rel="license" Link (For Explicit Licensing)

```html
<link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">
```

For pages with explicit license (Creative Commons, MIT, Apache, etc.). Points to canonical license URL.

### 6.4 Meta Copyright Tag (Legacy)

```html
<meta name="copyright" content="© 2026 ThatDeveloperGuy.com. All rights reserved.">
```

Informational. Not used by Google for ranking. Superseded by alternatives.

### 6.5 The Coverage Pattern

For typical Bubbles client content:

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Article Title | Client Business</title>
    <meta name="description" content="...">

    <!-- NO meta copyright (superseded by below) -->

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "Article",
        "copyrightHolder": {"@type": "Organization", "name": "Client Business"},
        "copyrightYear": "2026"
    }
    </script>
</head>
<body>
    <!-- ... content ... -->

    <footer>
        <p>© 2026 Client Business. Crafted by ThatDeveloperGuy.com.</p>
    </footer>
</body>
```

Three layers (Schema, visible footer, no meta tag). Comprehensive without redundancy.

### 6.6 The Decision Matrix

| Use case | Visible footer | Schema.org | rel="license" | meta copyright |
|---|---|---|---|---|
| Standard marketing site | YES | OPTIONAL | NO | NO |
| Editorial blog | YES | YES | OPTIONAL | NO |
| Creative Commons content | YES | YES | YES | NO |
| Federal public domain | NO/CONDITIONAL | YES (with note) | NO | NO |
| Open source documentation | YES | YES | YES (license) | NO |

Notice: meta copyright is "NO" in all cases.

---

## 7. THE SCHEMA.ORG COPYRIGHT PROPERTIES

Schema.org provides structured copyright information that the meta tag cannot.

### 7.1 The Properties

* **`copyrightHolder`**: Person or Organization that owns the copyright. Can include `url`, `sameAs`, and other identity properties.
* **`copyrightYear`**: integer year of first publication.
* **`copyrightNotice`**: text of the copyright notice (matches what's in the visible footer).
* **`license`**: URL of the applicable license (for explicit licensing).

### 7.2 The Article Example

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": "Article Title",
    "author": {
        "@type": "Person",
        "name": "Joseph Anady"
    },
    "copyrightHolder": {
        "@type": "Person",
        "name": "Joseph Anady",
        "url": "https://thatdeveloperguy.com/about/",
        "sameAs": ["https://www.wikidata.org/wiki/Q138610626"]
    },
    "copyrightYear": "2026",
    "copyrightNotice": "© 2026 Joseph Anady. All rights reserved.",
    "publisher": {
        "@type": "Organization",
        "name": "ThatDeveloperGuy.com"
    }
}
</script>
```

The `copyrightHolder` may be the same as `author` (common for editorial content), or different (e.g., a freelance writer's work where the magazine is the copyright holder).

### 7.3 The Organization Example

For corporate sites where the organization owns all content:

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "WebPage",
    "copyrightHolder": {
        "@type": "Organization",
        "name": "Client Business",
        "url": "https://example.com"
    },
    "copyrightYear": "2026"
}
</script>
```

### 7.4 The LocalBusiness Example

For Bubbles local clients (NWA/SWMO businesses):

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "LocalBusiness",
    "name": "Heritage Hardwood Floors NWA",
    "copyrightHolder": {
        "@type": "Organization",
        "name": "Heritage Hardwood Floors NWA"
    },
    "copyrightYear": "2026"
}
</script>
```

### 7.5 The Multi Year Pattern

For ongoing content updates:

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "Article",
    "copyrightYear": "2024",
    "dateModified": "2026-05-15T08:00:00-05:00",
    "copyrightNotice": "© 2024-2026 Joseph Anady. Content updated periodically."
}
</script>
```

The `copyrightYear` is first publication; `dateModified` tracks ongoing changes.

---

## 8. THE VISIBLE FOOTER CONVENTION (BUBBLES STANDARD)

The visible footer copyright notice is the most important signal. Get it right.

### 8.1 The Bubbles Footer Standard

Per Joseph's standing rule: every client build's footer includes "Crafted by ThatDeveloperGuy.com".

Combined with client copyright:

```html
<footer>
    <p>© 2026 Client Business Name. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

This combines:

* Client copyright notice (legal).
* Developer attribution (marketing).

Both serve their respective purposes without conflict.

### 8.2 The Year Update Pattern

**Pattern A: Server side rendering (preferred for Bubbles).**

```python
# /opt/bubbles/services/example.com/main.py
from datetime import datetime

@app.get("/")
async def homepage(request: Request):
    return templates.TemplateResponse("homepage.html", {
        "request": request,
        "current_year": datetime.now().year,
        "site_start_year": 2020,
    })
```

```html
<!-- template -->
<footer>
    <p>© {{ site_start_year }}-{{ current_year }} Client Business. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

**Pattern B: JavaScript fallback (defensive).**

```html
<footer>
    <p>© <span id="year-range">2024</span> Client Business. Crafted by ThatDeveloperGuy.com.</p>
</footer>

<script>
    document.getElementById('year-range').textContent = '2024-' + new Date().getFullYear();
</script>
```

JavaScript ensures the year updates even on cached pages. Combined with server side rendering for non JS users.

**Pattern C: Static (acceptable; needs annual review).**

```html
<footer>
    <p>© 2026 Client Business. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

Update annually as part of January maintenance. Simple but requires discipline.

### 8.3 The Footer Variants

**Single year (first year of operation):**

```html
<p>© 2026 Client Business. Crafted by ThatDeveloperGuy.com.</p>
```

**Year range (multi year operation):**

```html
<p>© 2020-2026 Client Business. Crafted by ThatDeveloperGuy.com.</p>
```

**With "All rights reserved" (international clarity):**

```html
<p>© 2026 Client Business. All rights reserved. Crafted by ThatDeveloperGuy.com.</p>
```

**With license link (for Creative Commons):**

```html
<p>© 2026 Joseph Anady. Licensed under <a href="https://creativecommons.org/licenses/by-sa/4.0/">CC BY-SA 4.0</a>. Crafted by ThatDeveloperGuy.com.</p>
```

### 8.4 The Client Education

When clients ask about copyright notices:

> "Your copyright exists automatically when you create the content. The visible footer notice is what users and courts reference. We use a dynamic year so it stays current. Schema.org structured data in the page head provides the technical signal to search engines. We don't use the older meta copyright tag because it's been superseded."

This framing helps clients understand the layered approach.

### 8.5 The Federal Subcontract Footer

For federal subcontractor sites where work may be partially or wholly public domain:

```html
<footer>
    <p>This site contains content created for [Federal Agency], which may be public domain pursuant to 17 U.S.C. § 105. Site framework © 2026 ThatDeveloperGuy.com. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

The notice distinguishes content (potentially public domain) from site infrastructure (copyrighted).

---

## 9. THE rel="license" LINK APPROACH

For pages with explicit licensing (Creative Commons, MIT, Apache, etc.), use `rel="license"`.

### 9.1 The Pattern

```html
<head>
    <link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">
</head>
```

The link points to the canonical license URL. Crawlers, license aggregators, and reuse tools follow this link to determine the terms.

### 9.2 The Common Licenses

**Creative Commons family:**

```html
<!-- CC BY 4.0 (attribution required, derivatives allowed) -->
<link rel="license" href="https://creativecommons.org/licenses/by/4.0/">

<!-- CC BY-SA 4.0 (attribution required, share alike derivatives) -->
<link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">

<!-- CC BY-NC 4.0 (attribution required, no commercial use) -->
<link rel="license" href="https://creativecommons.org/licenses/by-nc/4.0/">

<!-- CC BY-NC-SA 4.0 (attribution, no commercial, share alike) -->
<link rel="license" href="https://creativecommons.org/licenses/by-nc-sa/4.0/">

<!-- CC0 (public domain dedication) -->
<link rel="license" href="https://creativecommons.org/publicdomain/zero/1.0/">
```

**Software licenses:**

```html
<!-- MIT License -->
<link rel="license" href="https://opensource.org/licenses/MIT">

<!-- Apache 2.0 -->
<link rel="license" href="https://www.apache.org/licenses/LICENSE-2.0">

<!-- GPL 3.0 -->
<link rel="license" href="https://www.gnu.org/licenses/gpl-3.0.html">
```

### 9.3 The Pairing With Schema.org

```html
<head>
    <link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "Article",
        "copyrightHolder": {"@type": "Person", "name": "Joseph Anady"},
        "copyrightYear": "2026",
        "license": "https://creativecommons.org/licenses/by-sa/4.0/"
    }
    </script>
</head>
```

Both layers point to the same license. Consistent signal.

### 9.4 When To Use Explicit Licensing

For Bubbles client sites:

* **Marketing pages**: standard copyright; no license link needed.
* **Editorial blog with reuse permission**: license link applicable.
* **Documentation intended for community use**: license link essential.
* **Code samples or design assets**: license link for clarity.

For most Bubbles client work: no license link needed. The default is "all rights reserved" (implicit without notice).

---

## 10. THE CREATIVE COMMONS PATTERN

When content is intentionally shared under Creative Commons, the proper attribution pattern.

### 10.1 The Full Pattern

```html
<head>
    <link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "Article",
        "headline": "Article Title",
        "author": {
            "@type": "Person",
            "name": "Joseph Anady"
        },
        "copyrightHolder": {
            "@type": "Person",
            "name": "Joseph Anady"
        },
        "copyrightYear": "2026",
        "license": "https://creativecommons.org/licenses/by-sa/4.0/"
    }
    </script>
</head>
<body>
    <article>
        <h1>Article Title</h1>
        <p class="byline">By Joseph Anady</p>
        <!-- ... content ... -->

        <footer class="article-footer">
            <p>This article is licensed under
                <a href="https://creativecommons.org/licenses/by-sa/4.0/" rel="license">
                    Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)
                </a>.
            </p>
            <p>You are free to share and adapt this work, with attribution to Joseph Anady at ThatDeveloperGuy.com.</p>
        </footer>
    </article>

    <footer>
        <p>© 2026 Joseph Anady. Content licensed CC BY-SA 4.0. Crafted by ThatDeveloperGuy.com.</p>
    </footer>
</body>
```

Three signals reinforce the licensing:

1. `<link rel="license">` in head.
2. Schema.org `license` property.
3. Visible article footer and site footer notices.

### 10.2 The Attribution Requirement

Creative Commons licenses require attribution when others share the work. The attribution should include:

* Author name (Joseph Anady).
* Source URL.
* License name and link.

This is the responsibility of the REUSER, not the original publisher. But making it easy by clearly stating these in the article footer helps reusers comply.

### 10.3 The Bubbles Application

For Joseph's own editorial blog content where Creative Commons sharing is desired:

* CC BY-SA 4.0 is the convention (allows sharing with attribution and share alike).
* Visible footer notice plus rel="license" plus Schema.org license property.

For client sites: typically not CC licensed (client content is proprietary). Skip the license declarations.

---

## 11. THE YEAR UPDATE PROBLEM

Hard coded years become stale. Common pitfall across the web.

### 11.1 The Stale Year Symptom

Site visited in 2026 shows:

```html
<footer>
    <p>© 2019 Client Business. All rights reserved.</p>
</footer>
```

Visitors notice. Search engines parse the publication date. Both signals indicate site abandonment.

### 11.2 The Detection

```bash
# Find sites with old hard coded copyright years
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    YEARS=$(grep -roE '©|copyright)[[:space:]]+([12][0-9]{3})' "$site_dir/index.html" 2>/dev/null | \
            grep -oE '[12][0-9]{3}' | sort -u)

    if [ -n "$YEARS" ]; then
        # Get current year
        CURRENT_YEAR=$(date +%Y)
        for y in $YEARS; do
            # Flag if year is more than 1 year old
            if [ "$y" -lt "$((CURRENT_YEAR - 1))" ]; then
                echo "STALE: $SITE shows $y (current: $CURRENT_YEAR)"
            fi
        done
    fi
done
```

### 11.3 The Server Side Fix (Preferred)

```python
# FastAPI / Jinja2 template variable
from datetime import datetime

@app.get("/")
async def homepage(request: Request):
    return templates.TemplateResponse("homepage.html", {
        "request": request,
        "current_year": datetime.now().year,
    })
```

```html
<footer>
    <p>© {{ current_year }} Client Business. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

The year is always current; no annual updates needed.

### 11.4 The JavaScript Fix (Defensive)

```html
<footer>
    <p>© <span id="copyright-year">2024</span> Client Business. Crafted by ThatDeveloperGuy.com.</p>
</footer>

<script>
    document.getElementById('copyright-year').textContent = new Date().getFullYear();
</script>
```

The HTML has a fallback year; JavaScript updates it.

### 11.5 The Year Range Pattern

For sites that want to acknowledge multi year history:

```python
# Server side
{% set start_year = 2020 %}
{% set current_year = current_year %}
{% if start_year == current_year %}
    © {{ current_year }} Business
{% else %}
    © {{ start_year }}-{{ current_year }} Business
{% endif %}
```

Output:

```
© 2020-2026 Business
```

### 11.6 The Static Site Maintenance

For purely static Bubbles sites (no FastAPI):

* Annual maintenance: update year in template.
* Add to January maintenance checklist.
* Or use a build process that injects current year.

```bash
# Bulk update across static sites
CURRENT_YEAR=$(date +%Y)
for site_dir in /var/www/sites/*/; do
    find "$site_dir" -name "*.html" -type f -exec \
        sed -i "s/© 2019/© ${CURRENT_YEAR}/g; s/© 2020/© ${CURRENT_YEAR}/g; s/© 2021/© ${CURRENT_YEAR}/g; s/© 2022/© ${CURRENT_YEAR}/g; s/© 2023/© ${CURRENT_YEAR}/g; s/© 2024/© ${CURRENT_YEAR}/g; s/© 2025/© ${CURRENT_YEAR}/g" {} \;
done
```

(Note: be careful with this; it overwrites legitimate "2024" mentions in body content. Better to use template variables.)

---

## 12. THE CLIENT VS DEVELOPER ATTRIBUTION DISTINCTION

For Bubbles, the footer contains two distinct attributions:

1. **Copyright notice**: who owns the content (CLIENT).
2. **Developer attribution**: who built the site (Joseph / ThatDeveloperGuy.com).

These serve different purposes and should not conflict.

### 12.1 The Distinction

```html
<footer>
    <p>© 2026 Client Business. Crafted by ThatDeveloperGuy.com.</p>
    [^^^^^^^^^^^^^^^^^^^]   [^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^]
    Copyright (client)        Developer attribution (Joseph)
</footer>
```

The client owns the content copyright. Joseph is credited as the developer who built the site.

### 12.2 The Common Confusion

Some sites attribute the developer as the copyright holder:

```html
<!-- WRONG: developer claiming copyright on client content -->
<footer>
    <p>© 2026 ThatDeveloperGuy.com. All rights reserved.</p>
</footer>
```

This is incorrect. The CLIENT owns the copyright to their content (assuming work for hire or standard client agreement). The developer's role is documented separately.

### 12.3 The Bubbles Convention

For every client build:

```html
<footer>
    <p>© 2026 [Client Business Name]. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

Where:

* `[Client Business Name]` is the client's legal entity name (e.g., "Heritage Hardwood Floors NWA", "Arkansas Counseling and Wellness Services").
* `ThatDeveloperGuy.com` is the developer attribution per Joseph's standing footer rule.

### 12.4 The Schema.org Application

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "Article",
    "copyrightHolder": {
        "@type": "Organization",
        "name": "Client Business"
    },
    "copyrightYear": "2026",
    "publisher": {
        "@type": "Organization",
        "name": "Client Business"
    },
    "creator": {
        "@type": "Person",
        "name": "Joseph Anady",
        "url": "https://thatdeveloperguy.com"
    }
}
</script>
```

The `copyrightHolder` and `publisher` are the client. The `creator` is Joseph (the developer/builder).

### 12.5 The Joseph Own Sites

For Joseph's own sites (thatdeveloperguy.com, thataiguy.org):

* Joseph IS both the copyright holder and developer.
* Single attribution is appropriate.

```html
<footer>
    <p>© 2026 ThatDeveloperGuy.com. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

(Or simpler: just the copyright notice without the redundant "Crafted by" since it's the same entity.)

### 12.6 The Work For Hire Note

In most client engagements, the work is "work for hire" or otherwise transfers copyright to the client upon payment. The Bubbles client agreement typically specifies this.

For projects where Joseph retains some rights (e.g., he wants to feature the work in his portfolio):

```html
<footer>
    <p>© 2026 Client Business. Site portfolio licensed to ThatDeveloperGuy.com. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

This requires explicit client agreement.

---

## 13. THE FEDERAL PUBLIC DOMAIN CONSIDERATION

For SDVOSB federal subcontracting work, copyright handling differs.

### 13.1 The 17 U.S.C. § 105 Rule

Per 17 U.S.C. § 105: works of the United States Government are not subject to copyright protection.

This means:

* Content created BY federal employees for the government: not copyrightable.
* Content created BY contractors for the government: complicated; depends on contract terms.
* Content created BY the contractor for their own use: copyrightable (normal).

For Joseph's SDVOSB federal subcontracting:

* If the work product is delivered to and owned by a federal agency: may be subject to federal contract terms about copyright.
* If the work product is Joseph's tooling, framework, or other reusable assets: Joseph retains copyright.
* If the work product creates derivative federal documents: federal portions may be public domain.

### 13.2 The Footer Pattern For Federal Work

```html
<footer>
    <p>Content provided to [Federal Agency] subject to federal contract terms. Site framework © 2026 ThatDeveloperGuy.com. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

This distinguishes:

* Content (subject to federal contract terms; may be public domain or restricted).
* Site framework (Joseph's IP, copyrighted).

### 13.3 The Schema.org For Federal Work

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "WebPage",
    "copyrightHolder": {
        "@type": "Organization",
        "name": "ThatDeveloperGuy.com (site framework)",
        "url": "https://thatdeveloperguy.com"
    },
    "copyrightYear": "2026",
    "copyrightNotice": "Site framework © 2026 ThatDeveloperGuy.com. Content subject to federal contract terms."
}
</script>
```

The structured data clarifies the divided ownership.

### 13.4 The Documentation Importance

For every federal subcontract:

* Document copyright terms in the project file.
* Reference the specific contract clauses.
* Ensure visible footer accurately reflects the terms.
* Quarterly review during contract renewals.

---

## 14. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 14.1 Canonical Bubbles head (no meta copyright)

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Page Title | Client Business</title>
    <meta name="description" content="...">

    <!-- NO meta copyright (superseded by visible footer and Schema.org) -->

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "WebPage",
        "copyrightHolder": {
            "@type": "Organization",
            "name": "Client Business"
        },
        "copyrightYear": "2026"
    }
    </script>
</head>
<body>
    <!-- ... content ... -->

    <footer>
        <p>© 2026 Client Business. Crafted by ThatDeveloperGuy.com.</p>
    </footer>
</body>
```

### 14.2 Editorial blog post with author copyright

```html
<head>
    <meta charset="utf-8">

    <title>Article Title | ThatDeveloperGuy</title>
    <meta name="description" content="...">
    <meta name="author" content="Joseph Anady">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "headline": "Article Title",
        "author": {
            "@type": "Person",
            "name": "Joseph Anady",
            "sameAs": ["https://www.wikidata.org/wiki/Q138610626"]
        },
        "copyrightHolder": {
            "@type": "Person",
            "name": "Joseph Anady"
        },
        "copyrightYear": "2026",
        "copyrightNotice": "© 2026 Joseph Anady. All rights reserved.",
        "datePublished": "2026-05-15T08:00:00-05:00"
    }
    </script>
</head>
<body>
    <article>
        <h1>Article Title</h1>
        <p class="byline">By Joseph Anady</p>
        <!-- ... content ... -->
    </article>

    <footer>
        <p>© 2026 Joseph Anady. Crafted by ThatDeveloperGuy.com.</p>
    </footer>
</body>
```

### 14.3 Creative Commons licensed content

```html
<head>
    <link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "Article",
        "headline": "Article Title",
        "author": {"@type": "Person", "name": "Joseph Anady"},
        "copyrightHolder": {"@type": "Person", "name": "Joseph Anady"},
        "copyrightYear": "2026",
        "license": "https://creativecommons.org/licenses/by-sa/4.0/"
    }
    </script>
</head>
<body>
    <article>
        <h1>Article Title</h1>
        <!-- ... content ... -->

        <p>This article is licensed
            <a href="https://creativecommons.org/licenses/by-sa/4.0/" rel="license">CC BY-SA 4.0</a>.
            Share with attribution to Joseph Anady.
        </p>
    </article>

    <footer>
        <p>© 2026 Joseph Anady. Content licensed CC BY-SA 4.0. Crafted by ThatDeveloperGuy.com.</p>
    </footer>
</body>
```

### 14.4 Federal subcontractor site footer

```html
<footer>
    <p>Content provided to [Federal Agency] subject to federal contract terms. Site framework © 2026 ThatDeveloperGuy.com. Crafted by ThatDeveloperGuy.com.</p>
</footer>
```

### 14.5 Server side year (FastAPI Jinja2)

```python
from datetime import datetime
from fastapi import FastAPI
from fastapi.templating import Jinja2Templates

app = FastAPI()
templates = Jinja2Templates(directory="templates")

@app.middleware("http")
async def add_template_globals(request, call_next):
    request.state.current_year = datetime.now().year
    return await call_next(request)

@app.get("/")
async def homepage(request):
    return templates.TemplateResponse("homepage.html", {
        "request": request,
        "current_year": request.state.current_year,
        "site_start_year": 2020,
    })
```

```html
{# /opt/bubbles/services/example.com/templates/footer.html #}
<footer>
    {% if site_start_year == current_year %}
        <p>© {{ current_year }} {{ business_name }}. Crafted by ThatDeveloperGuy.com.</p>
    {% else %}
        <p>© {{ site_start_year }}-{{ current_year }} {{ business_name }}. Crafted by ThatDeveloperGuy.com.</p>
    {% endif %}
</footer>
```

### 14.6 JavaScript year fallback

```html
<footer>
    <p>© <span id="copyright-year">2024</span>-<span id="current-year">2024</span> Client Business. Crafted by ThatDeveloperGuy.com.</p>
</footer>

<script>
    const currentYear = new Date().getFullYear();
    document.getElementById('current-year').textContent = currentYear;
</script>
```

### 14.7 Removing meta copyright from legacy templates

```bash
# Find HTML files with meta copyright
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    grep -l 'meta name="copyright"' {} \;

# Remove the tags
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    sed -i '/meta name="copyright"/d' {} \;

# Verify
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    grep -l 'meta name="copyright"' {} \;
# Should produce no output
```

### 14.8 Bulk audit for stale years

```bash
#!/bin/bash
# /usr/local/bin/year-audit.sh

CURRENT_YEAR=$(date +%Y)

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"

    if [ -f "$INDEX" ]; then
        # Find copyright years in HTML
        YEARS=$(grep -oE '(©|copyright|Copyright)[[:space:]]+(20[0-9]{2})' "$INDEX" | grep -oE '20[0-9]{2}' | sort -u)

        for y in $YEARS; do
            if [ "$y" -lt "$((CURRENT_YEAR - 1))" ]; then
                echo "STALE YEAR: $SITE shows © $y (current: $CURRENT_YEAR)"
            fi
        done
    fi
done
```

---

## 15. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles configuration.

### 15.1 The Template Default

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    <!-- NO meta copyright -->

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "WebPage",
        "copyrightHolder": {
            "@type": "Organization",
            "name": "{{ business_name }}"
        },
        "copyrightYear": "{{ current_year }}"
    }
    </script>
</head>
<body>
    <!-- ... content ... -->

    {% include 'footer.html' %}
</body>
```

### 15.2 The Footer Template

```html
{# /opt/bubbles/services/example.com/templates/footer.html #}
<footer>
    <p>
        {% if site_start_year == current_year %}
            © {{ current_year }} {{ business_name }}.
        {% else %}
            © {{ site_start_year }}-{{ current_year }} {{ business_name }}.
        {% endif %}
        Crafted by ThatDeveloperGuy.com.
    </p>
</footer>
```

### 15.3 The Per Client Configuration

```python
# /opt/bubbles/services/example.com/client_config.py

CLIENT_CONFIG = {
    "business_name": "Client Business Name",  # legal entity for copyright
    "site_start_year": 2020,  # year site first launched
    # current_year is computed dynamically per request
}
```

### 15.4 The Audit Workflow

```bash
# /usr/local/bin/copyright-audit.sh

echo "=== Copyright audit ==="

# 1. Check for stale meta copyright tags (should be absent)
echo ""
echo "Sites with legacy meta copyright tags:"
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"
    if [ -f "$INDEX" ]; then
        if grep -q 'meta name="copyright"' "$INDEX" 2>/dev/null; then
            echo "  $SITE: has legacy meta copyright"
        fi
    fi
done

# 2. Check for stale years
echo ""
echo "Sites with stale copyright years:"
year-audit.sh

# 3. Verify Schema.org structured data
echo ""
echo "Sites with Schema.org copyright properties:"
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"
    if [ -f "$INDEX" ]; then
        if grep -q '"copyrightHolder"' "$INDEX" 2>/dev/null; then
            : # OK
        else
            echo "  $SITE: missing copyrightHolder in Schema.org"
        fi
    fi
done

echo ""
echo "=== End audit ==="
```

### 15.5 The Annual Maintenance

January maintenance for client sites without server side year:

```bash
#!/bin/bash
# /usr/local/bin/annual-year-update.sh

CURRENT_YEAR=$(date +%Y)
PREV_YEAR=$((CURRENT_YEAR - 1))

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")

    # Find files with previous year in footer
    if grep -rq "© $PREV_YEAR" "$site_dir" 2>/dev/null; then
        echo "Updating $SITE from $PREV_YEAR to $CURRENT_YEAR..."
        find "$site_dir" -name "*.html" -type f -exec \
            sed -i "s/© $PREV_YEAR/© $CURRENT_YEAR/g" {} \;
    fi
done

# Verify nginx still serves correctly
nginx -t && systemctl reload nginx
```

---

## 16. AUDIT CHECKLIST

Run through these 40 items.

### Core meta tag absence

1. [ ] No `<meta name="copyright">` on any indexable URL.
2. [ ] Legacy meta copyright tags removed during migrations.
3. [ ] No `<meta name="Copyright">` (case variations) either.

### Visible footer copyright

4. [ ] Every page has visible footer copyright notice.
5. [ ] Copyright notice includes © symbol or "Copyright" word.
6. [ ] Copyright notice includes current or year range.
7. [ ] Copyright holder is the CLIENT, not the developer.
8. [ ] Bubbles standard "Crafted by ThatDeveloperGuy.com" present.

### Year currency

9. [ ] Year is current (within current calendar year or one year prior).
10. [ ] Year updates automatically (server side preferred).
11. [ ] No hard coded years older than 1 year.
12. [ ] Year range format used for multi year sites.

### Schema.org structured data

13. [ ] Schema.org includes copyrightHolder property.
14. [ ] Schema.org includes copyrightYear property.
15. [ ] copyrightHolder matches visible footer.
16. [ ] copyrightYear matches visible footer.

### Editorial vs marketing differentiation

17. [ ] Editorial content (blog posts) attributes copyright to person (author).
18. [ ] Marketing pages attribute copyright to organization (business).
19. [ ] Consistent within each category across the site.

### Creative Commons (if applicable)

20. [ ] `<link rel="license" href="...">` present for CC content.
21. [ ] Schema.org `license` property points to license URL.
22. [ ] Visible article footer states license.
23. [ ] License URL is canonical (creativecommons.org).

### Federal subcontracting (if applicable)

24. [ ] Federal work footer references contract terms.
25. [ ] Distinguishes content (potentially public domain) from site framework.
26. [ ] Documentation maintained per contract.

### Cross cutting

27. [ ] No conflict between meta tags, Schema.org, and visible footer.
28. [ ] Year updates handled across all signal channels.
29. [ ] Footer "Crafted by ThatDeveloperGuy.com" consistent across all client sites.

### Joseph's own sites

30. [ ] thatdeveloperguy.com footer accurate.
31. [ ] thataiguy.org footer accurate.
32. [ ] Editorial content attributed to Joseph Anady (with Wikidata sameAs).

### CMS migration cleanup

33. [ ] Migrated sites have no stale year from old CMS.
34. [ ] WordPress copyright auto generation disabled.
35. [ ] Squarespace/Wix default copyright text removed.

### Tooling

36. [ ] `copyright-audit.sh` script in `/usr/local/bin/`.
37. [ ] `year-audit.sh` script in `/usr/local/bin/`.
38. [ ] Annual year update process documented.

### Documentation

39. [ ] Client copyright handling documented in project file.
40. [ ] Federal contract terms documented where applicable.

A site that passes all 40 has correctly handled copyright signaling across all channels with appropriate accuracy.

---

## 17. COMMON PITFALLS

Twelve patterns to recognize and avoid.

**Pitfall 1: Hard coded year that becomes stale.**
Symptom: 2026 visit shows "© 2019".
Why it breaks: hard coded year never updated.
Fix: server side template variable or JavaScript auto year.

**Pitfall 2: Developer attribution as copyright holder on client site.**
Symptom: `<footer>© 2026 ThatDeveloperGuy.com</footer>` on client business site.
Why it breaks: client owns content copyright, not developer.
Fix: `© 2026 [Client Business]. Crafted by ThatDeveloperGuy.com.`

**Pitfall 3: Meta copyright tag from legacy WordPress.**
Symptom: `<meta name="copyright" content="© WordPress.org">` on migrated site.
Why it breaks: stale signal from old CMS.
Fix: remove during migration.

**Pitfall 4: Multiple conflicting copyright statements.**
Symptom: meta tag says 2018, footer says 2026, Schema.org says 2022.
Why it breaks: signal fragmentation; user confusion.
Fix: align all signals (or omit meta tag and rely on footer + Schema).

**Pitfall 5: Wrong year range format.**
Symptom: "© 2024 to 2026" or "© 2024 - present" or "© 2024, 2026".
Why it breaks: non standard formats; user uncertainty.
Fix: standard formats: "© 2026" or "© 2024-2026".

**Pitfall 6: Creative Commons license without rel="license" link.**
Symptom: footer says "Licensed CC BY-SA" but no link to license terms.
Why it breaks: incomplete signal; reusers can't determine exact terms.
Fix: add `<link rel="license" href="https://creativecommons.org/licenses/by-sa/4.0/">`.

**Pitfall 7: Copyright on federal public domain work.**
Symptom: `© 2026 Federal Agency. All rights reserved.` on US government work.
Why it breaks: 17 U.S.C. § 105; US government works not copyrightable.
Fix: remove copyright claim or restructure to attribute site framework copyright only.

**Pitfall 8: All rights reserved on Creative Commons content.**
Symptom: site footer says "All rights reserved" but content is CC licensed.
Why it breaks: contradicts the license terms.
Fix: "Content licensed CC BY-SA 4.0. Site framework © 2026 [Owner]."

**Pitfall 9: Year auto update only on home page.**
Symptom: homepage shows current year; inner pages show stale year.
Why it breaks: template fragmentation.
Fix: use shared footer template or partial that's included everywhere.

**Pitfall 10: Schema.org copyrightYear as integer with quotes around it.**
Symptom: `"copyrightYear": "2026"` instead of `"copyrightYear": 2026`.
Why it breaks: technically the schema expects an integer.
Fix: use unquoted integer. (In practice, both work for major consumers.)

**Pitfall 11: Inconsistent copyright holder across pages on same site.**
Symptom: about page says "© Joseph Anady", services page says "© ThatDeveloperGuy", blog says "© Joseph W. Anady".
Why it breaks: fragmented brand identity.
Fix: standardize on canonical copyright holder name; use consistently.

**Pitfall 12: Copyright statement reveals personal information beyond what's needed.**
Symptom: "© 2026 Joseph W. Anady, 463 State Highway 76, Cassville MO 65625".
Why it breaks: unnecessary personal information disclosure.
Fix: "© 2026 Joseph Anady" or "© 2026 ThatDeveloperGuy.com" without full address.

---

## 18. DIAGNOSTIC COMMANDS

Reference of commands useful for copyright investigation.

### Inspect a single URL

```bash
# Check for meta copyright (should be absent)
curl -s https://example.com/ | grep -c 'meta name="copyright"'
# Expected: 0

# Get visible footer copyright
curl -s https://example.com/ | grep -oE '©[^<]+\|copyright[^<]+' | head -5

# Get Schema.org copyrightHolder
curl -s https://example.com/ | python3 -c "
import sys, json, re
text = sys.stdin.read()
for m in re.findall(r'<script type=\"application/ld\+json\">(.*?)</script>', text, re.DOTALL):
    try:
        data = json.loads(m)
        if 'copyrightHolder' in data:
            print(json.dumps(data['copyrightHolder'], indent=2))
        if 'copyrightYear' in data:
            print(f'copyrightYear: {data[\"copyrightYear\"]}')
    except:
        pass
"

# Get rel="license"
curl -s https://example.com/ | grep -oE 'link[^>]*rel="license"[^>]+' | head -1
```

### Find sites with stale years

```bash
# All sites with copyright years
CURRENT_YEAR=$(date +%Y)
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"
    if [ -f "$INDEX" ]; then
        YEARS=$(grep -oE '©[[:space:]]+20[0-9]{2}' "$INDEX" | grep -oE '20[0-9]{2}' | sort -u)
        for y in $YEARS; do
            if [ "$y" -lt "$((CURRENT_YEAR - 1))" ]; then
                echo "STALE: $SITE -> $y (current: $CURRENT_YEAR)"
            fi
        done
    fi
done
```

### Find sites with legacy meta copyright

```bash
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    if grep -rq 'meta name="copyright"' "$site_dir" 2>/dev/null; then
        echo "LEGACY META: $SITE"
    fi
done
```

### Verify Bubbles footer attribution

```bash
# Check that "Crafted by ThatDeveloperGuy.com" is present
for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")
    INDEX="$site_dir/index.html"
    if [ -f "$INDEX" ]; then
        HAS_FOOTER=$(grep -c "Crafted by ThatDeveloperGuy.com" "$INDEX")
        if [ "$HAS_FOOTER" = "0" ]; then
            echo "MISSING FOOTER: $SITE"
        fi
    fi
done
```

### Cross check copyright holder consistency

```bash
# Check that copyrightHolder in Schema matches visible footer
URL=$1
SCHEMA_HOLDER=$(curl -s "$URL" | python3 -c "
import sys, json, re
text = sys.stdin.read()
for m in re.findall(r'<script type=\"application/ld\+json\">(.*?)</script>', text, re.DOTALL):
    try:
        data = json.loads(m)
        h = data.get('copyrightHolder')
        if isinstance(h, dict):
            print(h.get('name', ''))
            break
        elif isinstance(h, str):
            print(h)
            break
    except:
        pass
")

FOOTER_HOLDER=$(curl -s "$URL" | grep -oE '©[[:space:]]*([0-9]{4}[-0-9]*[[:space:]]+)?[^.<]+' | head -1)

echo "Schema copyrightHolder: $SCHEMA_HOLDER"
echo "Footer text: $FOOTER_HOLDER"
```

### Apply annual year update

```bash
# Update years from previous to current
PREV_YEAR=$((`date +%Y` - 1))
CURRENT_YEAR=$(date +%Y)

for site_dir in /var/www/sites/*/; do
    find "$site_dir" -name "*.html" -type f -exec \
        sed -i "s/© $PREV_YEAR/© $CURRENT_YEAR/g" {} \;
done

nginx -t && systemctl reload nginx
```

### Audit all client sites

```bash
copyright-audit.sh
```

---

## 19. CROSS-REFERENCES

* [framework-html-meta-author.md](framework-html-meta-author.md): the copyright holder may differ from the author; both are signaled separately via Schema.org.
* [framework-html-meta-generator.md](framework-html-meta-generator.md): the Bubbles "Crafted by ThatDeveloperGuy.com" footer is developer attribution; the copyright notice is separate.
* [framework-html-meta-description.md](framework-html-meta-description.md): the description does not typically contain copyright (use schema or footer instead).
* [framework-html-meta-charset.md](framework-html-meta-charset.md): the © symbol requires UTF-8 (otherwise mojibake).
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): copyright not directly ranking but E-E-A-T uses authorship signals.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including footer convention.
* US Copyright Office: https://www.copyright.gov
* Berne Convention: https://www.wipo.int/treaties/en/ip/berne/
* 17 U.S.C. § 105 (Federal Government works): https://www.law.cornell.edu/uscode/text/17/105
* Schema.org Article (copyrightHolder, copyrightYear, license): https://schema.org/Article
* Schema.org CreativeWork: https://schema.org/CreativeWork
* Creative Commons license chooser: https://creativecommons.org/choose/
* MDN meta name: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name
* MDN link rel=license: https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/license

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Omit `<meta name="copyright">`. Use visible footer plus Schema.org instead.**

### The canonical pattern

```html
<head>
    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "WebPage",
        "copyrightHolder": {"@type": "Organization", "name": "Client Business"},
        "copyrightYear": "2026"
    }
    </script>
</head>
<body>
    <footer>
        <p>© 2026 Client Business. Crafted by ThatDeveloperGuy.com.</p>
    </footer>
</body>
```

### The forbidden patterns

```html
<!-- Stale year hard coded -->
<footer><p>© 2019 Business. All rights reserved.</p></footer>

<!-- Developer claiming copyright on client content -->
<footer><p>© 2026 ThatDeveloperGuy.com. All rights reserved.</p></footer>

<!-- Useless meta copyright on top of working footer -->
<meta name="copyright" content="© 2026 Business">
```

### Five rules to memorize

1. **Copyright is automatic. The tag declares; it doesn't establish.**
2. **Visible footer is primary. Schema.org is supplementary. Meta tag is obsolete.**
3. **Client owns copyright on client content. Joseph is credited as developer.**
4. **Update the year dynamically (server side or JavaScript fallback).**
5. **Federal public domain: distinguish content from site framework.**

### Five commands every operator should know

```bash
# 1. Check for legacy meta copyright (should be 0)
curl -s URL | grep -c 'meta name="copyright"'

# 2. Get footer copyright
curl -s URL | grep -oE '©[^<]+\|copyright[^<]+' | head -3

# 3. Get Schema.org copyrightHolder
curl -s URL | grep -oE '"copyrightHolder":[^,]+' | head -1

# 4. Find stale years across all sites
year-audit.sh

# 5. Apply annual year update
sed -i "s/© $((`date +%Y`-1))/© `date +%Y`/g" /var/www/sites/*/index.html
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. No legacy meta copyright
URL=https://example.com/
HAS_META=$(curl -s "$URL" | grep -c 'meta name="copyright"')
[ "$HAS_META" = "0" ] && echo "OK: no legacy meta" || echo "FAIL"

# 2. Visible footer present
HAS_FOOTER=$(curl -s "$URL" | grep -c "Crafted by ThatDeveloperGuy.com")
[ "$HAS_FOOTER" -ge "1" ] && echo "OK: footer present" || echo "FAIL"

# 3. Year is current
CURRENT_YEAR=$(date +%Y)
YEARS_FOUND=$(curl -s "$URL" | grep -oE '©[[:space:]]+20[0-9]{2}' | grep -oE '20[0-9]{2}' | sort -u)
HAS_CURRENT=0
for y in $YEARS_FOUND; do
    if [ "$y" = "$CURRENT_YEAR" ] || [ "$y" = "$((CURRENT_YEAR - 1))" ]; then
        HAS_CURRENT=1
    fi
done
[ "$HAS_CURRENT" = "1" ] && echo "OK: year is current" || echo "FAIL: stale year"
```

If all three pass AND Schema.org copyrightHolder matches the visible footer holder, the copyright layer is correctly wired.

---

End of framework-html-meta-copyright.md.
