# framework-html-meta-keywords.md

Comprehensive reference for HTML `<meta name="keywords">`, the historically central SEO signal that Google publicly stopped using in 2009 but that is still parsed by some search engines and tools. Covers the historical context (rise in the 1990s, fall to keyword stuffing abuse, 2009 Google official rejection), current search engine behavior across Google, Bing, Yandex, Baidu, DuckDuckGo, and others, legitimate modern use cases (internal site search relevance signals, regional search engines, niche documentation systems), the Bing anti-spam signal consideration (excessive irrelevant keywords flags spam), the competitor intelligence reality (publishing your target keywords reveals strategy), why CMS systems still auto-generate the tag from page categories or tags, the alternatives that actually work for signaling page focus (title, H1, description, schema.org keywords property, internal link anchor text), and the Bubbles decision to omit it entirely from hand coded sites. Built for Bubbles (Debian, Nginx 1.26+, FastAPI sidecar on port 9090, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front).

**This is the fifth framework in the HTML signal track**, following framework-html-meta-robots.md, framework-html-meta-charset.md, framework-html-meta-viewport.md, and framework-html-meta-description.md. Companion to the 12 wire layer frameworks.

Audience: humans evaluating whether legacy templates need meta keywords cleanup, AI assistants generating HTML head sections that should NOT include meta keywords, SEO operators migrating client sites from WordPress or Squarespace where meta keywords plugins may auto generate the tag, and anyone troubleshooting "should we add meta keywords back to our site", "WordPress Yoast suggests adding focus keywords", "our 2008 template still has meta keywords with 50 terms", or "Bing flagged our pages as low quality with no apparent reason".

---

## TABLE OF CONTENTS

1. Definition
2. Why It (No Longer) Matters
3. What This Covers
4. The Mental Model: It Is Dead But Still Parsed
5. The Historical Context (1990s Rise, 2009 Fall)
6. Current Search Engine Behavior
7. Legitimate Modern Uses
8. The Bing Anti Spam Signal Consideration
9. The Competitive Intelligence Reality
10. Why Some Sites Still Emit It
11. Better Alternatives For Signaling Page Focus
12. The Internal Site Search Use Case
13. The Bubbles Decision: Omit Entirely
14. The CMS Auto Generation Problem
15. Asset Class And Use Case Recipes
16. Bubbles Standard Pattern (paste ready)
17. Audit Checklist
18. Common Pitfalls
19. Diagnostic Commands
20. Cross-References

---

## 1. DEFINITION

`<meta name="keywords">` is the HTML directive originally intended to declare a list of relevant keywords for the page, helping search engines understand the page's topical focus.

```html
<head>
    <meta name="keywords" content="web development, hand coded sites, northwest arkansas, sdvosb, bentonville">
</head>
```

Three structural facts shape how the directive works in 2026:

* **Google ignores it for ranking.** Per Matt Cutts (Google) in a September 2009 official statement: "Google does not use the keywords meta tag in our web search ranking." This has been reconfirmed multiple times through 2024 and remains current as of 2026.
* **Other search engines vary.** Bing parses it as one of many signals but primarily for anti spam detection. Yandex (Russia) and Baidu (China) still consider it. DuckDuckGo derives from multiple sources including Bing.
* **It is still parsed but rarely acted on.** The tag is read during page crawling, the values are stored in some indexes, but for Google specifically the values do not influence ranking.

For Bubbles client sites in 2026, the convention is to **omit `<meta name="keywords">` entirely**. The tag provides no ranking benefit on Google (the majority of search traffic), may trigger Bing anti spam scrutiny if abused, and reveals your target keyword strategy to anyone viewing page source.

---

## 2. WHY IT (NO LONGER) MATTERS

Eight independent considerations clarify why meta keywords usage decision is the inverse of most metadata: the default is omit, with narrow exceptions.

**Google's 2009 announcement is still current.** Matt Cutts published a video and blog post in September 2009 explicitly stating: "Our web search disregards keyword metatags completely. They simply don't have any effect in our search ranking at present." This has not changed. Google's official Search Central documentation as of 2026 does not mention meta keywords as a recommended optimization.

**Keyword stuffing abuse killed the signal.** In the mid 1990s through early 2000s, meta keywords was the central ranking signal. Webmasters quickly learned to stuff dozens or hundreds of keywords (including competitor brand names, popular search terms unrelated to page content). The signal became noise. Google's algorithm shift to content based signals (PageRank, semantic relevance, user behavior) made meta keywords obsolete by design.

**Modern AI and semantic search algorithms don't need it.** Google's BERT, MUM, and subsequent natural language models extract topical understanding directly from page content. Bing's similar models do the same. Telling Google "this page is about web development" via meta keywords adds nothing the algorithm doesn't already extract from the title, headings, and body text.

**Bing's anti spam consideration cuts both ways.** Per Bing's webmaster guidelines, excessive or irrelevant meta keywords can flag a page as low quality. A page with 50 keywords (especially competitor brand names or unrelated popular terms) is more likely to be deprioritized than helped. Conservative use (3-5 relevant terms) is neutral; aggressive use is negative.

**The competitive intelligence reveal is real.** Anyone who views your page source sees your meta keywords. Competitors can extract these to understand your keyword strategy. Tools like SimilarWeb, SEMrush, and Ahrefs scrape these. Publishing meta keywords is publishing your strategy.

**Regional and niche search engines may still use it.** Yandex (Russia) and Baidu (China) include meta keywords as a ranking factor. Internal site search engines (Algolia, ElasticSearch, Solr, internal databases) often use meta keywords as one relevance signal. For sites targeting these markets specifically, the tag has some value.

**Yoast and other WordPress plugins removed it.** Yoast SEO removed the meta keywords feature in 2014 (version 1.5). Other major SEO plugins followed. This reflects the industry consensus that the tag is dead for Google.

**The cost of omission is zero.** Removing meta keywords from a site causes no ranking loss. Sites that previously emitted meta keywords and removed them universally see zero negative impact (in some cases minor positive impact from cleaner code).

**Cost of getting it wrong.** Aggressive meta keywords usage produces silent damage. Real examples:

* Client site (legacy WordPress, pre Yoast 2014) had meta keywords with 25-40 keywords per page including competitor brand names. Bing rankings declined over 2 years; correlation with the meta keyword stuffing was strong. Removing the tag stopped the decline.
* Federal subcontractor site published meta keywords revealing target federal contract opportunity codes. Competitors used the information to bid against them on specific contracts. Lost three subcontracting bids over six months before discovering the leak.
* Tax client site had meta keywords listing "CPA", "EA", and "tax preparation" despite the actual credential being PTIN only. This created an SDVOSB protective concern: while Google ignored the keywords, the page source contained credential overstatement that could appear in screenshots or audit materials.
* Site copy paste from a 2005 example included `<meta name="keywords" content="best, top, leading, premier, professional, expert, certified, licensed">`. The "best" and "top" terms are particularly flagged by Bing's quality signals. The page was hit with a quality penalty before discovery.

All preventable with the rule below: **omit `<meta name="keywords">` entirely.**

---

## 3. WHAT THIS COVERS

The meta keywords tag plus the operational context for the omit decision:

1. **The historical and current status**: rise, fall, current state.
2. **Search engine specific behavior**: Google, Bing, Yandex, Baidu, DuckDuckGo.
3. **Legitimate modern uses**: internal site search, regional engines, documentation.
4. **The competitive intelligence concern**: revealing strategy via page source.
5. **Why CMS systems still auto generate it**: WordPress legacy plugins, others.
6. **The Bubbles decision**: omit entirely on all client sites.

This is a shorter framework than the previous HTML signal track entries because the topic is essentially a negative recommendation. The detail is in WHY and HOW to handle legacy cleanup.

---

## 4. THE MENTAL MODEL: IT IS DEAD BUT STILL PARSED

The meta keywords tag occupies an unusual position: it was the most important SEO signal of the 1990s and is functionally irrelevant in 2026, but the tag still exists in the HTML spec, browsers still parse it, and some search tools still read it.

```
Crawler fetches your page, sees <meta name="keywords" content="...">
   |
   v
The tag is parsed and stored in the crawler's understanding of the page.
   |
   v
What happens next depends on the search engine:

==================== PER ENGINE BEHAVIOR ====================

Google:
   - Reads the tag.
   - Stores the value in its index.
   - Does NOT use it for ranking.
   - May use it for anti spam signal detection (excessive keywords flagged).

Bing:
   - Reads the tag.
   - Uses as one of many relevance signals.
   - Excessive or irrelevant keywords trigger anti spam.
   - Conservative use is neutral.

Yandex (Russia):
   - Reads the tag.
   - Uses as a ranking factor.
   - More important than for Google/Bing.

Baidu (China):
   - Reads the tag.
   - Uses as a ranking factor.

DuckDuckGo:
   - Derives from Bing's index.
   - Inherits Bing's behavior.

Internal site search:
   - Often uses meta keywords as a relevance signal.
   - Varies by implementation.

==================== THE PRACTICAL EFFECT ====================

For US/English market sites (the Bubbles client base):
   - Omitting meta keywords: no negative effect.
   - Including conservative meta keywords (3-5 relevant terms): no positive effect.
   - Including aggressive meta keywords (10+ terms, especially irrelevant): negative
     effect on Bing.
```

Six rules govern the decision:

1. **Default: omit the tag entirely.** No ranking benefit on Google (majority of traffic).
2. **If targeting Russia or China specifically: include 3-5 highly relevant keywords.**
3. **For internal site search: maintain a separate keywords source (not exposed via meta tag).**
4. **Never use it as a strategic signal**: assume competitors read your page source.
5. **Legacy CMS sites: audit and remove any auto generated meta keywords.**
6. **Bing webmaster guidelines: <5 highly relevant terms if any.**

A correctly configured Bubbles client site has no `<meta name="keywords">` tag at all.

---

## 5. THE HISTORICAL CONTEXT (1990S RISE, 2009 FALL)

Understanding why meta keywords is dead requires understanding why it was once dominant.

### 5.1 The 1990s Rise

In the early World Wide Web (1993-1998), search engines had primitive ranking algorithms. AltaVista, Lycos, Infoseek, HotBot, and Excite all relied heavily on meta tag content for understanding pages. The meta keywords tag was THE primary signal for "what is this page about?"

The interaction was straightforward:

* Webmaster declares page topics via meta keywords.
* Search engine indexes those keywords.
* When users search for those terms, the page ranks highly.

This worked when webmasters were honest. The signal was useful for several years.

### 5.2 The Stuffing Era (1998-2005)

By the late 1990s, webmasters realized they could:

* Add hundreds of keywords to the tag.
* Include competitor brand names.
* Include popular but unrelated terms ("Britney Spears", "free downloads", etc.).
* Hide additional keywords in invisible text on the page.

Search results degraded. Looking for "personal finance" returned spam sites with meta keywords listing every conceivable financial term. Search engines had to respond.

### 5.3 Google's PageRank Reset (1998-2005)

Google's PageRank algorithm (1998) fundamentally changed search ranking. Instead of relying on what a page CLAIMED to be about (meta keywords), Google measured what other pages SAID the page was about (link anchor text plus topical context). This was much harder to spam.

Through the early 2000s, Google's algorithm increasingly relied on:

* PageRank (link based authority).
* Anchor text (what other sites linked the page as).
* Body content analysis.
* User behavior signals.

Meta keywords steadily decreased in importance through this period.

### 5.4 The 2009 Official Death

In September 2009, Google's Matt Cutts (head of webspam at the time) published a YouTube video and blog post titled "Google does not use the keywords meta tag in web ranking." The official statement:

> "Google does not use the keywords meta tag in our web search ranking. Our web search disregards keyword metatags completely. They simply don't have any effect in our search ranking at present."

This was the public acknowledgment of what had been true for years. The signal had been deprioritized to zero years before; 2009 was when Google made it explicit.

### 5.5 The Post 2009 Era

After 2009:

* WordPress SEO plugins (Yoast, All in One SEO, others) increasingly removed or hid the meta keywords feature.
* Yoast removed it entirely in version 1.5 (2014).
* New CMS systems built post 2009 generally don't include meta keywords by default.
* SEO consultants stopped recommending it.

Yet many older sites still have meta keywords from legacy templates. The cleanup is ongoing.

### 5.6 The Current Status

As of 2026:

* Google: ignored for ranking; potentially considered for anti spam signal.
* Bing: minor signal, mostly anti spam focused.
* Yandex: still used, particularly for Russian language sites.
* Baidu: still used for Chinese language sites.
* DuckDuckGo, Brave Search, Ecosia, Startpage: all derive from Bing or Google; same behavior.
* Modern CMS systems: rarely include by default.
* Old CMS systems: still auto generate from page tags/categories.

---

## 6. CURRENT SEARCH ENGINE BEHAVIOR

Specific per engine behavior in 2026.

### 6.1 Google

**Official statement**: "Google does not use the keywords meta tag in our web search ranking" (Matt Cutts, 2009; reconfirmed multiple times since).

**Anti spam consideration**: Google may use excessive or irrelevant meta keywords as a quality signal. Pages with 20+ keywords or keywords unrelated to page content may be deprioritized for quality reasons (not directly for the meta keyword reason, but for the overall low quality signal).

**Practical recommendation**: omit the tag.

### 6.2 Bing

**Official statement** (Bing Webmaster Guidelines): "Meta keywords can be helpful when used in moderation. Bing uses meta keywords as one of many signals when determining the topical relevance of a page."

**Practical impact**: minor positive when 3-5 highly relevant keywords are present. Zero or slightly negative when missing. Strongly negative when excessive or irrelevant keywords are present.

**The anti spam consideration**: Bing's documentation notes that "stuffing meta keywords with words unrelated to your page can hurt your rankings."

**Practical recommendation**: omit the tag (the upside is minor; the downside potential is real). If included, limit to 3-5 highly relevant terms.

### 6.3 Yandex (Russia)

**Behavior**: Yandex uses meta keywords as a ranking factor. The behavior is closer to early 2000s Google than current Google.

**Practical recommendation for Russian language sites**: include meta keywords with 5-10 highly relevant terms in Russian. Use Cyrillic script. Avoid stuffing.

For Bubbles client sites NOT targeting Russia: omit.

### 6.4 Baidu (China)

**Behavior**: Baidu uses meta keywords for Chinese language site ranking. Similar treatment to Yandex.

**Practical recommendation for Chinese language sites**: include meta keywords with 5-10 highly relevant terms in Simplified Chinese (or Traditional Chinese for Taiwan/Hong Kong targets).

For Bubbles client sites NOT targeting China: omit.

### 6.5 DuckDuckGo, Brave Search, Ecosia, Startpage

**Behavior**: these are search engines that derive their index from Bing (Brave Search has its own crawl but small relative to Bing). They inherit Bing's behavior.

**Practical recommendation**: same as Bing. Omit or limit to 3-5 relevant terms.

### 6.6 Internal Site Search

**Behavior**: internal site search systems (Algolia, ElasticSearch, Solr, custom databases) often use meta keywords as one relevance signal when indexing pages.

**Practical recommendation**: see Section 12. For internal search, maintain a separate keywords source (database, JSON metadata) rather than exposing via meta tag.

### 6.7 The Bubbles Aggregate Decision

For US/English market client sites (the Bubbles base):

* Google: dominant traffic source. Meta keywords ignored.
* Bing: secondary traffic. Conservative meta keywords neutral; aggressive is negative.
* DuckDuckGo etc.: inherit Bing.
* Yandex, Baidu: not relevant for NWA/SWMO local businesses.

**The decision: omit meta keywords entirely.** The tag has no upside for the dominant traffic source, potential downside for the secondary source, no relevance for the tertiary sources.

---

## 7. LEGITIMATE MODERN USES

The narrow cases where meta keywords still has some value.

### 7.1 Targeting Russian Or Chinese Markets

Sites with significant traffic from Russia (Yandex) or China (Baidu) should include meta keywords:

```html
<meta name="keywords" content="разработка веб-сайтов, ручное кодирование, Арканзас">
```

For Russian language sites with Russian users searching Yandex.

```html
<meta name="keywords" content="网站开发, 手工编码, 阿肯色州">
```

For Chinese language sites with Chinese users searching Baidu.

For Bubbles NWA/SWMO clients: not relevant.

### 7.2 Government And Public Sector Sites

Some government and public sector indexing systems (Federal Search, USA.gov, state government search systems) may use meta keywords as a relevance signal. The criteria vary by system.

For Joseph's SDVOSB federal subcontracting work, this could be relevant if any client publishes to a federal indexing system. Worth checking on a per case basis.

### 7.3 Internal Documentation Systems

Confluence, Notion, GitBook, Docusaurus, and other documentation platforms sometimes use meta keywords for internal search relevance. For documentation sites where internal search is more important than external search, the tag can boost the right pages.

For Bubbles client documentation: relevant. For Bubbles client public sites: not relevant.

### 7.4 Niche Industry Search Engines

Some industries have niche search engines or directories that index sites by meta keywords:

* Legal: FindLaw, Justia, Avvo (some still use meta keywords).
* Medical: ClinicalTrials.gov, PubMed (different indexing models).
* Real estate: some MLS systems and aggregators.

For Bubbles real estate clients (Lake Homes Realty Diana Undergust, Local Living Real Estate Laycee Maupin): worth checking what their target MLS aggregators index by. Usually not meta keywords, but verify.

### 7.5 Academic And Library Indexing

Library OPAC systems, academic databases, and institutional repositories often use meta keywords as part of their indexing metadata. Dublin Core meta tag conventions overlap here.

For Bubbles client sites: not typically relevant.

### 7.6 Browser History And Bookmark Search

Some browser implementations use meta keywords as part of their bookmark and history search. Edge cases, low impact.

---

## 8. THE BING ANTI SPAM SIGNAL CONSIDERATION

The most active negative use of meta keywords in 2026 is Bing's anti spam classification.

### 8.1 What Bing Watches For

Bing's documented anti spam considerations for meta keywords:

* **Keyword stuffing**: 10+ keywords, especially repeating root variations ("car, cars, automobile, automobiles, vehicle, vehicles, ...").
* **Irrelevant keywords**: terms that don't match page content.
* **Competitor brand names**: targeting competitor terms.
* **Spam adjacent terms**: "free", "best", "guaranteed", "amazing", "incredible".
* **Numbers and symbols designed for matching**: "buy 1 get 1 free!", "$$$$", etc.

### 8.2 The Quality Threshold

Pages flagged by Bing's anti spam signal experience:

* Reduced ranking on Bing search.
* Reduced visibility in DuckDuckGo (which derives from Bing).
* Potential downstream effects on other Bing index consumers.

### 8.3 The Safe Pattern (If Including At All)

If meta keywords must be included (for Yandex, Baidu, or specific niche reasons):

* 3-5 keywords maximum.
* All keywords directly relevant to page content.
* No competitor brand names.
* No promotional language ("best", "free", "guaranteed").
* No keyword variations of the same term.

### 8.4 The Bubbles Default

Omit entirely. The Bing anti spam consideration is one more reason to skip the tag.

---

## 9. THE COMPETITIVE INTELLIGENCE REALITY

Meta keywords are visible in page source. Anyone can view them.

### 9.1 What Anyone Can See

```bash
# Any visitor can run this on any URL:
curl -s https://example.com/ | grep -oE 'meta name="keywords"[^>]+'

# Or in the browser:
# View Source (Cmd/Ctrl + U) reveals all meta tags.
```

### 9.2 What This Reveals

* The target keywords the page is optimized for.
* The brand strategy (if competitor names are included).
* The geographic targeting.
* The product positioning.

### 9.3 Who Looks

* Direct competitors auditing your strategy.
* SEO tools that aggregate this data (SEMrush, Ahrefs, SimilarWeb, etc.).
* Industry analysts.
* Journalists writing about your business.
* AI scraping tools for competitive intelligence platforms.

### 9.4 The Cost Of Publishing Strategy

Publishing meta keywords reveals:

* Which keywords you think are valuable enough to target.
* Which terms you believe match your audience.
* Your relative emphasis on different positioning aspects.

For a SDVOSB business, this could include revealing:

* Federal contracting code targets.
* Specific industry verticals being pursued.
* Geographic expansion plans.

### 9.5 The Bubbles Implication

Joseph's federal subcontracting work especially benefits from omitting meta keywords. Federal contract targeting strategy is high value competitive intelligence. The information has zero value to Google for ranking; significant value to competitors for targeting.

**Decision: omit.**

---

## 10. WHY SOME SITES STILL EMIT IT

Despite meta keywords being effectively dead, the tag persists across the web. The reasons:

### 10.1 Legacy WordPress Themes And Plugins

Sites built before Yoast's 2014 removal (or with older or alternative plugins) may have meta keywords auto generated from:

* Post tags.
* Categories.
* Custom fields.
* SEO plugin defaults.

### 10.2 CMS Defaults

Some lesser used CMS platforms still include meta keywords as a default field:

* Older Drupal versions.
* Older Joomla configurations.
* Some Wix themes.
* Some Weebly templates.
* Custom CMS systems built before 2010.

### 10.3 Hand Coded Legacy Templates

Sites hand coded in the 2000s often include meta keywords as a "best practice" of that era. Subsequent maintainers may not realize it can be removed.

### 10.4 Habit And Misinformation

Some SEO guides still reference meta keywords as a best practice (despite being outdated). Templates copy pasted from these sources include the tag. Junior SEO consultants sometimes recommend including it because they encountered the practice during training.

### 10.5 "It Can't Hurt" Thinking

Some operators believe "including meta keywords might help and can't hurt." This is incorrect as documented in Section 8: aggressive usage can hurt via Bing anti spam.

### 10.6 The Bubbles Cleanup Pattern

When taking over a client site from another developer or platform:

1. **Audit existing meta keywords** in all templates.
2. **Categorize**: appropriate (3-5 relevant), aggressive (10+ or irrelevant), or absent.
3. **Remove** all aggressive meta keywords.
4. **Optional**: leave appropriate ones if the client expressed preference (rare).
5. **Default**: remove all meta keywords for full Bubbles consistency.

---

## 11. BETTER ALTERNATIVES FOR SIGNALING PAGE FOCUS

The legitimate concern behind wanting meta keywords (signaling page topic to search engines) is better addressed via other signals.

### 11.1 The Page Title

The `<title>` element is the single most important on page SEO signal in 2026. It carries the topic focus that meta keywords used to carry, but it's actually used for ranking.

```html
<title>Hand Coded Web Development | Northwest Arkansas | ThatDeveloperGuy</title>
```

### 11.2 The H1 Heading

The first major heading on a page is the second most important topic signal:

```html
<h1>Custom hand coded websites for Northwest Arkansas businesses</h1>
```

### 11.3 The Meta Description

Covered in framework-html-meta-description.md. The description plays the role of "expanded title" with topic signaling and CTR optimization.

### 11.4 Body Content

Modern algorithms extract topical understanding from page content. The H2 headings, subheading structure, paragraph topic sentences, and overall content quality all signal page topic far more reliably than any meta tag could.

### 11.5 Internal Link Anchor Text

Pages linked TO from other pages on your site with descriptive anchor text receive that anchor text as a topic signal:

```html
<!-- on /about page -->
<a href="/services/">our hand coded web development services</a>
```

The link tells Google that /services/ is about "hand coded web development services" without requiring meta keywords on /services/.

### 11.6 Schema.org Structured Data

The `keywords` property on Schema.org types (Article, BlogPosting, Product, etc.) is a structured equivalent:

```json
{
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": "Why Hand Coded Sites Outperform Platforms",
    "keywords": "hand coded, web development, SEO, Northwest Arkansas, ThatDeveloperGuy"
}
```

Note: this is NOT the same as `<meta name="keywords">`. Schema.org keywords is structured data; Google's rich snippet documentation describes specific properties Google uses. Schema.org keywords is informational metadata that may or may not be used for ranking.

For Bubbles: include schema.org keywords if implementing other schema. Do NOT include meta keywords.

### 11.7 The Bubbles Topic Signal Stack

Per Bubbles convention:

1. **`<title>`**: page focus, 50-60 chars, brand at end.
2. **`<h1>`**: clear page topic, 1 per page.
3. **`<meta name="description">`**: topic + value + CTA, 120-155 chars.
4. **Body content**: clear topic sentences in opening paragraphs.
5. **Internal links**: descriptive anchor text from related pages.
6. **Schema.org**: structured data including keywords property where applicable.

No meta keywords.

---

## 12. THE INTERNAL SITE SEARCH USE CASE

If a Bubbles client site has an internal search feature, meta keywords might seem useful for boosting specific pages in search results. Better alternatives exist.

### 12.1 The Common Pattern (Wrong)

```html
<!-- attempting to boost this page in internal search -->
<meta name="keywords" content="pricing, plans, packages, cost, fee, rate">
```

This exposes the strategy and depends on the internal search system honoring meta keywords (many don't).

### 12.2 The Better Pattern (Right)

Maintain a separate keywords source for internal search:

```python
# /opt/bubbles/services/example.com/search_metadata.py

SEARCH_METADATA = {
    "/pricing/": {
        "keywords": ["pricing", "plans", "packages", "cost", "fee", "rate"],
        "boost": 1.5,
    },
    "/services/": {
        "keywords": ["services", "what we offer", "capabilities"],
        "boost": 1.2,
    },
    # ... per URL
}
```

The internal search system (Algolia, ElasticSearch, or custom Python search) reads from this metadata, applies the boost. Public visitors see no meta keywords in page source.

### 12.3 The Algolia Pattern

Algolia indexing supports custom attributes:

```python
import algoliasearch

client = algoliasearch.Client('YourApplicationID', 'YourAdminAPIKey')
index = client.init_index('your_index_name')

index.save_objects([
    {
        "objectID": "/pricing/",
        "title": "Pricing | ThatDeveloperGuy",
        "description": "Pricing for hand coded websites...",
        "_searchKeywords": ["pricing", "plans", "packages", "cost", "fee", "rate"],
        # _searchKeywords is internal; not exposed via meta tag
    }
])
```

### 12.4 The ElasticSearch Pattern

```json
PUT /bubbles_search/_doc/pricing-page
{
    "url": "/pricing/",
    "title": "Pricing | ThatDeveloperGuy",
    "content": "...",
    "search_keywords": ["pricing", "plans", "packages", "cost", "fee", "rate"]
}
```

Index settings:

```json
PUT /bubbles_search/_mapping
{
    "properties": {
        "search_keywords": {
            "type": "text",
            "boost": 2.0
        }
    }
}
```

### 12.5 The Bubbles Convention

For internal site search:

* Maintain search keywords in a server side data structure (database, JSON file, Python dict).
* The internal search indexer reads from this source.
* Page source has no `<meta name="keywords">` tag.
* Competitive intelligence stays internal.

---

## 13. THE BUBBLES DECISION: OMIT ENTIRELY

The summary recommendation for all Bubbles client sites.

### 13.1 The Rule

**Do not include `<meta name="keywords">` in any Bubbles client site.**

This applies to:

* New site builds.
* Site migrations from WordPress, Squarespace, Wix, or other platforms.
* Site overhauls.
* Federal subcontract sites.
* Local business sites.
* E-commerce sites.
* Documentation sites.

### 13.2 The Migration Cleanup

When migrating a client site that previously had meta keywords:

```bash
# Find all HTML files with meta keywords
find /var/www/sites/example.com/ -name "*.html" -type f | xargs grep -l 'meta name="keywords"'

# Remove meta keywords tags from all HTML files
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    sed -i '/meta name="keywords"/d' {} \;

# Verify removal
find /var/www/sites/example.com/ -name "*.html" -type f | xargs grep -l 'meta name="keywords"'
# Should produce no output
```

### 13.3 The Template Discipline

For FastAPI Jinja2 templates:

```html
{# /opt/bubbles/services/example.com/templates/base.html #}
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>{{ title }}</title>
    <meta name="description" content="{{ description }}">

    {# NO meta keywords #}
    {# Topic signaling via title, h1, description, and schema instead #}

    {% block head_extra %}{% endblock %}
</head>
```

No conditional logic to add meta keywords. The default is no tag.

### 13.4 The Exception Process

The narrow exceptions (Yandex/Baidu targeting, federal indexing systems):

1. **Document the specific requirement** in the client's project documentation.
2. **Include 3-5 highly relevant terms only**.
3. **Avoid all flagged terms** (best, free, guaranteed, etc.).
4. **Avoid competitor names**.
5. **Review quarterly** to confirm the targeting is still valid.

For typical Bubbles work in NWA/SWMO: no exceptions apply.

---

## 14. THE CMS AUTO GENERATION PROBLEM

Many CMS platforms auto generate meta keywords from page tags or categories. This is often the root cause of meta keywords appearing on client sites.

### 14.1 WordPress (Yoast, All in One SEO, Rank Math)

Yoast removed meta keywords entirely in 2014. Modern Yoast does not auto generate.

All in One SEO Pack still has a "Keyword Settings" option (off by default in recent versions).

Rank Math has meta keywords in advanced settings.

WordPress without an SEO plugin: post tags can auto generate meta keywords via the theme.

**Cleanup for WordPress migrations**: check active SEO plugin settings; disable meta keywords feature.

### 14.2 Squarespace

Squarespace has historically auto generated meta keywords from tags. The behavior varies by template version. As of 2026, newer templates do not include meta keywords; older templates do.

**Cleanup for Squarespace migrations**: when migrating to Bubbles hand coded, the meta keywords tag does not carry over (a feature, not a bug).

### 14.3 Wix

Wix uses an Advanced SEO settings panel. Meta keywords can be added but are not auto generated.

**Cleanup for Wix migrations**: check Advanced SEO settings; clear meta keywords field before export.

### 14.4 Webflow

Webflow has a "Custom Code" area where meta tags can be added. Not auto generated.

**Cleanup for Webflow migrations**: review Custom Code; remove meta keywords if present.

### 14.5 Drupal

Older Drupal versions (5, 6, 7) often included meta keywords by default. Drupal 8+ requires explicit module configuration.

**Cleanup for Drupal migrations**: check Metatag module configuration; disable keywords.

### 14.6 Joomla

Older Joomla versions defaulted to including meta keywords. Joomla 4+ has more conservative defaults.

**Cleanup for Joomla migrations**: review article metadata configuration.

### 14.7 The Generic Cleanup Pattern

For any client site coming from another platform:

```bash
# 1. Export the existing site (if possible) and grep for meta keywords
grep -rn 'meta name="keywords"' /tmp/site-export/

# 2. Note the patterns: are they auto generated from tags? Hand crafted? Empty?

# 3. After migrating to Bubbles, ensure no meta keywords in new templates.

# 4. Verify on production
curl -s https://example.com/ | grep -oE 'meta name="keywords"' | wc -l
# Expected: 0
```

---

## 15. ASSET CLASS AND USE CASE RECIPES

Paste ready snippets per scenario. Note: most recipes show what NOT to do (the legacy patterns) and what to do instead (omission).

### 15.1 Canonical Bubbles head (no meta keywords)

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>Page Title | ThatDeveloperGuy</title>
    <meta name="description" content="...">

    <!-- NO meta keywords. Topic signaling via title, h1, description. -->
</head>
```

### 15.2 Yandex targeting (Russian language sites only)

```html
<head>
    <meta charset="utf-8">
    <meta name="description" content="...">
    <meta name="keywords" content="разработка веб-сайтов, ручное кодирование, малый бизнес">
    <!-- 3 highly relevant Russian language keywords for Yandex -->
</head>
```

Not relevant for typical Bubbles work.

### 15.3 Baidu targeting (Chinese language sites only)

```html
<head>
    <meta charset="utf-8">
    <meta name="description" content="...">
    <meta name="keywords" content="网站开发, 手工编码, 小型企业">
    <!-- 3 highly relevant Simplified Chinese keywords for Baidu -->
</head>
```

Not relevant for typical Bubbles work.

### 15.4 Schema.org structured data with keywords property

```html
<head>
    <meta charset="utf-8">
    <meta name="description" content="Why hand coded sites outperform Wix and Squarespace for SEO.">

    <script type="application/ld+json">
    {
        "@context": "https://schema.org",
        "@type": "BlogPosting",
        "headline": "Hand Coded vs Platforms: SEO Comparison",
        "description": "Analysis of how hand coded sites outperform Wix and Squarespace.",
        "keywords": "hand coded websites, SEO comparison, Wix vs custom, web development NWA",
        "author": {
            "@type": "Person",
            "name": "Joseph Anady"
        }
    }
    </script>

    <!-- NO <meta name="keywords"> -->
</head>
```

The Schema.org `keywords` property is structured metadata. Not the same as the deprecated meta tag.

### 15.5 Internal site search with separate keywords source

```html
<!-- The page itself: no meta keywords -->
<head>
    <meta charset="utf-8">
    <title>Pricing | ThatDeveloperGuy</title>
    <meta name="description" content="Pricing for hand coded websites...">
</head>
```

```python
# /opt/bubbles/services/example.com/search_metadata.py
# Keywords for internal search indexer; not exposed via meta tag

INTERNAL_SEARCH_KEYWORDS = {
    "/pricing/": ["pricing", "plans", "packages", "cost", "fee", "rate", "investment"],
    "/services/": ["services", "what we offer", "capabilities", "web development"],
    "/about/": ["about", "Joseph", "founder", "story", "team"],
}
```

### 15.6 Removing meta keywords from a legacy template

```bash
# Find and remove meta keywords from all HTML files
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    sed -i '/meta name="keywords"/d' {} \;

# Apply changes (nginx reload not needed for static files)
# Verify
curl -s https://example.com/ | grep -oE 'meta name="keywords"' | wc -l
# Expected: 0
```

### 15.7 Conditional emission (rare exception case)

```python
# /opt/bubbles/services/example.com/main.py
# Only emit meta keywords for specific routes when Yandex targeting is on

EMIT_KEYWORDS_ROUTES = {
    # No routes by default
}

# For exception cases only:
# EMIT_KEYWORDS_ROUTES = {
#     "/ru/": ["разработка веб-сайтов", "ручное кодирование"],
# }

@app.get("/")
async def homepage(request: Request):
    keywords = EMIT_KEYWORDS_ROUTES.get(str(request.url.path), [])
    return templates.TemplateResponse(
        "page.html",
        {"request": request, "keywords": keywords}
    )
```

```html
{# template #}
<head>
    <meta charset="utf-8">
    {% if keywords %}
        <meta name="keywords" content="{{ keywords|join(', ') }}">
    {% endif %}
</head>
```

### 15.8 Bulk audit across all Bubbles client sites

```bash
#!/bin/bash
# /usr/local/bin/keywords-audit.sh
# Find meta keywords across all client sites

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")

    # Count files with meta keywords
    FILES_WITH=$(find "$site_dir" -name "*.html" -type f -exec grep -l 'meta name="keywords"' {} \; 2>/dev/null | wc -l)

    if [ "$FILES_WITH" -gt 0 ]; then
        echo "$SITE: $FILES_WITH HTML files have meta keywords"

        # Show one example
        EXAMPLE=$(find "$site_dir" -name "*.html" -type f -exec grep -l 'meta name="keywords"' {} \; 2>/dev/null | head -1)
        KEYWORD_VALUE=$(grep 'meta name="keywords"' "$EXAMPLE" | head -1)
        echo "  Example: $EXAMPLE"
        echo "  Value: $KEYWORD_VALUE"
    fi
done
```

---

## 16. BUBBLES STANDARD PATTERN (PASTE READY)

The canonical Bubbles approach: omit meta keywords entirely.

### 16.1 The Template

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <title>{specific page title} | {brand}</title>
    <meta name="description" content="{120-155 character unique description}">

    <!-- NO meta keywords -->
    <!-- Topic signaling: title (above), h1 (in body), description (above), schema (below if applicable) -->

    {% block head_extra %}{% endblock %}
</head>
```

### 16.2 The Migration Audit Script

```bash
#!/bin/bash
# /usr/local/bin/keywords-cleanup-audit.sh
# Check for meta keywords across all sites; flag for removal

REMOVED=0
FOUND=0

for site_dir in /var/www/sites/*/; do
    SITE=$(basename "$site_dir")

    # Find files with meta keywords
    FILES=$(find "$site_dir" -name "*.html" -type f -exec grep -l 'meta name="keywords"' {} \; 2>/dev/null)

    if [ -n "$FILES" ]; then
        FOUND=$((FOUND + 1))
        echo "=== $SITE ==="

        # Show each file
        echo "$FILES" | while read f; do
            VALUE=$(grep 'meta name="keywords"' "$f" | head -1)
            echo "  $f"
            echo "    $VALUE"
        done
    fi
done

echo ""
echo "Sites with meta keywords: $FOUND"

if [ $FOUND -gt 0 ]; then
    echo ""
    echo "To remove, run:"
    echo "  find /var/www/sites/<site_dir>/ -name '*.html' -type f -exec sed -i '/meta name=\"keywords\"/d' {} \;"
fi
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/keywords-cleanup-audit.sh
keywords-cleanup-audit.sh
```

### 16.3 The Code Review Pattern

In code review for new templates or migrations:

* **Reject**: any template that adds `<meta name="keywords">`.
* **Accept**: templates that omit the tag (default).
* **Question**: any exception case; require justification (Yandex/Baidu targeting, etc).

### 16.4 The Client Conversation

When clients ask about meta keywords (because they read older SEO articles or had a previous developer use them):

> "Meta keywords aren't used by Google or major search engines for ranking. We focus on the signals that actually work: clear page titles, descriptive headings, useful meta descriptions, and quality content. This keeps your strategy private and avoids quality penalties on Bing."

This frames the decision positively without dismissing the client's question.

---

## 17. AUDIT CHECKLIST

Smaller checklist than other frameworks because the answer is mostly "ensure tag is absent."

### Absence verification

1. [ ] No `<meta name="keywords">` tag in homepage.
2. [ ] No `<meta name="keywords">` tag in any page template.
3. [ ] No `<meta name="keywords">` tag in any blog post.
4. [ ] No `<meta name="keywords">` tag in any service page.
5. [ ] No `<meta name="keywords">` tag in any product page.
6. [ ] No `<meta name="keywords">` tag in any city/location landing page.

### Migration cleanup

7. [ ] Migration audit run: `keywords-cleanup-audit.sh`.
8. [ ] Old templates that included meta keywords have been updated.
9. [ ] CMS plugins/settings that auto generated meta keywords have been disabled.
10. [ ] Old WordPress/Squarespace exports have been cleaned before deployment.

### Verification across the sitemap

11. [ ] Bulk sitemap audit confirms zero meta keywords across all indexable URLs.
12. [ ] Spot check 10 random URLs from sitemap; all confirm absence.

### Exception cases

13. [ ] If any meta keywords are present (Yandex/Baidu/niche case):
    - [ ] Documented in client project notes.
    - [ ] Limited to 3-5 highly relevant terms.
    - [ ] No competitor brand names.
    - [ ] No promotional language ("best", "free", etc.).
    - [ ] Reviewed quarterly.

### Internal site search

14. [ ] If internal search exists, keywords are stored server side (not via meta tag).
15. [ ] Search index references the internal keywords source.
16. [ ] Public page source has no meta keywords.

### Schema.org structured data

17. [ ] If Schema.org structured data is used, keywords property is in JSON-LD (acceptable).
18. [ ] Schema.org keywords property is concise and relevant.
19. [ ] No `<meta name="keywords">` even when schema keywords are present.

### Code review and process

20. [ ] Template code review rejects any new meta keywords additions.
21. [ ] Migration checklist includes "remove meta keywords" step.
22. [ ] Client education materials explain why meta keywords are omitted.

### Documentation

23. [ ] Bubbles convention "no meta keywords" documented in style guide.
24. [ ] Exception cases documented per client.

A site that passes all 24 has correctly handled the meta keywords decision: omitted by default, with documented exceptions only for narrow legitimate cases.

---

## 18. COMMON PITFALLS

Eleven patterns to recognize and avoid. (Shorter than other frameworks; the topic is narrower.)

**Pitfall 1: Adding meta keywords because "it can't hurt".**
Symptom: tag added by junior developer or following outdated SEO advice.
Why it breaks: provides no benefit on Google; potential Bing anti spam negative.
Fix: omit the tag.

**Pitfall 2: Stuffing meta keywords with 20+ terms.**
Symptom: long list of related terms in the tag.
Why it breaks: triggers Bing anti spam scrutiny; reveals strategy to competitors.
Fix: omit the tag entirely. If retaining for niche reasons, limit to 3-5 terms.

**Pitfall 3: Including competitor brand names in meta keywords.**
Symptom: meta keywords list "Wix", "Squarespace", "Webflow" alongside actual page topics.
Why it breaks: classic black hat SEO pattern; flagged by Bing; doesn't help on Google.
Fix: never include competitor names in meta keywords.

**Pitfall 4: Including promotional terms in meta keywords.**
Symptom: terms like "best", "leading", "premier", "guaranteed", "free", "amazing".
Why it breaks: triggers Bing's promotional language detection; low quality signal.
Fix: factual descriptive terms only (if meta keywords used at all).

**Pitfall 5: Auto generated meta keywords from WordPress legacy plugin.**
Symptom: every post has identical meta keywords because plugin uses all post tags or categories.
Why it breaks: duplicate content signals; reveals tag taxonomy; provides no benefit.
Fix: disable the plugin's meta keywords feature; remove existing tags.

**Pitfall 6: Federal subcontract page revealing target codes via meta keywords.**
Symptom: meta keywords includes NAICS codes, contract type codes, agency abbreviations.
Why it breaks: competitive intelligence leak. Competitors can use this to bid against you.
Fix: omit. Federal targeting strategy stays internal.

**Pitfall 7: Tax client meta keywords overstating credentials.**
Symptom: PTIN preparer page meta keywords include "CPA", "EA", "Enrolled Agent".
Why it breaks: SDVOSB protective concern; credential misrepresentation risk.
Fix: omit. Page content accurately describes credentials.

**Pitfall 8: Meta keywords listed on noindex pages.**
Symptom: login pages, admin pages, thank you pages have meta keywords.
Why it breaks: signals on pages crawlers shouldn't index; pointless work.
Fix: omit. Noindex pages don't benefit from any metadata signals.

**Pitfall 9: Meta keywords in different language than page content.**
Symptom: English page has meta keywords in Spanish or Russian.
Why it breaks: language mismatch; confuses search engine understanding.
Fix: omit. If keywords must be present, match page language.

**Pitfall 10: Schema.org keywords property confused with meta keywords tag.**
Symptom: developer adds both `<meta name="keywords">` and Schema.org keywords property.
Why it breaks: meta keywords offers no benefit; Schema.org keywords is structured data, different mechanism.
Fix: include Schema.org keywords if schema markup is used. Do NOT include meta keywords.

**Pitfall 11: Migrating WordPress site without checking SEO plugin meta keywords settings.**
Symptom: post migration to Bubbles, hand coded templates accidentally include meta keywords from old WordPress data.
Why it breaks: migration didn't strip them.
Fix: run `keywords-cleanup-audit.sh` post migration; remove all meta keywords from new templates.

---

## 19. DIAGNOSTIC COMMANDS

Smaller diagnostic section than other frameworks because the answer is mostly "verify absence."

### Check a single URL

```bash
# Verify meta keywords is absent (expected behavior)
curl -s https://example.com/ | grep -c 'meta name="keywords"'
# Expected: 0

# If present, see the value
curl -s https://example.com/ | grep -oE 'meta name="keywords"[^>]+'
```

### Bulk audit across a site

```bash
# Count meta keywords occurrences across all sitemap URLs
curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | while read url; do
    HAS=$(curl -s "$url" 2>/dev/null | grep -c 'meta name="keywords"')
    if [ "$HAS" != "0" ]; then
        echo "$url: HAS meta keywords"
    fi
done
```

### File system audit

```bash
# Find HTML files containing meta keywords
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    grep -l 'meta name="keywords"' {} \;

# Show the values
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    grep -H 'meta name="keywords"' {} \;
```

### Bulk removal across files

```bash
# Strip meta keywords from all HTML files in a site
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    sed -i '/meta name="keywords"/d' {} \;

# Verify removal
find /var/www/sites/example.com/ -name "*.html" -type f -exec \
    grep -l 'meta name="keywords"' {} \;
# Should produce no output
```

### Check all Bubbles client sites

```bash
# Find any site still emitting meta keywords
for site_dir in /var/www/sites/*/; do
    INDEX="$site_dir/index.html"
    if [ -f "$INDEX" ]; then
        if grep -q 'meta name="keywords"' "$INDEX" 2>/dev/null; then
            echo "STILL HAS META KEYWORDS: $(basename $site_dir)"
        fi
    fi
done
```

### Check WordPress plugin settings (for sites being migrated)

```bash
# For WordPress export XML (or current WP database)
# Check for Yoast meta keywords (should be empty after 2014 Yoast)
grep -A 1 'yoast_wpseo_metakeywords' wordpress-export.xml | grep -v "^--" | head -20

# Check for All in One SEO meta keywords
grep -A 1 'aioseop_keywords' wordpress-export.xml | grep -v "^--" | head -20
```

### After migration cleanup

```bash
# Verify the cleanup
keywords-cleanup-audit.sh

# Expected output:
# Sites with meta keywords: 0
```

---

## 20. CROSS-REFERENCES

* [framework-html-meta-description.md](framework-html-meta-description.md): the meta description is the legitimate replacement for meta keywords' original purpose; topic signaling that actually works for ranking.
* [framework-html-meta-robots.md](framework-html-meta-robots.md): noindex pages should not have any metadata signals including meta keywords.
* [framework-html-meta-charset.md](framework-html-meta-charset.md): if meta keywords is included (rare exception), it must use UTF-8 properly especially for non Latin language keywords.
* [framework-html-meta-viewport.md](framework-html-meta-viewport.md): all four meta tags placed early in head; meta keywords if included goes AFTER charset, viewport, and description.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): X-Robots-Tag HTTP header complements meta tags; the alternative to meta keywords for ranking signals does not exist as an HTTP header.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. Meta keywords ignored by Google means alternative ranking signals (E-E-A-T, content quality, links) are what matter.
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook including the "omit meta keywords" decision.
* Google's 2009 statement (Matt Cutts): https://www.youtube.com/watch?v=jK7IPbnmvVU
* Google's current statement (Search Central documentation): Search Console help article on meta tags Google understands.
* Bing Webmaster Guidelines on meta keywords: https://www.bing.com/webmasters/help/webmaster-guidelines-30fba23a
* Yandex meta keyword documentation (Russian): https://yandex.com/support/webmaster/
* MDN meta name=keywords: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta/name
* schema.org keywords property: https://schema.org/keywords
* The Yoast SEO 2014 removal announcement: https://yoast.com/wordpress/plugins/seo/

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### The Bubbles rule

**Do not include `<meta name="keywords">` in any Bubbles client site.**

### Why

* Google ignored it since 2009.
* Bing uses it minimally; aggressive usage triggers anti spam.
* Reveals strategy to competitors via page source.
* Other ranking signals (title, h1, description, content) do the same job better.

### The narrow exceptions

* Targeting Russia (Yandex): include 3-5 Russian language keywords.
* Targeting China (Baidu): include 3-5 Chinese language keywords.
* Internal documentation systems that index by meta keywords.

For Bubbles NWA/SWMO clients: no exceptions apply.

### Five rules to memorize

1. **Omit meta keywords entirely.**
2. **Topic signaling: title, h1, description, content. Not meta keywords.**
3. **Migration cleanup: strip meta keywords from old templates.**
4. **CMS auto generation: disable the feature.**
5. **Competitive intelligence: don't publish your strategy in page source.**

### Five commands every operator should know

```bash
# 1. Check if a URL has meta keywords (should be 0)
curl -s URL | grep -c 'meta name="keywords"'

# 2. See the value if present
curl -s URL | grep -oE 'meta name="keywords"[^>]+'

# 3. Find files containing meta keywords
find /var/www/sites/example.com/ -name "*.html" -type f -exec grep -l 'meta name="keywords"' {} \;

# 4. Remove meta keywords from all files
find /var/www/sites/example.com/ -name "*.html" -type f -exec sed -i '/meta name="keywords"/d' {} \;

# 5. Audit all Bubbles client sites
keywords-cleanup-audit.sh
```

### Three end to end tests

```bash
# 1. Homepage has no meta keywords
HAS=$(curl -s https://example.com/ | grep -c 'meta name="keywords"')
[ "$HAS" = "0" ] && echo "OK: no meta keywords" || echo "FAIL: meta keywords present"

# 2. No sitemap URLs have meta keywords
PROBLEMS=$(curl -s https://example.com/sitemap.xml | grep -oE "<loc>[^<]+</loc>" | sed 's/<[^>]*>//g' | head -10 | while read url; do
    HAS=$(curl -s "$url" | grep -c 'meta name="keywords"')
    [ "$HAS" != "0" ] && echo "$url"
done | wc -l)
[ "$PROBLEMS" = "0" ] && echo "OK: no sitemap URLs have meta keywords" || echo "FAIL: $PROBLEMS sitemap URLs have meta keywords"

# 3. All files audit
find /var/www/sites/example.com/ -name "*.html" -type f -exec grep -l 'meta name="keywords"' {} \; | wc -l | xargs -I {} bash -c 'if [ {} -eq 0 ]; then echo "OK: file audit clean"; else echo "FAIL: {} files have meta keywords"; fi'
```

If all three pass, the meta keywords layer is correctly handled (which is to say: absent).

---

End of framework-html-meta-keywords.md.
