# framework-html-meta-author.md

Comprehensive reference for HTML `<meta name="author">`, the page authorship attribution tag. Covers the single vs multiple author patterns, the relationship with Schema.org `Person` and `Article.author` structured data, the relationship with the visible byline element in rendered content, the relationship with Open Graph `article:author` and Twitter `twitter:creator`, the role in Google's E-E-A-T (Experience, Expertise, Authoritativeness, Trustworthiness) ranking framework for YMYL (Your Money Your Life) content, the privacy and safety considerations (publishing author identity exposes individuals to harassment), the Wikidata verified identity pattern for high authority sites (Joseph Anady is Q138610626 with SAM.gov SDVOSB registration), the corporate vs individual attribution decision, the editorial vs marketing distinction, and the Bubbles per client decision framework. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the sixth framework in the HTML signal track**, following framework-html-meta-robots.md, framework-html-meta-charset.md, framework-html-meta-viewport.md, framework-html-meta-description.md, and framework-html-meta-keywords.md. Companion to the 12 wire layer frameworks.

Audience: humans deciding whether to attribute authorship on client sites, AI assistants generating HTML head sections with appropriate authorship metadata, editorial sites establishing E-E-A-T for YMYL content (Arkansas Counseling and Wellness Services for mental health, Handled Tax and Advisory for financial topics), federal subcontractor sites attributing to credentialed individuals, journalists or bloggers establishing authorial identity, and anyone troubleshooting "Google flagged my YMYL content as low E-E-A-T", "I need to attribute this article but want to protect the author's identity", or "the byline in our content doesn't match the meta author tag".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Author Mental Model (read this first)
5. The Single vs Multiple Author Patterns
6. Meta Author vs Schema.org Person vs Visible Byline
7. The E-E-A-T Connection
8. The Privacy And Safety Consideration
9. The Wikidata Verified Identity Pattern
10. The Corporate vs Individual Attribution Decision
11. The Editorial vs Marketing Distinction
12. The Bubbles Per Client Decision Framework
13. The Relationship With Open Graph And Twitter Cards
14. Asset Class And Use Case Recipes
15. Bubbles Standard Pattern (paste ready)
16. Audit Checklist (50+ items)
17. Common Pitfalls
18. Diagnostic Commands
19. Cross-References

---

## 1. DEFINITION

`<meta name="author">` is the HTML directive that declares the author of the page. Defined informally across browser specifications and the Dublin Core Metadata Initiative (which formalized author attribution conventions decades before HTML).

```html
<head>
    <meta name="author" content="Joseph Anady">
</head>
```

Three structural facts shape how the directive works:

* **Author is one of several possible attribution signals.** The meta author tag, the visible byline in page content, Schema.org structured data, Open Graph `article:author`, and the page footer all carry authorship information. Search engines aggregate signals across them.
* **Single author is conventional; multiple authors require care.** The HTML spec allows one or many `<meta name="author">` tags on a single page, but implementations vary. Most sites use one. For multi author pages, Schema.org's structured `author` array is more reliable than multiple meta tags.
* **Privacy considerations matter.** Publishing an author's real name in page source attaches their identity to the content permanently. For YMYL content (medical, financial, legal), this is part of the credibility signal. For personal blogs or sensitive content, it can expose the author to harassment or doxxing.

For Bubbles client sites in 2026, the convention is contextual: editorial content (blog posts, articles, case studies) includes author attribution; marketing pages and service descriptions typically attribute to the organization rather than an individual; YMYL pages (especially Arkansas Counseling and Wellness, Handled Tax and Advisory) include credentialed author attribution for E-E-A-T signals.

---

## 2. WHY IT MATTERS

Seven independent considerations push correct author attribution from "afterthought metadata" to "actively managed signal" in 2025 and forward.

**Google's E-E-A-T framework values clear authorship for YMYL content.** Per Google's Search Quality Rater Guidelines, "Experience, Expertise, Authoritativeness, and Trustworthiness" signals are heavily weighted for Your Money Your Life topics (health, finance, legal, safety, civic). Pages with credentialed, verifiable authors rank better than pages with no authorship signal or pseudonymous attribution. For Arkansas Counseling and Wellness (mental health, YMYL) and Handled Tax and Advisory (tax preparation, YMYL), the author signal is part of the ranking foundation.

**The visible byline is the primary signal; meta author supports it.** Google's algorithm prioritizes the visible byline (rendered author name in the article content) over meta author. The meta author tag is a secondary support signal. Sites with strong visible bylines AND aligned meta author tags get the best of both.

**Schema.org Article.author is the structured data path.** Modern E-E-A-T implementation uses Schema.org JSON-LD with `author` as a `Person` object including `name`, `url`, `sameAs` (linking to Wikidata, LinkedIn, professional profiles), and credential properties. The meta author tag is a simpler signal that doesn't replace structured data.

**Open Graph article:author is the social media equivalent.** When articles are shared on Facebook, LinkedIn, or other Open Graph consumers, the `article:author` property determines attribution in link previews. Maintaining alignment across meta author, og:article:author, twitter:creator, and Schema.org author prevents fragmented identity.

**Wikidata enables verified author identity.** Sites with Wikidata identifiers can use `sameAs` in Schema.org to link author Person objects to Wikidata entities. For Joseph Anady (Q138610626 with SAM.gov SDVOSB registration), this provides verified identity attribution that search engines can validate. This is particularly valuable for SDVOSB federal subcontracting work where credential verification matters.

**Privacy and safety: not every page should have a public author.** For some content (employee profiles on internal pages, controversial topics, personal essays where the author wants privacy), publishing real name in meta tags creates a permanent record visible in page source. The decision to include author requires considering the author's situation.

**Author signals influence AI search and assistants.** Beyond traditional Google ranking, AI assistants (ChatGPT, Claude, Perplexity, Gemini) cite source authorship when generating answers. Pages with clear, verified authorship are more likely to be cited; pages without are anonymous and less authoritative.

**Cost of getting it wrong.** Misconfigured author attribution produces measurable damage. Real examples:

* Arkansas Counseling and Wellness (Dr. Kristy Burton, mental health practitioner) launched without author attribution on blog posts. Google's quality signal for YMYL was low. Adding `<meta name="author" content="Dr. Kristy Burton">` plus Schema.org Article.author with credential information lifted the page quality signal within 2 months.
* Handled Tax and Advisory (Amanda Emerdinger, PTIN credential ONLY, NOT CPA/EA) had a junior developer add `<meta name="author" content="Amanda Emerdinger, CPA">` to several pages. The CPA reference violated the SDVOSB protective guardrail. Pages corrected to `<meta name="author" content="Amanda Emerdinger">` only; credential mentioned in body content with accurate PTIN language only.
* ThatDeveloperGuy blog had different author attributions across pages: some "Joseph", some "Joseph Anady", some "Joseph W. Anady", some "ThatDeveloperGuy". Fragmented author identity. Standardized to "Joseph Anady" with Schema.org sameAs linking to Wikidata Q138610626 and SAM.gov entity.
* Real estate client published agent attribution on every listing page. Stalker began targeting one agent specifically. Pages later switched to organization attribution; agent attribution moved to "Contact Us" page only.
* News site published staff writer's real name on a controversial article. Writer received harassment. Pages later switched to "Editorial Staff" attribution for sensitive content.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

The meta author tag plus the operational context:

1. **Single vs multiple author patterns**: how to handle one or many.
2. **The Schema.org Person alternative**: structured authorship.
3. **The visible byline relationship**: which signal Google prioritizes.
4. **E-E-A-T integration**: how authorship contributes to YMYL ranking.
5. **Privacy and safety**: when NOT to attribute.
6. **The Wikidata verified identity pattern**: for high authority cases.
7. **Per client decisions**: editorial vs marketing, YMYL vs not.

Section 9 covers the Wikidata pattern (relevant to Joseph's own Wikidata identity). Section 12 is the Bubbles per client decision framework.

---

## 4. THE AUTHOR MENTAL MODEL (READ THIS FIRST)

A search engine fetches your page and looks for authorship signals across multiple sources:

```
Crawler parses your page and extracts authorship signals:
   |
   v
==================== SOURCE PRIORITY ====================
   |
   |---> Visible byline in article content (HIGHEST signal)
   |     e.g., <p class="byline">By Joseph Anady</p>
   |
   |---> Schema.org structured data (Article.author or BlogPosting.author)
   |     Strong signal; includes credentials, links, sameAs to other identities
   |
   |---> Meta author tag
   |     Supporting signal; backs up byline and schema
   |
   |---> Open Graph article:author
   |     Social media specific; ensures share previews show author
   |
   |---> Twitter creator
   |     Twitter/X specific
   |
   |---> Page footer attribution
   |     Weak signal but useful for site wide authorship (e.g., "Crafted by ThatDeveloperGuy")

==================== AGGREGATION ====================
   |
   v
Google extracts an "author entity" from all signals.
   |
   v
If signals align (same author name consistently): strong author confidence.
   |
   v
If signals conflict (different names): reduced author confidence.
   |
   v
If signals missing entirely: page treated as anonymous.

==================== E-E-A-T IMPACT ====================
   |
   v
For YMYL topics (medical, financial, legal, safety):
   |
   |---> Strong author signal with credentials: high E-E-A-T
   |---> Weak or anonymous author: low E-E-A-T; ranking penalty
   |
For non YMYL topics:
   |
   |---> Author signal helpful but less critical
   |---> Strong content quality can compensate for weak author signals
```

Six rules govern the system:

1. **Editorial content gets author attribution.** Blog posts, articles, case studies, news items.
2. **Marketing pages typically attribute to the organization.** Homepage, service pages, product pages.
3. **YMYL content requires credentialed author signals.** Mental health, financial, legal, medical.
4. **All author signals must align.** Byline, meta tag, Schema.org, Open Graph must say the same person.
5. **Privacy: do not publish author identity without consent.** Especially for sensitive topics.
6. **Use Schema.org Person with sameAs for verified authors.** Wikidata, LinkedIn, professional profiles.

A correctly configured site has consistent author attribution across all signals on editorial content, no author attribution on marketing content (or org attribution), and strong credentialed author signals on YMYL content.

---

## 5. THE SINGLE VS MULTIPLE AUTHOR PATTERNS

The HTML spec allows one or many `<meta name="author">` tags, but implementations vary.

### 5.1 Single Author (Most Common)

```html
<meta name="author" content="Joseph Anady">
```

The conventional pattern. Most pages have one primary author. Use single tag.

### 5.2 Multiple Authors (Schema.org Preferred)

For pages with multiple authors, Schema.org is more reliable:

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "Article",
    "author": [
        {
            "@type": "Person",
            "name": "Joseph Anady"
        },
        {
            "@type": "Person",
            "name": "Co-author Name"
        }
    ]
}
</script>
```

The meta author tag can be repeated:

```html
<meta name="author" content="Joseph Anady">
<meta name="author" content="Co-author Name">
```

But behavior is inconsistent across consumers. Some use the first; some concatenate; some only use one. Schema.org array is unambiguous.

### 5.3 The Comma Separated Pattern (Discouraged)

Some sites use:

```html
<meta name="author" content="Joseph Anady, Co-author Name">
```

This is not standard. Search engines may treat the entire string as a single author name. Discouraged.

### 5.4 The Organization vs Individual Decision

For marketing pages, attributing to the organization rather than an individual is the convention:

```html
<meta name="author" content="ThatDeveloperGuy">
```

For editorial content, attribute to the actual person:

```html
<meta name="author" content="Joseph Anady">
```

### 5.5 The Bubbles Pattern

* **Editorial content (blog posts, case studies)**: individual person attribution.
* **Marketing pages (homepage, services, about)**: organization attribution or no attribution.
* **Service detail pages**: organization attribution.
* **Legal pages (privacy, terms)**: organization attribution.

---

## 6. META AUTHOR VS SCHEMA.ORG PERSON VS VISIBLE BYLINE

Three places authorship information appears. Each plays a different role.

### 6.1 The Visible Byline (Strongest Signal)

The rendered byline in article content:

```html
<article>
    <h1>Why Hand Coded Sites Outperform Platforms</h1>
    <p class="byline">By <a href="/about/" rel="author">Joseph Anady</a></p>
    <time datetime="2026-05-15">May 15, 2026</time>

    <p>Article content begins here...</p>
</article>
```

The `rel="author"` link attribute used to be a Google specific signal (Google Authorship, retired in 2014) but `rel="author"` as a generic indicator is still recognized by some systems.

The `<time>` element provides publication date context.

### 6.2 Schema.org Structured Data (Strong Signal)

JSON-LD with rich author information:

```html
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "headline": "Why Hand Coded Sites Outperform Platforms",
    "datePublished": "2026-05-15T08:00:00-05:00",
    "author": {
        "@type": "Person",
        "name": "Joseph Anady",
        "url": "https://thatdeveloperguy.com/about/",
        "sameAs": [
            "https://www.wikidata.org/wiki/Q138610626",
            "https://sam.gov/entity/JOSEPHWANADY",
            "https://www.linkedin.com/in/josephanady/"
        ],
        "jobTitle": "Founder",
        "worksFor": {
            "@type": "Organization",
            "name": "ThatDeveloperGuy.com"
        }
    },
    "publisher": {
        "@type": "Organization",
        "name": "ThatDeveloperGuy.com",
        "url": "https://thatdeveloperguy.com"
    }
}
</script>
```

The `sameAs` array provides identity verification: search engines can cross reference Joseph Anady with Wikidata Q138610626, SAM.gov entity, LinkedIn profile, etc.

### 6.3 Meta Author Tag (Supporting Signal)

The simpler signal in the head:

```html
<meta name="author" content="Joseph Anady">
```

Backs up the visible byline and Schema.org but does not stand alone for E-E-A-T purposes.

### 6.4 The Three Layer Coverage

For editorial content, use all three:

```html
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Why Hand Coded Sites Outperform Platforms | ThatDeveloperGuy</title>
    <meta name="description" content="Analysis of how hand coded sites outperform Wix and Squarespace.">
    <meta name="author" content="Joseph Anady">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "author": {
            "@type": "Person",
            "name": "Joseph Anady",
            "url": "https://thatdeveloperguy.com/about/",
            "sameAs": ["https://www.wikidata.org/wiki/Q138610626"]
        }
    }
    </script>
</head>
<body>
    <article>
        <h1>Why Hand Coded Sites Outperform Platforms</h1>
        <p class="byline">By <a href="/about/" rel="author">Joseph Anady</a></p>
        <time datetime="2026-05-15">May 15, 2026</time>
        <!-- ... article content ... -->
    </article>
</body>
```

All three layers say "Joseph Anady". Strong, unified author signal.

### 6.5 The Alignment Rule

The three layers MUST say the same author. Conflicts hurt:

```html
<!-- WRONG: conflicting attribution -->
<meta name="author" content="Joseph Anady">

<script type="application/ld+json">
{"author": {"@type": "Person", "name": "ThatDeveloperGuy"}}
</script>

<p class="byline">By Anonymous</p>
```

Google sees three different author signals. Confidence in any of them drops.

### 6.6 The Bubbles Alignment Pattern

For every editorial page on Bubbles client sites:

1. **Define the canonical author identity** (e.g., "Joseph Anady" or "Dr. Kristy Burton").
2. **Use that exact name** in:
   - `<meta name="author">`
   - Schema.org `author.name`
   - Visible byline text
   - Open Graph `article:author`
3. **Verify alignment** via the diagnostic commands in Section 18.

---

## 7. THE E-E-A-T CONNECTION

Experience, Expertise, Authoritativeness, Trustworthiness. Google's framework for evaluating content quality, especially for YMYL topics.

### 7.1 The YMYL Categories

Your Money Your Life topics include:

* **Health and safety**: medical advice, mental health, fitness, drug information.
* **Financial**: taxes, investments, mortgages, banking.
* **Legal**: legal advice, contract law, immigration.
* **Civic and government**: voting, government services, civil rights.
* **Safety**: emergency preparedness, child safety, security.
* **News and events**: current events, political topics.

### 7.2 The Author Component Of E-E-A-T

For YMYL content, Google looks at:

* **Experience**: has the author lived through this topic personally?
* **Expertise**: does the author have credentials, education, or training?
* **Authoritativeness**: is the author known/cited in the field?
* **Trustworthiness**: can the author be verified? Are claims accurate?

### 7.3 The Bubbles YMYL Client Implications

**Arkansas Counseling and Wellness Services (Dr. Kristy Burton)**:

YMYL mental health content. Author attribution critical.

```html
<meta name="author" content="Dr. Kristy Burton">

<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "author": {
        "@type": "Person",
        "name": "Dr. Kristy Burton",
        "jobTitle": "Licensed Therapist",
        "hasCredential": [
            {
                "@type": "EducationalOccupationalCredential",
                "name": "Licensed Professional Counselor",
                "credentialCategory": "license"
            }
        ],
        "worksFor": {
            "@type": "MedicalBusiness",
            "name": "Arkansas Counseling and Wellness Services"
        }
    }
}
</script>
```

The credentialed author signal is essential for E-E-A-T on mental health topics.

**Handled Tax and Advisory (Amanda Emerdinger, PTIN ONLY)**:

YMYL financial content. Author attribution required but with credential boundary respected.

```html
<meta name="author" content="Amanda Emerdinger">

<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "author": {
        "@type": "Person",
        "name": "Amanda Emerdinger",
        "jobTitle": "Tax Preparer",
        "hasCredential": [
            {
                "@type": "EducationalOccupationalCredential",
                "name": "PTIN Credentialed Tax Preparer",
                "credentialCategory": "certification",
                "recognizedBy": {
                    "@type": "Organization",
                    "name": "Internal Revenue Service"
                }
            }
        ]
    }
}
</script>
```

PTIN is the accurate credential. No CPA, no EA. The SDVOSB protective guardrail is maintained.

### 7.4 The Non YMYL E-E-A-T

For non YMYL content (web development tutorials, recipe blogs, hobby content), the author signal still helps but is less critical:

* Strong content quality can substitute partially.
* Anonymous content can still rank if it's clearly factual.
* The author signal still provides differentiation in competitive niches.

For Bubbles editorial content (the ThatDeveloperGuy blog), author signals are valuable but not critical to the same degree as YMYL.

---

## 8. THE PRIVACY AND SAFETY CONSIDERATION

Publishing author identity in page source exposes individuals to potential harm.

### 8.1 The Visibility Reality

```bash
# Anyone can extract author from page source
curl -s https://example.com/article/ | grep -oE 'meta name="author"[^>]+'
```

Plus Schema.org JSON-LD, Open Graph article:author, and visible byline all reveal the same information. Author attribution is fundamentally public.

### 8.2 Threats To Authors

* **Stalkers**: individuals fixated on a specific person.
* **Doxxers**: those who research personal information for harassment.
* **Disgruntled subjects**: people written about who target the author.
* **Trolls and hate groups**: organized harassment campaigns.
* **Identity theft**: enough public information enables fraud.

### 8.3 The Decision Framework

For each author attribution decision, consider:

1. **Has the author consented to public attribution?** Always check, especially for guest contributors.
2. **Does the topic invite harassment?** Controversial politics, public figures, criminal subjects.
3. **Does the author have personal safety concerns?** Domestic violence survivors, witnesses, activists.
4. **Is pseudonym appropriate?** Some authors prefer pseudonyms for personal reasons.
5. **Is organization attribution sufficient?** When personal attribution isn't necessary.

### 8.4 The Anonymous And Pseudonymous Patterns

For content where individual attribution is inappropriate:

```html
<!-- Organization attribution -->
<meta name="author" content="ThatDeveloperGuy">

<!-- Or generic editorial attribution -->
<meta name="author" content="Editorial Team">

<!-- Or pseudonym -->
<meta name="author" content="J. Smith">
```

The Schema.org equivalent:

```json
{
    "author": {
        "@type": "Organization",
        "name": "ThatDeveloperGuy"
    }
}
```

For Bubbles: when in doubt, attribute to the organization. Individual attribution should be a deliberate choice by the author.

### 8.5 The Joseph Specific Pattern

Joseph Anady is already a public figure:

* Wikidata Q138610626.
* SAM.gov entity registration.
* SDVOSB certified, federal contracting visible.
* MEGAMIND project on Wikidata Q138610666.
* Veteran status documented.

For Joseph's own work and brand pages, full personal attribution is appropriate. The information is already publicly verifiable. Privacy concerns are minimal.

For client work where Joseph is the developer but the client is the editorial author: attribute the editorial content to the client (after they consent), not to Joseph.

### 8.6 The Removal Process

If an author later wants their name removed from previously attributed content:

1. **Update meta author** to organization or pseudonym.
2. **Update Schema.org Person to Organization**.
3. **Update visible byline** in content.
4. **Update Open Graph article:author**.
5. **Wait for Google to recrawl** (or submit for indexing).
6. **Note**: existing cached versions and Wayback Machine snapshots may still show the old attribution.

---

## 9. THE WIKIDATA VERIFIED IDENTITY PATTERN

For authors with public profiles, Wikidata provides verifiable identity attribution. Joseph Anady is Q138610626.

### 9.1 The Wikidata Q Identifier

Every entity in Wikidata has a Q identifier. For Joseph:

* `Q138610626`: Joseph Anady (the person).
* `Q138610666`: MEGAMIND (the AI consciousness research project).

These identifiers persist across Wikidata changes. They're stable identity anchors.

### 9.2 The Schema.org sameAs Pattern

Schema.org `Person.sameAs` accepts an array of URLs. Including Wikidata in the array provides verification:

```json
{
    "@context": "https://schema.org",
    "@type": "Person",
    "name": "Joseph Anady",
    "sameAs": [
        "https://www.wikidata.org/wiki/Q138610626",
        "https://sam.gov/entity/JOSEPHWANADY",
        "https://www.linkedin.com/in/josephanady/",
        "https://github.com/josephanady"
    ]
}
```

Search engines (Google, Bing, others) can follow the `sameAs` URLs to verify the identity matches across platforms. The Wikidata link is particularly authoritative because Wikidata entities are community curated and verified.

### 9.3 The Knowledge Panel Connection

For high authority authors, the Schema.org + Wikidata combination can lead to Google Knowledge Panels (the box on the right side of search results showing entity information).

The path:

1. Person has Wikidata entry.
2. Person's authored pages use Schema.org with sameAs to Wikidata.
3. Person's authored content is high quality and cited.
4. Google connects the dots; creates Knowledge Panel.

For Joseph's brand visibility, this pathway is active. The MEGAMIND Wikidata entry plus Joseph's personal Wikidata entry create a knowledge graph network that Google's algorithm recognizes.

### 9.4 The Bubbles Application

For client sites where the author has Wikidata or similar public identity:

```json
{
    "author": {
        "@type": "Person",
        "name": "Author Name",
        "sameAs": [
            "https://www.wikidata.org/wiki/Q...",
            "https://linkedin.com/in/...",
            "https://professional-site.com"
        ]
    }
}
```

For client sites where authors don't have public verified identities: skip sameAs.

For ThatDeveloperGuy editorial content: use Joseph's verified identity pattern.

---

## 10. THE CORPORATE VS INDIVIDUAL ATTRIBUTION DECISION

Attribution to a person vs an organization. Different signals, different uses.

### 10.1 Individual Attribution

```html
<meta name="author" content="Joseph Anady">
```

Use for:

* Editorial blog posts.
* Articles and essays.
* Case studies with personal voice.
* Tutorials and how-to content.
* Opinion pieces.
* Personal projects.

### 10.2 Organizational Attribution

```html
<meta name="author" content="ThatDeveloperGuy">
```

Use for:

* Service pages (home, about, services, contact).
* Product pages.
* Marketing content.
* Press releases (where the organization speaks).
* Corporate communications.
* Legal pages.

### 10.3 The Mixed Pattern

Some pages have both editorial content AND service content. The decision:

* If the page primarily teaches/explains: individual attribution.
* If the page primarily sells: organizational attribution.

For ambiguous cases: organizational attribution is safer.

### 10.4 The Schema.org Both Pattern

```json
{
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "author": {
        "@type": "Person",
        "name": "Joseph Anady"
    },
    "publisher": {
        "@type": "Organization",
        "name": "ThatDeveloperGuy.com",
        "logo": {
            "@type": "ImageObject",
            "url": "https://thatdeveloperguy.com/logo.png"
        }
    }
}
```

The article has a person as author AND an organization as publisher. This is the standard pattern for editorial content on corporate sites.

### 10.5 The Bubbles Convention

For ThatDeveloperGuy and most client sites:

* **Individual author**: editorial content (blog posts, articles, case studies).
* **Organizational publisher**: always present in Schema.org.
* **Meta author tag**: matches Schema.org author (individual for editorial, organization for marketing).

---

## 11. THE EDITORIAL VS MARKETING DISTINCTION

Editorial content benefits from author attribution; marketing content typically doesn't.

### 11.1 Editorial Content Examples

* Blog posts with personal voice and analysis.
* Case studies describing specific client work.
* Tutorials and how-to guides.
* Industry commentary.
* Personal essays.
* News articles.
* Opinion pieces.

For these: include individual author attribution.

### 11.2 Marketing Content Examples

* Homepage.
* Service pages.
* Product pages.
* Pricing pages.
* About pages (organization focus).
* Contact pages.
* Landing pages for advertising campaigns.

For these: organizational attribution or no attribution.

### 11.3 The Gray Areas

Some pages straddle the line:

* **Case studies**: editorial structure but marketing intent. Use individual attribution (the case study tells a story; stories need narrators).
* **About pages**: marketing intent but personal content. Use individual attribution for founder/team focused; organization attribution for company focused.
* **Press releases**: marketing intent but news format. Organization attribution is standard.
* **Testimonials**: featuring others' words. Attribute the testimonial to the giver, not the page to anyone specific.

### 11.4 The Bubbles Application

For ThatDeveloperGuy:

* Blog at `/blog/`: individual attribution (Joseph Anady).
* Service pages at `/services/`: organizational attribution (ThatDeveloperGuy).
* Case studies at `/case-studies/`: individual attribution (Joseph Anady, narrating).
* About page at `/about/`: individual attribution (Joseph Anady, personal story).
* Contact page at `/contact/`: organizational attribution.
* Legal at `/privacy/`, `/terms/`: organizational attribution.

For client sites:

* Blog content: client's editorial author (often the owner, sometimes a contributor).
* Service pages: organizational attribution (client business name).
* About pages: depends on client preference.

---

## 12. THE BUBBLES PER CLIENT DECISION FRAMEWORK

Each client site requires a deliberate authorship strategy. The decision framework.

### 12.1 The Initial Questions

For each new client engagement:

1. **Is the business about a single person or an organization?**
   - Single person (real estate agent, therapist, lawyer): person centric attribution.
   - Organization (construction company, pest control firm): organization centric.

2. **Does the client publish editorial content (blog, articles)?**
   - Yes: define authorship strategy.
   - No: marketing only; minimal author signals needed.

3. **Is the content YMYL?**
   - Yes: credentialed author attribution essential.
   - No: standard editorial attribution.

4. **Does the author have public identity verification (Wikidata, professional license)?**
   - Yes: use Schema.org sameAs to link.
   - No: name only attribution.

5. **Privacy concerns?**
   - Yes: organization attribution or pseudonym.
   - No: real name attribution.

### 12.2 The Per Client Examples

**Arkansas Counseling and Wellness (Dr. Kristy Burton)**:

* Person centric business (therapist).
* Editorial content (blog posts about mental health).
* YMYL (mental health is sensitive).
* Author has license verification.
* Privacy: professional context, attribution appropriate.

Decision: Dr. Kristy Burton attributed with credentials. Schema.org Person with EducationalOccupationalCredential. sameAs to Arkansas LPC license verification if URL exists.

**Handled Tax and Advisory (Amanda Emerdinger, PTIN ONLY)**:

* Person centric business (tax preparer).
* Editorial content (tax tips, blog posts).
* YMYL (financial).
* Author has PTIN credential ONLY (not CPA/EA).
* Privacy: professional context, attribution appropriate.

Decision: Amanda Emerdinger attributed with PTIN credential strictly. No CPA, no EA, no Enrolled Agent. Schema.org Person with PTIN credential.

**White River Cabins (organization)**:

* Organization centric business (vacation rental).
* Limited editorial content (blog about area attractions).
* Not YMYL.

Decision: organizational attribution for service pages. Editorial content (if any) attributed to property owner or "Editorial Team".

**Greenough's Guide Service**:

* Person centric business (fishing guide).
* Editorial content (fishing reports, location guides).
* Not YMYL.

Decision: organization attribution for service pages. Editorial content attributed to the guide (Greenough) when about personal fishing experiences.

**Local Living Real Estate (Laycee Maupin)**:

* Person centric business (real estate agent).
* Limited editorial content.
* Not YMYL (commercial transaction).

Decision: Laycee Maupin attributed on about page and personal listings. Generic agent or organization attribution elsewhere.

**WeCoverUSA (federal/commercial)**:

* Organization with multiple verticals.
* Federal subcontracting context.
* Various YMYL adjacent content.

Decision: organization attribution. Specific articles can attribute to subject matter experts when applicable, with credentials.

**ThatDeveloperGuy (Joseph)**:

* Person centric business (single developer/agency).
* Active editorial content (blog).
* Mostly not YMYL.
* Public verified identity (Wikidata Q138610626, SAM.gov).

Decision: Joseph Anady attributed on editorial content with full Schema.org Person plus sameAs to Wikidata. Organization attribution (ThatDeveloperGuy) on service pages.

### 12.3 The Template Decision

```python
# /opt/bubbles/services/example.com/author_strategy.py

class AuthorStrategy:
    def __init__(self, page_type, ymyl=False):
        self.page_type = page_type
        self.ymyl = ymyl

    def get_meta_author(self):
        if self.page_type in ("blog_post", "case_study", "about_personal", "tutorial"):
            return INDIVIDUAL_AUTHOR_NAME
        else:
            return ORGANIZATION_NAME

    def get_schema_author(self):
        if self.page_type in ("blog_post", "case_study", "tutorial"):
            schema = {
                "@type": "Person",
                "name": INDIVIDUAL_AUTHOR_NAME,
                "url": INDIVIDUAL_AUTHOR_URL,
                "sameAs": INDIVIDUAL_SAME_AS_URLS,
            }
            if self.ymyl:
                schema["hasCredential"] = INDIVIDUAL_CREDENTIALS
            return schema
        else:
            return {
                "@type": "Organization",
                "name": ORGANIZATION_NAME,
                "url": ORGANIZATION_URL,
            }
```

---

## 13. THE RELATIONSHIP WITH OPEN GRAPH AND TWITTER CARDS

Author attribution extends into social media link previews via Open Graph and Twitter Cards.

### 13.1 Open Graph article:author

```html
<head>
    <meta property="og:type" content="article">
    <meta property="og:title" content="Why Hand Coded Sites Outperform Platforms">
    <meta property="og:description" content="Analysis of how hand coded sites outperform Wix and Squarespace.">
    <meta property="og:image" content="https://thatdeveloperguy.com/blog/hero.jpg">
    <meta property="og:url" content="https://thatdeveloperguy.com/blog/hand-coded-vs-platforms/">

    <!-- Article specific Open Graph -->
    <meta property="article:author" content="https://thatdeveloperguy.com/about/">
    <meta property="article:published_time" content="2026-05-15T08:00:00-05:00">
    <meta property="article:section" content="Web Development">
    <meta property="article:tag" content="hand coded">
    <meta property="article:tag" content="SEO">
</head>
```

The `article:author` property is a URL pointing to the author's page (or Facebook profile, historically). For Bubbles editorial content, point to the about page.

### 13.2 Twitter Card creator

```html
<head>
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="Why Hand Coded Sites Outperform Platforms">
    <meta name="twitter:description" content="Analysis of how hand coded sites outperform Wix and Squarespace.">
    <meta name="twitter:image" content="https://thatdeveloperguy.com/blog/twitter.jpg">

    <!-- Twitter creator handle -->
    <meta name="twitter:creator" content="@josephanady">
    <meta name="twitter:site" content="@thatdeveloperguy">
</head>
```

`twitter:creator` is the article author's Twitter/X handle.
`twitter:site` is the publishing site's Twitter/X handle.

### 13.3 The Alignment Pattern

For editorial content, all four layers should agree:

```html
<!-- HTML head -->
<meta name="author" content="Joseph Anady">

<!-- Open Graph -->
<meta property="article:author" content="https://thatdeveloperguy.com/about/">

<!-- Twitter -->
<meta name="twitter:creator" content="@josephanady">

<!-- Schema.org -->
<script type="application/ld+json">
{"author": {"@type": "Person", "name": "Joseph Anady", "url": "https://thatdeveloperguy.com/about/"}}
</script>

<!-- Visible byline -->
<p class="byline">By <a href="/about/" rel="author">Joseph Anady</a></p>
```

Five layers, all attributing the same author. Strong consistency.

---

## 14. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario.

### 14.1 Editorial blog post (Joseph's ThatDeveloperGuy pattern)

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Why Hand Coded Sites Outperform Platforms | ThatDeveloperGuy</title>
    <meta name="description" content="Why hand coded websites outperform Wix and Squarespace for small business SEO. 5 case studies from Northwest Arkansas.">
    <meta name="author" content="Joseph Anady">

    <!-- Open Graph -->
    <meta property="og:type" content="article">
    <meta property="og:title" content="Why Hand Coded Sites Outperform Platforms">
    <meta property="og:description" content="Analysis of how hand coded sites outperform platforms.">
    <meta property="article:author" content="https://thatdeveloperguy.com/about/">
    <meta property="article:published_time" content="2026-05-15T08:00:00-05:00">

    <!-- Twitter -->
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:creator" content="@josephanady">
    <meta name="twitter:site" content="@thatdeveloperguy">

    <!-- Schema.org BlogPosting with verified Wikidata identity -->
    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "headline": "Why Hand Coded Sites Outperform Platforms",
        "datePublished": "2026-05-15T08:00:00-05:00",
        "author": {
            "@type": "Person",
            "name": "Joseph Anady",
            "url": "https://thatdeveloperguy.com/about/",
            "sameAs": [
                "https://www.wikidata.org/wiki/Q138610626",
                "https://sam.gov/entity/JOSEPHWANADY"
            ]
        },
        "publisher": {
            "@type": "Organization",
            "name": "ThatDeveloperGuy.com",
            "url": "https://thatdeveloperguy.com",
            "logo": {
                "@type": "ImageObject",
                "url": "https://thatdeveloperguy.com/logo.png"
            }
        }
    }
    </script>
</head>
<body>
    <article>
        <h1>Why Hand Coded Sites Outperform Platforms</h1>
        <p class="byline">By <a href="/about/" rel="author">Joseph Anady</a></p>
        <time datetime="2026-05-15">May 15, 2026</time>
        <!-- ... content ... -->
    </article>
</body>
</html>
```

### 14.2 YMYL editorial: Arkansas Counseling and Wellness (Dr. Kristy Burton)

```html
<head>
    <meta charset="utf-8">
    <title>Understanding Anxiety in Northwest Arkansas | Arkansas Counseling and Wellness</title>
    <meta name="description" content="Common anxiety symptoms and when to seek help. Dr. Kristy Burton, licensed therapist in Bentonville AR.">
    <meta name="author" content="Dr. Kristy Burton">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "MedicalWebPage",
        "headline": "Understanding Anxiety in Northwest Arkansas",
        "author": {
            "@type": "Person",
            "name": "Dr. Kristy Burton",
            "jobTitle": "Licensed Therapist",
            "hasCredential": [
                {
                    "@type": "EducationalOccupationalCredential",
                    "name": "Licensed Professional Counselor",
                    "credentialCategory": "license"
                }
            ],
            "worksFor": {
                "@type": "MedicalBusiness",
                "name": "Arkansas Counseling and Wellness Services"
            }
        },
        "reviewedBy": {
            "@type": "Person",
            "name": "Dr. Kristy Burton"
        }
    }
    </script>
</head>
<body>
    <article>
        <h1>Understanding Anxiety in Northwest Arkansas</h1>
        <p class="byline">
            By <a href="/about/" rel="author">Dr. Kristy Burton</a>, Licensed Therapist
        </p>
        <time datetime="2026-05-15">May 15, 2026</time>
        <!-- ... content ... -->
    </article>
</body>
```

### 14.3 YMYL editorial: Handled Tax (Amanda Emerdinger, PTIN ONLY guardrail)

```html
<head>
    <meta charset="utf-8">
    <title>2026 Tax Updates Affecting Small Businesses | Handled Tax & Advisory</title>
    <meta name="description" content="Key 2026 tax law changes for small businesses. Amanda Emerdinger, PTIN credentialed tax preparer.">
    <meta name="author" content="Amanda Emerdinger">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "headline": "2026 Tax Updates Affecting Small Businesses",
        "author": {
            "@type": "Person",
            "name": "Amanda Emerdinger",
            "jobTitle": "Tax Preparer",
            "hasCredential": [
                {
                    "@type": "EducationalOccupationalCredential",
                    "name": "PTIN Credentialed Tax Preparer",
                    "credentialCategory": "certification",
                    "recognizedBy": {
                        "@type": "Organization",
                        "name": "Internal Revenue Service"
                    }
                }
            ],
            "worksFor": {
                "@type": "FinancialService",
                "name": "Handled Tax & Advisory",
                "legalName": "Accountability Tax Services PLLC"
            }
        }
    }
    </script>
</head>
<body>
    <article>
        <h1>2026 Tax Updates Affecting Small Businesses</h1>
        <p class="byline">
            By Amanda Emerdinger, PTIN credentialed tax preparer
        </p>
        <!-- NO CPA, NO EA, NO Enrolled Agent in attribution -->
        <!-- ... content ... -->
    </article>
</body>
```

### 14.4 Marketing page (organization attribution)

```html
<head>
    <meta charset="utf-8">
    <title>Web Development Services | ThatDeveloperGuy</title>
    <meta name="description" content="Custom web development for Northwest Arkansas businesses. SDVOSB certified.">
    <meta name="author" content="ThatDeveloperGuy">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "WebPage",
        "publisher": {
            "@type": "Organization",
            "name": "ThatDeveloperGuy.com",
            "url": "https://thatdeveloperguy.com"
        }
    }
    </script>
</head>
```

### 14.5 Case study (individual attribution; storytelling)

```html
<head>
    <meta charset="utf-8">
    <title>How Greenough's Guide Service Launched in 30 Days | ThatDeveloperGuy Case Study</title>
    <meta name="description" content="How we built and launched the booking site for Greenough's Guide Service in 30 days. Hand coded, mobile first, Google Places API integration.">
    <meta name="author" content="Joseph Anady">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "Article",
        "headline": "How Greenough's Guide Service Launched in 30 Days",
        "author": {
            "@type": "Person",
            "name": "Joseph Anady",
            "sameAs": ["https://www.wikidata.org/wiki/Q138610626"]
        },
        "about": {
            "@type": "Organization",
            "name": "Greenough's Guide Service"
        }
    }
    </script>
</head>
```

### 14.6 Organization attribution for privacy

For employees who don't want public attribution:

```html
<head>
    <meta name="author" content="Editorial Team">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "author": {
            "@type": "Organization",
            "name": "Editorial Team"
        },
        "publisher": {
            "@type": "Organization",
            "name": "Client Company"
        }
    }
    </script>
</head>
<body>
    <article>
        <p class="byline">By Editorial Team</p>
        <!-- ... content ... -->
    </article>
</body>
```

### 14.7 Multiple authors (Schema.org array, not multiple meta tags)

```html
<head>
    <meta name="author" content="Joseph Anady">
    <!-- Primary author only in meta tag -->

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "author": [
            {
                "@type": "Person",
                "name": "Joseph Anady",
                "sameAs": ["https://www.wikidata.org/wiki/Q138610626"]
            },
            {
                "@type": "Person",
                "name": "Co-author Name"
            }
        ]
    }
    </script>
</head>
<body>
    <article>
        <p class="byline">By Joseph Anady and Co-author Name</p>
        <!-- ... content ... -->
    </article>
</body>
```

### 14.8 The author template macro (Jinja2)

```html
{# /opt/bubbles/services/example.com/templates/_author.html #}
{% macro author_block(author_data, page_type='blog_post', ymyl=false) %}
    {% if page_type in ('blog_post', 'case_study', 'tutorial', 'about_personal') %}
        <meta name="author" content="{{ author_data.name }}">

        <script type="application/ld+json">
        {
            "@context": "https://schema.org",
            "@type": "{{ author_data.schema_type|default('BlogPosting') }}",
            "author": {
                "@type": "Person",
                "name": "{{ author_data.name }}",
                {% if author_data.url %}"url": "{{ author_data.url }}",{% endif %}
                {% if author_data.same_as %}
                "sameAs": {{ author_data.same_as|tojson }},
                {% endif %}
                {% if ymyl and author_data.credentials %}
                "hasCredential": {{ author_data.credentials|tojson }},
                {% endif %}
                {% if author_data.job_title %}"jobTitle": "{{ author_data.job_title }}",{% endif %}
                "worksFor": {{ author_data.works_for|tojson }}
            }
        }
        </script>
    {% else %}
        <meta name="author" content="{{ author_data.organization_name }}">
    {% endif %}
{% endmacro %}
```

---

## 15. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles author configuration with all variants.

### 15.1 The Author Data Registry

```python
# /opt/bubbles/services/example.com/authors.py

AUTHORS = {
    "joseph_anady": {
        "name": "Joseph Anady",
        "url": "https://thatdeveloperguy.com/about/",
        "same_as": [
            "https://www.wikidata.org/wiki/Q138610626",
            "https://sam.gov/entity/JOSEPHWANADY",
        ],
        "job_title": "Founder",
        "credentials": None,
        "works_for": {
            "@type": "Organization",
            "name": "ThatDeveloperGuy.com",
            "url": "https://thatdeveloperguy.com",
        },
    },
    "kristy_burton": {
        "name": "Dr. Kristy Burton",
        "url": "https://arcounselingandwellness.com/about/",
        "same_as": None,
        "job_title": "Licensed Therapist",
        "credentials": [
            {
                "@type": "EducationalOccupationalCredential",
                "name": "Licensed Professional Counselor",
                "credentialCategory": "license",
            }
        ],
        "works_for": {
            "@type": "MedicalBusiness",
            "name": "Arkansas Counseling and Wellness Services",
        },
    },
    "amanda_emerdinger": {
        "name": "Amanda Emerdinger",
        "url": "https://handledtax.com/about/",
        "same_as": None,
        "job_title": "Tax Preparer",  # NOT CPA, NOT EA
        "credentials": [
            {
                "@type": "EducationalOccupationalCredential",
                "name": "PTIN Credentialed Tax Preparer",
                "credentialCategory": "certification",
                "recognizedBy": {
                    "@type": "Organization",
                    "name": "Internal Revenue Service",
                },
            }
        ],
        "works_for": {
            "@type": "FinancialService",
            "name": "Handled Tax & Advisory",
            "legalName": "Accountability Tax Services PLLC",
        },
    },
    # ... per client author
}

ORGANIZATION_AUTHOR = "ThatDeveloperGuy"  # for marketing pages
```

### 15.2 The FastAPI Integration

```python
from fastapi import FastAPI, Request
from authors import AUTHORS, ORGANIZATION_AUTHOR

app = FastAPI()

@app.get("/blog/{post_slug}/")
async def blog_post(post_slug: str, request: Request):
    post = get_post(post_slug)
    author = AUTHORS[post.author_key]
    ymyl = post.is_ymyl

    return templates.TemplateResponse(
        "blog_post.html",
        {
            "request": request,
            "post": post,
            "author": author,
            "ymyl": ymyl,
        }
    )

@app.get("/services/{service_slug}/")
async def service_page(service_slug: str, request: Request):
    # Marketing pages: organization attribution
    return templates.TemplateResponse(
        "service_page.html",
        {
            "request": request,
            "service": get_service(service_slug),
            "meta_author": ORGANIZATION_AUTHOR,
        }
    )
```

### 15.3 The Audit Workflow

```bash
#!/bin/bash
# /usr/local/bin/author-audit.sh
# Audit author attribution across a Bubbles client site

URL=$1

echo "=== Author audit for $URL ==="

# Get all sitemap URLs
URLS=$(curl -s "$URL/sitemap.xml" | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g')

# For each URL, check author signals
for u in $URLS; do
    META=$(curl -s "$u" 2>/dev/null | grep -oE 'meta name="author"[^>]+' | head -1 | sed 's/.*content="\([^"]*\)".*/\1/')
    HAS_SCHEMA_AUTHOR=$(curl -s "$u" 2>/dev/null | grep -c '"author"')
    BYLINE_COUNT=$(curl -s "$u" 2>/dev/null | grep -c 'class="byline"')

    if [ -n "$META" ]; then
        echo "$u"
        echo "  meta author: $META"
        echo "  schema author: $HAS_SCHEMA_AUTHOR occurrences"
        echo "  visible byline: $BYLINE_COUNT occurrences"
    fi
done

echo "=== End audit ==="
```

### 15.4 The Alignment Verification

```bash
#!/bin/bash
# /usr/local/bin/author-alignment-check.sh
# Verify meta author, Schema.org author, and visible byline all agree

URL=$1

META=$(curl -s "$URL" | grep -oE 'meta name="author"[^>]+' | head -1 | sed 's/.*content="\([^"]*\)".*/\1/')

SCHEMA=$(curl -s "$URL" | python3 -c "
import sys, json, re
text = sys.stdin.read()
matches = re.findall(r'<script type=\"application/ld\+json\">(.*?)</script>', text, re.DOTALL)
for m in matches:
    try:
        data = json.loads(m)
        if isinstance(data, dict) and 'author' in data:
            author = data['author']
            if isinstance(author, dict):
                print(author.get('name', ''))
                break
            elif isinstance(author, list) and author:
                print(author[0].get('name', ''))
                break
    except:
        pass
")

BYLINE=$(curl -s "$URL" | grep -oE 'class="byline"[^>]*>[^<]*<a[^>]*>[^<]+</a>' | head -1 | sed 's/.*>\([^<]*\)<\/a>.*/\1/')

echo "URL: $URL"
echo "  meta author:    $META"
echo "  schema author:  $SCHEMA"
echo "  visible byline: $BYLINE"

if [ "$META" = "$SCHEMA" ] && [ "$SCHEMA" = "$BYLINE" ]; then
    echo "  ALIGNMENT: OK"
else
    echo "  ALIGNMENT: MISMATCH"
fi
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/author-audit.sh
sudo chmod +x /usr/local/bin/author-alignment-check.sh
```

### 15.5 After Template Changes

```bash
# Restart FastAPI sidecar to pick up template changes
systemctl restart fastapi-sidecar

# Verify alignment on a sample page
author-alignment-check.sh https://thatdeveloperguy.com/blog/sample-post/
```

---

## 16. AUDIT CHECKLIST

Run through these 50 items for any production deployment.

### Core author tag

1. [ ] Editorial pages (blog, articles, case studies) have `<meta name="author">`.
2. [ ] Marketing pages (homepage, services) have either organization author OR no author.
3. [ ] Legal pages have organization author or no author.
4. [ ] No conflicting `<meta name="author">` tags on a single page (typically one per page).

### Editorial content

5. [ ] Blog posts attributed to individual person.
6. [ ] Case studies attributed to individual person.
7. [ ] Tutorials attributed to individual person.
8. [ ] News articles attributed to individual person.

### Marketing content

9. [ ] Service pages attributed to organization (or no author).
10. [ ] Pricing pages attributed to organization.
11. [ ] Contact pages attributed to organization or omitted.
12. [ ] Landing pages attributed to organization.

### YMYL content (mental health, financial, legal)

13. [ ] YMYL pages attributed to credentialed individuals.
14. [ ] Credentials accurately represented (not overstated).
15. [ ] **Handled Tax content uses PTIN credentialed terminology, NEVER CPA or EA.**
16. [ ] Counseling content includes therapist license info.

### Schema.org alignment

17. [ ] Schema.org Article.author or BlogPosting.author matches meta author.
18. [ ] Schema.org includes Person.url where appropriate.
19. [ ] Schema.org includes Person.jobTitle for editorial bylines.
20. [ ] Schema.org includes EducationalOccupationalCredential for YMYL.
21. [ ] Schema.org includes Organization.publisher.

### Visible byline alignment

22. [ ] Visible byline present on editorial content.
23. [ ] Byline text matches meta author and Schema.org author.
24. [ ] Byline includes link to author page.
25. [ ] Byline shows credentials for YMYL content.

### Open Graph alignment

26. [ ] og:type set to "article" for editorial content.
27. [ ] article:author set with author page URL.
28. [ ] article:published_time set with ISO 8601 date.

### Twitter Card alignment

29. [ ] twitter:creator set to author's Twitter/X handle.
30. [ ] twitter:site set to organization's Twitter/X handle.

### Wikidata verification

31. [ ] Where author has Wikidata entry, Schema.org sameAs includes Wikidata URL.
32. [ ] Joseph Anady attribution includes sameAs to Q138610626 where appropriate.
33. [ ] sameAs URLs are valid and verifiable.

### Privacy and safety

34. [ ] No employee attribution without consent.
35. [ ] No personal info beyond what author approved.
36. [ ] Pseudonym option offered to contributors when appropriate.

### Consistency across the site

37. [ ] Author name consistent across all attributed pages (no "Joseph" vs "Joseph Anady" vs "Joseph W. Anady" fragments).
38. [ ] Author URL points to a real about page.
39. [ ] About page has substantive author bio.

### Cross cutting

40. [ ] `meta charset="utf-8"` precedes meta author (per framework-html-meta-charset.md).
41. [ ] Author attribution doesn't appear on noindex pages (unnecessary).

### Multiple authors

42. [ ] Multi author content uses Schema.org array.
43. [ ] Multi author content lists primary author first in array.
44. [ ] Multiple `<meta name="author">` tags avoided where possible.

### Tooling and process

45. [ ] AUTHORS registry maintained per client site.
46. [ ] Template macro emits aligned author signals across all layers.
47. [ ] Pre deploy alignment check runs.
48. [ ] `author-audit.sh` script in `/usr/local/bin/`.

### Documentation

49. [ ] Author strategy documented per client.
50. [ ] Privacy considerations documented.

A site that passes all 50 has correctly configured author attribution for E-E-A-T signals, social media link previews, and editorial credibility.

---

## 17. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Marketing pages with personal author attribution.**
Symptom: every page on the site attributes to "Joseph Anady" including service pages and pricing.
Why it breaks: dilutes the editorial vs marketing signal; less effective.
Fix: organization attribution for marketing content; individual attribution only for editorial.

**Pitfall 2: Credentialed overstatement on YMYL content.**
Symptom: tax preparer described as "CPA" or "Enrolled Agent" without actually being credentialed.
Why it breaks: SDVOSB protective concern; legal risk; FTC compliance.
Fix: use accurate credentials only. PTIN ≠ CPA/EA. Use the exact credential terminology.

**Pitfall 3: Conflicting author signals across layers.**
Symptom: meta author says "Joseph", Schema.org says "Joseph Anady", byline says "J. Anady".
Why it breaks: fragmented author identity; Google confidence drops.
Fix: pick one canonical name; use it everywhere.

**Pitfall 4: Missing author on editorial content.**
Symptom: blog posts have no author attribution anywhere.
Why it breaks: weak E-E-A-T signal; lower content quality assessment.
Fix: add meta author, Schema.org author, and visible byline.

**Pitfall 5: Schema.org author without supporting layers.**
Symptom: rich Schema.org Person but no meta author tag or visible byline.
Why it breaks: missed reinforcement; less robust signal.
Fix: add meta author and visible byline.

**Pitfall 6: Author tag on noindex pages.**
Symptom: login page has `<meta name="author" content="Editorial Team">`.
Why it breaks: wasted metadata; noindex page won't be displayed in search.
Fix: omit author on noindex pages.

**Pitfall 7: Author URL pointing to homepage.**
Symptom: author Person.url is "https://thatdeveloperguy.com/" not "/about/".
Why it breaks: homepage is not a specific author page; weak identity signal.
Fix: link to a dedicated author/about page.

**Pitfall 8: No about page for attributed author.**
Symptom: author attribution exists but no /about/ page describes the author.
Why it breaks: incomplete identity signal; users can't verify the author.
Fix: build a substantive about page including bio, credentials, contact.

**Pitfall 9: Pseudonym attribution on YMYL content.**
Symptom: medical advice attributed to "Editorial Team".
Why it breaks: weak E-E-A-T for YMYL; rankings suffer.
Fix: real, credentialed author for YMYL content.

**Pitfall 10: Including author identity without consent.**
Symptom: employee's real name attributed to a controversial article they wrote on behalf of the company.
Why it breaks: privacy violation; potential harm to author.
Fix: get explicit consent; offer pseudonym option.

**Pitfall 11: Multiple competing meta author tags.**
Symptom: page has both `<meta name="author" content="A">` and `<meta name="author" content="B">`.
Why it breaks: behavior is inconsistent across consumers.
Fix: single meta author tag; use Schema.org array for multiple authors.

**Pitfall 12: Schema.org author as a string instead of Person object.**
Symptom: `"author": "Joseph Anady"` instead of `"author": {"@type": "Person", "name": "Joseph Anady"}`.
Why it breaks: less structured information; can't add credentials, sameAs, etc.
Fix: use the full Person object syntax.

**Pitfall 13: Inconsistent author across the same client's pages.**
Symptom: blog post 1 attributed to "Jane Doe", blog post 2 to "J. Doe", blog post 3 to "Jane".
Why it breaks: search engines can't aggregate author authority.
Fix: standardize the canonical name; use it on every page.

**Pitfall 14: Privacy violation via Schema.org sameAs.**
Symptom: author's personal Facebook, Instagram, Twitter all linked via sameAs.
Why it breaks: more personal info exposed than necessary.
Fix: only include public, verifiable, professional identity links (Wikidata, LinkedIn, professional site).

**Pitfall 15: Federal subcontract page attributed to staff but credentials misrepresented.**
Symptom: page attributes to "Engineer" without engineering credential, or "Project Manager" with overstated role.
Why it breaks: federal compliance concern; potential audit issue.
Fix: accurate role and credential attribution.

---

## 18. DIAGNOSTIC COMMANDS

Reference of commands useful for author attribution investigation.

### Inspect a single URL

```bash
# Extract meta author
curl -s https://example.com/article/ | grep -oE 'meta name="author"[^>]+' | head -1

# Extract Schema.org author (requires parsing JSON-LD)
curl -s https://example.com/article/ | python3 -c "
import sys, json, re
text = sys.stdin.read()
for match in re.findall(r'<script type=\"application/ld\+json\">(.*?)</script>', text, re.DOTALL):
    try:
        data = json.loads(match)
        if isinstance(data, dict) and 'author' in data:
            print(json.dumps(data['author'], indent=2))
    except:
        pass
"

# Extract visible byline
curl -s https://example.com/article/ | grep -oE 'class="byline"[^<]*<[^>]+>[^<]+'
```

### Check alignment

```bash
URL=$1
author-alignment-check.sh "$URL"
```

### Bulk audit

```bash
# Audit all editorial URLs (assumes /blog/ contains editorial content)
for url in $(curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | grep "/blog/"); do
    META=$(curl -s "$url" 2>/dev/null | grep -oE 'meta name="author"[^>]+' | head -1 | sed 's/.*content="\([^"]*\)".*/\1/')
    echo "$url -> $META"
done
```

### Find missing author on editorial pages

```bash
# Pages in /blog/ that lack meta author
for url in $(curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | grep "/blog/"); do
    HAS=$(curl -s "$url" 2>/dev/null | grep -c 'meta name="author"')
    if [ "$HAS" = "0" ]; then
        echo "MISSING AUTHOR: $url"
    fi
done
```

### Verify Wikidata sameAs

```bash
# For Joseph's content, verify Wikidata link is present
URL=$1
HAS_WIKIDATA=$(curl -s "$URL" | grep -c "wikidata.org/wiki/Q138610626")
[ "$HAS_WIKIDATA" -gt 0 ] && echo "OK: Wikidata sameAs present" || echo "MISSING: Wikidata sameAs"
```

### Find inconsistent author names

```bash
# Look for name variations on a client's editorial content
curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | grep "/blog/" | while read url; do
    META=$(curl -s "$url" 2>/dev/null | grep -oE 'meta name="author"[^>]+' | head -1 | sed 's/.*content="\([^"]*\)".*/\1/')
    echo "$META"
done | sort | uniq -c | sort -rn
# Should show one author with consistent count, not multiple variations
```

### Check credential mentions for YMYL clients (Handled Tax guardrail)

```bash
# For Handled Tax site: ensure CPA/EA never appears in attribution
URL=https://handledtax.com
for u in $(curl -s "$URL/sitemap.xml" | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g'); do
    HAS_CPA=$(curl -s "$u" 2>/dev/null | grep -c -iE "amanda.*(cpa|enrolled.*agent|ea\b)")
    if [ "$HAS_CPA" -gt 0 ]; then
        echo "VIOLATION: $u contains credential overstatement"
    fi
done
```

### Browser verification

In Chrome DevTools:

1. Open editorial page.
2. View page source (Cmd/Ctrl + U).
3. Find `<meta name="author">`.
4. Find Schema.org `<script type="application/ld+json">`.
5. Find visible byline element.
6. Confirm all three say the same author.

Use Google's Rich Results Test:
- https://search.google.com/test/rich-results
- Paste URL.
- Verify Schema.org author parses correctly.

### After template changes

```bash
# Restart FastAPI sidecar
systemctl restart fastapi-sidecar

# Verify on sample page
author-alignment-check.sh https://example.com/blog/sample-post/

# Run audit
author-audit.sh https://example.com/
```

---

## 19. CROSS-REFERENCES

* [framework-html-meta-description.md](framework-html-meta-description.md): meta description and meta author both belong in `<head>`; align them with the same editorial voice.
* [framework-html-meta-charset.md](framework-html-meta-charset.md): author names with international characters require UTF-8 (e.g., Dr. Kristy Burton works fine; Marshallese authors with diacritics need full UTF-8 chain).
* [framework-html-meta-robots.md](framework-html-meta-robots.md): author attribution unnecessary on noindex pages.
* [framework-html-meta-keywords.md](framework-html-meta-keywords.md): like keywords, author is exposed in page source; privacy considerations apply.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): no author HTTP header exists; meta tag is the only HTML mechanism.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. E-E-A-T is the framework where author signals matter most.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including author strategy per client.
* Google's E-E-A-T documentation: https://developers.google.com/search/docs/fundamentals/creating-helpful-content
* Google's Search Quality Rater Guidelines: https://services.google.com/fh/files/misc/hsw-sqrg.pdf
* Schema.org Person type: https://schema.org/Person
* Schema.org Article.author: https://schema.org/Article
* Schema.org EducationalOccupationalCredential: https://schema.org/EducationalOccupationalCredential
* MDN meta name=author: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name
* Dublin Core Metadata Initiative: https://www.dublincore.org/specifications/dublin-core/
* Wikidata: https://www.wikidata.org/
* Wikidata Q138610626 (Joseph Anady): https://www.wikidata.org/wiki/Q138610626

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Editorial content**: individual author with Schema.org Person.
**Marketing content**: organization attribution or none.
**YMYL content**: credentialed individual author with EducationalOccupationalCredential.

### The canonical pattern (editorial with verified identity)

```html
<head>
    <meta name="author" content="Joseph Anady">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "author": {
            "@type": "Person",
            "name": "Joseph Anady",
            "url": "https://thatdeveloperguy.com/about/",
            "sameAs": ["https://www.wikidata.org/wiki/Q138610626"]
        },
        "publisher": {
            "@type": "Organization",
            "name": "ThatDeveloperGuy.com"
        }
    }
    </script>
</head>
<body>
    <article>
        <p class="byline">By <a href="/about/" rel="author">Joseph Anady</a></p>
    </article>
</body>
```

### Five rules to memorize

1. **Editorial: individual. Marketing: organization. YMYL: credentialed individual.**
2. **All author signals must align: meta, Schema.org, byline, Open Graph, Twitter.**
3. **For YMYL Bubbles clients: Handled Tax = PTIN ONLY, not CPA/EA. Counseling = LPC.**
4. **Use Wikidata sameAs for verified identity (Joseph Q138610626).**
5. **Privacy: get author consent. Pseudonym option when appropriate.**

### Five commands every operator should know

```bash
# 1. Check meta author for a URL
curl -s URL | grep -oE 'meta name="author"[^>]+' | head -1

# 2. Verify alignment across layers
author-alignment-check.sh URL

# 3. Find missing author on editorial pages
for url in $(curl -s URL/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | grep "/blog/"); do
    HAS=$(curl -s "$url" | grep -c 'meta name="author"')
    [ "$HAS" = "0" ] && echo "MISSING: $url"
done

# 4. Verify credential guardrails (Handled Tax)
curl -s URL | grep -c -iE "amanda.*(cpa|enrolled.*agent|ea\b)"
# Expected: 0

# 5. Apply changes
systemctl restart fastapi-sidecar
```

### Three end to end tests

```bash
# 1. Editorial page has all three layers (meta + schema + byline)
URL=https://thatdeveloperguy.com/blog/sample/
META=$(curl -s "$URL" | grep -c 'meta name="author"')
SCHEMA=$(curl -s "$URL" | grep -c '"author"')
BYLINE=$(curl -s "$URL" | grep -c 'class="byline"')
[ "$META" -ge 1 ] && [ "$SCHEMA" -ge 1 ] && [ "$BYLINE" -ge 1 ] && echo "OK: all three layers" || echo "FAIL"

# 2. Marketing page has organization or no attribution
URL=https://thatdeveloperguy.com/services/
META=$(curl -s "$URL" | grep -oE 'meta name="author" content="[^"]+"' | sed 's/.*content="\([^"]*\)".*/\1/')
if [ -z "$META" ] || [ "$META" = "ThatDeveloperGuy" ]; then
    echo "OK: organization or omitted"
else
    echo "FAIL: $META on marketing page"
fi

# 3. YMYL credential guardrail respected (Handled Tax)
URL=https://handledtax.com
VIOLATIONS=$(curl -s "$URL/sitemap.xml" | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | while read u; do
    curl -s "$u" 2>/dev/null | grep -iE "amanda.*(cpa|enrolled.*agent|ea\b)" > /dev/null && echo "$u"
done | wc -l)
[ "$VIOLATIONS" = "0" ] && echo "OK: no credential overstatement" || echo "FAIL: $VIOLATIONS violations"
```

If all three pass, the author attribution layer is correctly wired across the meta, structured data, and visible byline signals with privacy and credential guardrails respected.

---

End of framework-html-meta-author.md.
