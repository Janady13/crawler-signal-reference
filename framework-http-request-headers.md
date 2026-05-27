# framework-http-request-headers.md

Comprehensive reference for the eleven HTTP request headers most relevant to server side decisions about how to respond, route, log, throttle, or filter: `User-Agent` (identity, never to be trusted alone), `Accept` (content type negotiation), `Accept-Language` (language negotiation), `Accept-Encoding` (compression negotiation), `Accept-Charset` (legacy charset negotiation), `From` (legacy contact identifier), `Host` (virtual host selection), `If-Modified-Since` (conditional GET by date), `If-None-Match` (conditional GET by ETag), `Referer` (request origin), and `Connection` (transport disposition). Built for Bubbles (Debian, Nginx 1.26+, self hosted origin at 169.155.162.118, no Cloudflare or third party CDN in front). Companion to the seven response side frameworks (caching, content, SEO, security, performance, CORS, rate control), UNIVERSAL-RANKING-FRAMEWORK.md, and SEO BUILD REFERENCE v2.4.

**This is the first request side framework.** The previous seven frameworks documented what the server SENDS in responses. This one documents what crawlers and clients SEND in requests, and how the server reads, validates, and reacts to those headers. The two sides interlock: the server's `Last-Modified` response is meaningful because clients send `If-Modified-Since` requests; the server's `Vary: Accept-Encoding` response only makes sense because clients send `Accept-Encoding` requests.

Audience: humans configuring nginx and FastAPI sidecars to react correctly to incoming headers, AI assistants generating server logic that handles crawler traffic intelligently, SEO operators verifying that high value crawlers are actually who they claim to be, and anyone troubleshooting "Googlebot is being spoofed", "crawlers not getting 304s", "compression not negotiated", or "wrong language served to Spanish speaking crawlers".

---

## TABLE OF CONTENTS

1. Definition
2. Why It Matters
3. What This Covers
4. The Request Header Mental Model (read this first)
5. User-Agent (identity claim, never trust alone)
6. The Crawler Verification Procedure (reverse plus forward DNS, IP ranges)
7. Accept (content type negotiation)
8. Accept-Language (language and locale negotiation)
9. Accept-Encoding (compression negotiation, the gzip plus brotli plus zstd stack)
10. Accept-Charset (legacy charset negotiation)
11. From (legacy contact identifier some bots use)
12. Host (virtual host selection)
13. If-Modified-Since (conditional GET by date)
14. If-None-Match (conditional GET by ETag)
15. Referer (request origin, often absent for crawlers)
16. Connection (transport disposition)
17. How These Headers Interact
18. Asset Class And Use Case Recipes
19. Bubbles Nginx Reference Block (paste ready)
20. Audit Checklist (50+ items)
21. Common Pitfalls
22. Diagnostic Commands
23. Cross-References

---

## 1. DEFINITION

Request headers are everything the client (browser, crawler, API client) attaches to its HTTP request before the body. They communicate identity, capability, preference, and context. The server reads them to decide what to do: which virtual host to route to, what content type to serve, whether to compress, whether the cached version is still valid, whether the client deserves rate limiting or special handling.

The eleven headers in this framework split into five concerns:

* **Identity and verification**: `User-Agent`, `From`. They answer "who claims to be making this request?" but never authoritatively. Verification requires looking beyond the request body.
* **Content negotiation**: `Accept`, `Accept-Language`, `Accept-Encoding`, `Accept-Charset`. They answer "what can this client accept in the response?" The server picks the best match.
* **Routing**: `Host`. It answers "which virtual host is this request for?" Essential for multi tenant nginx serving 181 hostnames from one IP.
* **Conditional GET**: `If-Modified-Since`, `If-None-Match`. They answer "do I already have a fresh copy?" Pair with response side `Last-Modified` and `ETag` from framework-http-caching-headers.md.
* **Context**: `Referer`, `Connection`. They answer "where did this request come from?" and "how should this connection behave after the response?"

A correctly configured server reads these headers, validates them, makes content negotiation decisions, returns 304 Not Modified when appropriate, suppresses or amplifies based on identity (with proper verification), and logs them for security and analytics.

---

## 2. WHY IT MATTERS

Six independent pressures push correct request header handling from "passive" to "actively shaped" infrastructure in 2025 and forward.

**AI crawler traffic is now a measurable percentage of all web traffic.** Cloudflare reported in March 2025 that AI crawlers generated more than 50 billion requests per day across its network. By January 2026, Cloudflare's two month analysis showed Googlebot reached 1.70 times more unique URLs than ClaudeBot, 1.76 times more than GPTBot, 2.99 times more than Meta-ExternalAgent. The fleet of crawlers your server sees is fundamentally different from 2022. Correctly identifying and responding to them is now a first class concern.

**User-Agent spoofing is trivial.** Any scraper, scanner, or attacker can claim to be Googlebot by setting a User-Agent string. Sites that trust the User-Agent without verification get tricked into bypassing rate limits, serving privileged content, or skipping bot management. The verification procedure (reverse DNS plus forward DNS, or IP range matching against published lists) is the actual identity check.

**Conditional GET saves crawl budget and bandwidth.** When a crawler sends `If-Modified-Since` or `If-None-Match` and the content has not changed, the server should return `304 Not Modified` with no body. Google specifically requests this behavior for AdsBot crawls. Every 304 saved is real bandwidth and crawler goodwill. A site that ignores conditional GET headers and always returns 200 with full body is wasting both server resources and crawler time.

**Modern Accept-Encoding has three formats.** In 2026, browsers and most clients negotiate gzip, brotli, and zstd. Servers that handle only gzip leave 20 to 30 percent compression savings on the table for modern browsers. Servers that get the negotiation wrong serve broken or oversized content.

**Accept-Language matters for international SEO.** Bingbot and Baidu reportedly use Accept-Language when crawling to discover language variants. Even when not used by crawlers, browser visitors send it and use it to pick the right localized version of a multilingual site.

**Host header attacks are real.** Sending a request to one IP with a Host header for a different domain (Host header injection) can confuse virtual host routing, password reset emails, and cache poisoning. nginx servers that have no `default_server` block and accept any Host header are vulnerable.

**Cost of getting it wrong.** Misconfigured request handling produces silent SEO damage, security incidents, and crawler frustration. Real examples:

* Site trusts User-Agent for "Allow Googlebot to bypass paywall" feature. Attacker spoofs Googlebot UA, reads all paywalled content. Discovery years later in security audit.
* Crawler sends If-Modified-Since on every revisit. Server ignores it and always returns 200 with full HTML. Googlebot wastes crawl budget on unchanged pages; new pages crawl slower.
* Site only supports gzip in Accept-Encoding. Modern browsers send `gzip, deflate, br, zstd` and get gzipped responses 20 percent larger than necessary. LCP regression no one investigates.
* nginx has no `default_server` and a wildcard SSL certificate. Attacker sends request with Host header `evil.com`, gets routed to whichever server block parses first. Cache key now includes a contaminated Host value. Cache poisoning at scale.
* Site uses Referer for auth ("only allow from our own pages"). Browsers stop sending Referer due to privacy settings. Auth fails for users who installed privacy extensions or use specific browsers. Support tickets flood in.

All preventable with the rules below.

---

## 3. WHAT THIS COVERS

Each of the eleven headers gets the same six part treatment:

1. **What it does**: the canonical RFC definition plus practical implication.
2. **Syntax and values**: every common format, what it means, when it is wrong.
3. **How to access it in nginx and FastAPI**: variable names, request object attributes, with concrete examples.
4. **How to use it for server side decisions**: routing, content negotiation, logging, identity gates.
5. **How to verify and troubleshoot**: curl commands and diagnostic patterns.
6. **Crawler specific behavior**: what major search and AI crawlers actually send.

The User-Agent header gets the largest treatment because verification is non trivial. Sections 5 and 6 are paired: identity claim plus identity verification.

---

## 4. THE REQUEST HEADER MENTAL MODEL (READ THIS FIRST)

A request arrives. The server has milliseconds to make several decisions before responding. Each decision is driven by request headers. Internalize the flow.

```
TCP connection accepted by nginx
        |
        v
==================== ROUTING ====================
        |
        v
Read Host header
   * Match against server_name in each server block
   * If no match and no default_server: nginx returns 444 (close connection)
        |
        v
Routed to correct server block
        |
        v
==================== IDENTITY ====================
        |
        v
Read User-Agent header
   * Is this a known crawler? (UA pattern match)
   * Is it actually that crawler? (reverse DNS check, optional)
   * Is it on the rate limit whitelist? (from framework-http-rate-control-headers.md)
        |
        v
==================== CONTENT NEGOTIATION ====================
        |
        v
Read Accept (which content types client wants)
   * Pick best match for this resource
   * Set Vary: Accept on response if we vary

Read Accept-Language (which languages client prefers)
   * Pick best match for multilingual content
   * Set Vary: Accept-Language on response if we vary

Read Accept-Encoding (which compression algorithms client supports)
   * gzip, br, zstd: pick best supported by both client and server
   * Set Vary: Accept-Encoding on response (mandatory when compressing)

Read Accept-Charset (legacy, almost universally ignored in 2026)
   * Just use UTF-8 and move on
        |
        v
==================== CONDITIONAL GET ====================
        |
        v
Read If-Modified-Since (RFC date)
   * Compare to resource's Last-Modified
   * If unchanged: return 304 Not Modified, no body

Read If-None-Match (ETag value)
   * Compare to resource's current ETag
   * If matches: return 304 Not Modified, no body
   * If both If-* present: ETag takes precedence
        |
        v
==================== CONTEXT FOR LOGGING ====================
        |
        v
Read Referer (where the request originated)
   * Log for analytics (referral tracking)
   * Be aware: often empty for crawlers, often empty for privacy aware browsers

Read From (legacy contact field, used by some bots)
   * Log for crawler accountability
   * Rarely actionable
        |
        v
==================== TRANSPORT DISPOSITION ====================
        |
        v
Read Connection (close, keep-alive, upgrade)
   * keep-alive: subsequent requests can reuse the connection
   * close: close after this response
   * upgrade: switch protocol (WebSocket negotiation)
        |
        v
==================== APPLICATION ====================
        |
        v
Process the request, build the response with appropriate Vary headers
```

Five rules govern the system:

1. **Never trust identity claims alone.** User-Agent, From, and Host can all be set to anything. Verify before granting privileges.
2. **Always respond to conditional GET headers.** Return 304 when appropriate. It is free server side bandwidth.
3. **Always negotiate Accept-Encoding correctly.** Modern clients want brotli or zstd; gzip is the universal fallback.
4. **Set Vary on every response that varies on a request header.** Without Vary, caches serve the wrong variant to subsequent visitors.
5. **Always have a default_server.** nginx routes unmatched Host headers to default_server. Without one, it falls back to the first server block (potentially the wrong one).

A correctly configured server reads identity carefully, negotiates content faithfully, returns 304 when warranted, and logs context for analytics. Every header is either a decision point or a logging signal.

---

## 5. USER-AGENT (IDENTITY CLAIM, NEVER TRUST ALONE)

### 5.1 What It Does

`User-Agent` is the client's claim about its own identity, typically including the software name, version, and operating system. Defined in RFC 9110. Trivially spoofable. Used everywhere despite being unverifiable on its own.

```
User-Agent: Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
User-Agent: Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko); compatible; ClaudeBot/1.0; +claudebot@anthropic.com
User-Agent: GPTBot/1.0 (+https://openai.com/gptbot)
```

The User-Agent string is the **identity claim**. The identity verification is separate (Section 6).

### 5.2 The 2026 Crawler Taxonomy

The list of crawlers your Bubbles sites will see, with their current 2026 user agent patterns:

**Search engine crawlers (the historical mainstays):**

| Crawler | User Agent Pattern | Purpose |
|---|---|---|
| Googlebot | `Googlebot/2.1` | Primary Google search index |
| Googlebot-Image | `Googlebot-Image/1.0` | Google Images |
| Googlebot-News | `Googlebot-News` | Google News |
| Googlebot-Video | `Googlebot-Video/1.0` | Google Videos |
| AdsBot-Google | `AdsBot-Google` | Google Ads landing page quality |
| AdsBot-Google-Mobile | `AdsBot-Google-Mobile` | Google Ads mobile landing pages |
| Bingbot | `bingbot/2.0` | Bing search + ChatGPT search backend |
| BingPreview | `BingPreview/1.0b` | Bing snapshot generation |
| DuckDuckBot | `DuckDuckBot/1.1` | DuckDuckGo |
| YandexBot | `YandexBot/3.0` | Yandex (Russian search) |
| Baiduspider | `Baiduspider` | Baidu (Chinese search) |
| Applebot | `Applebot/0.1` | Apple Spotlight and Siri |
| AmazonBot | `Amazonbot/0.1` | Amazon products and Alexa |

**AI training crawlers (data harvesting for LLM training):**

| Crawler | User Agent Pattern | Operator |
|---|---|---|
| GPTBot | `GPTBot/1.0` | OpenAI (training only) |
| ClaudeBot | `ClaudeBot/1.0` | Anthropic (training only; replaces deprecated Claude-Web) |
| Google-Extended | (used in robots.txt only) | Google (Gemini training opt out) |
| CCBot | `CCBot/2.0` | Common Crawl (open dataset used by many LLMs) |
| meta-externalagent | `meta-externalagent/1.1` | Meta (Llama training) |
| Bytespider | `Bytespider` | ByteDance (TikTok parent) |

**AI search and retrieval crawlers (per query fetches for answering users):**

| Crawler | User Agent Pattern | Operator |
|---|---|---|
| OAI-SearchBot | `OAI-SearchBot/1.0` | OpenAI ChatGPT search |
| ChatGPT-User | `ChatGPT-User/1.0` | OpenAI user initiated fetches |
| Claude-SearchBot | `Claude-SearchBot/1.0` | Anthropic search |
| Claude-User | `Claude-User/1.0` | Anthropic user initiated fetches |
| PerplexityBot | `PerplexityBot/1.0` | Perplexity index |
| Perplexity-User | `Perplexity-User/1.0` | Perplexity user fetches |
| YouBot | `YouBot/0.1` | You.com |
| DuckAssistBot | `DuckAssistBot/1.0` | DuckDuckGo Assist |
| Mistral-User | `Mistral-User/1.0` | Mistral user fetches |

**Social and link preview crawlers:**

| Crawler | User Agent Pattern | Operator |
|---|---|---|
| facebookexternalhit | `facebookexternalhit/1.1` | Facebook link previews |
| Twitterbot | `Twitterbot/1.0` | Twitter/X card previews |
| LinkedInBot | `LinkedInBot/1.0` | LinkedIn preview |
| Slackbot | `Slackbot-LinkExpanding 1.0` | Slack link previews |
| Discordbot | `Discordbot/2.0` | Discord embeds |

**Important taxonomy notes for 2026:**

1. **ClaudeBot replaced Claude-Web.** Sites still using `User-agent: Claude-Web` in robots.txt are not actually blocking Anthropic's current crawler. Update to `ClaudeBot`.

2. **OpenAI splits training from search.** `GPTBot` is training only; `OAI-SearchBot` is for ChatGPT search; `ChatGPT-User` is for user initiated fetches. A site can block GPTBot to protect its content from training while allowing OAI-SearchBot to remain visible in ChatGPT search results.

3. **Google-Extended is a robots.txt only token.** It is used in robots.txt to opt out of Google's generative AI training while still allowing standard Googlebot search indexing. It does not appear in actual User-Agent strings.

4. **The browser like AI fetches (Claude-User, Perplexity-User, ChatGPT-User) are user initiated.** Each represents a real person asking the AI assistant to fetch a URL. They are not crawl traffic in the traditional sense; they are pull through views. Allow these for GEO (generative engine optimization) visibility.

### 5.3 How To Access User-Agent

**In nginx:**

```nginx
# Variable: $http_user_agent
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent"';

# Use in map for routing decisions
map $http_user_agent $is_crawler {
    default 0;
    ~*googlebot          1;
    ~*bingbot            1;
    ~*claudebot          1;
    ~*gptbot             1;
    ~*oai-searchbot      1;
    ~*perplexitybot      1;
}
```

**In FastAPI:**

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/")
async def index(request: Request):
    user_agent = request.headers.get("user-agent", "")
    # Use it for content decisions, logging, etc
```

### 5.4 Using User-Agent For Server Side Decisions (With Caveats)

**Acceptable uses (UA alone is sufficient):**

* Logging and analytics. Track which crawlers visit which pages.
* Bot detection signals for analytics dashboards.
* Permissive routing where being wrong is harmless (rate limit whitelist for a public page).
* Content adaptation hints (mobile vs desktop layout, though not the recommended approach).

**Unacceptable uses (UA alone is insufficient, must verify):**

* Paywall bypass for Googlebot. Verify by reverse DNS first.
* Premium content access for known crawlers. Same.
* Skipping security checks (CAPTCHA, fraud detection). Always verify.
* Any privilege grant where being wrong has cost.

The reverse DNS verification procedure is Section 6.

### 5.5 Common UA Anti-Patterns

**Cloaking (illegal in Google's eyes):** Serving different content to Googlebot than to users. Google penalizes sites that do this when detected. The exception is well disclosed dynamic rendering for JavaScript heavy sites.

**Browser sniffing:** Detecting "Internet Explorer" or "Edge" and serving different code. Modern best practice is feature detection, not UA detection.

**Mobile vs desktop UA routing:** Historically common, increasingly discouraged. Use responsive design instead. Google's mobile first indexing prefers responsive sites.

### 5.6 How To Verify The UA Was Set

```bash
# 1. Check what UA your test client sends
curl -sI -A "test-bot/1.0" https://example.com/ -v 2>&1 | grep -i "User-Agent"
# Expected (in request side): User-Agent: test-bot/1.0

# 2. Verify nginx logs the UA correctly
curl -sI -A "TestBot/1.0 Research" https://example.com/
sudo tail -1 /var/log/nginx/access.log
# Look for: "TestBot/1.0 Research"

# 3. Simulate major crawlers for testing routing logic
for ua in "Googlebot/2.1" "bingbot/2.0" "ClaudeBot/1.0" "GPTBot/1.0" "OAI-SearchBot/1.0" "PerplexityBot/1.0"; do
    echo "=== $ua ==="
    curl -sI -A "Mozilla/5.0 (compatible; $ua)" https://example.com/ | head -1
done
```

### 5.7 Troubleshooting

**Symptom: log shows requests from unfamiliar UA strings.**
Likely scenarios:
1. A new crawler launched. Check against the 2026 taxonomy (Section 5.2).
2. A scraper or scanner spoofing a known crawler. Verify by reverse DNS (Section 6).
3. A monitoring tool from a customer or operator. Check if Bubbles or client tools are visiting.
4. An attacker probing for vulnerabilities. Check if the requests have malicious paths.

**Symptom: a UA pattern matches but verification fails.**
The client is spoofing the UA. Treat as unverified traffic; apply standard rate limits and bot management.

**Symptom: Googlebot stopped sending traditional crawl UA, sending different format.**
Google updates their UA strings periodically. Check the official documentation at `https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers` for current strings.

**Symptom: Cloaking penalty from Google.**
Detected serving different content to Googlebot than to users. Audit any UA based content variation. Move to consistent content delivery; use Hreflang for language variation, structured data for content disambiguation.

### 5.8 The Bubbles UA Logging Pattern

For Joseph's 130+ client sites, the goal is to track AI crawler visibility (which is part of the engine optimization investment). Add a dedicated log field for AI crawlers:

```nginx
map $http_user_agent $crawler_type {
    default                  "other";
    ~*googlebot              "search-google";
    ~*bingbot                "search-bing";
    ~*duckduckbot            "search-ddg";
    ~*applebot               "search-apple";
    ~*claudebot              "ai-train-claude";
    ~*gptbot                 "ai-train-openai";
    ~*google-extended        "ai-train-google";
    ~*ccbot                  "ai-train-cc";
    ~*meta-externalagent     "ai-train-meta";
    ~*oai-searchbot          "ai-search-openai";
    ~*chatgpt-user           "ai-user-openai";
    ~*claude-searchbot       "ai-search-claude";
    ~*claude-user            "ai-user-claude";
    ~*perplexitybot          "ai-search-perplexity";
    ~*perplexity-user        "ai-user-perplexity";
    ~*youbot                 "ai-search-you";
    ~*facebookexternalhit    "social-fb";
    ~*twitterbot             "social-twitter";
    ~*linkedinbot            "social-linkedin";
    ~*slackbot               "social-slack";
    ~*discordbot             "social-discord";
}

log_format with_crawler '$remote_addr [$time_local] "$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" crawler=$crawler_type';

access_log /var/log/nginx/access.log with_crawler;
```

Now `awk` can aggregate crawler types per day:

```bash
awk '/crawler=ai-/ {print $NF}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

This is the foundation for the Bubbles AI visibility analytics dashboard.

---

## 6. THE CRAWLER VERIFICATION PROCEDURE (REVERSE PLUS FORWARD DNS, IP RANGES)

User-Agent is a claim. This section is how to verify the claim. The procedure differs by operator: some publish IP ranges; some require reverse DNS.

### 6.1 The Two Verification Methods

**Method A: Reverse DNS plus forward DNS lookup.**

1. Take the IP from the request.
2. Do a PTR lookup: what hostname does this IP resolve to?
3. Check the hostname against an allow list (e.g., `*.googlebot.com`, `*.search.msn.com`).
4. Do a forward A or AAAA lookup on that hostname: what IP does it resolve to?
5. Confirm the forward lookup IP matches the original request IP.

If all four match and the hostname is in the operator's domain, the request is verified.

**Method B: IP range matching against published lists.**

Most major operators publish their IP ranges as JSON files. Match the request IP against the published list. If it falls within an operator's range, it is verified.

| Operator | IP range URL | Notes |
|---|---|---|
| Google (Googlebot, AdsBot, Crawler) | `https://developers.google.com/static/search/apis/ipranges/googlebot.json` | Updated periodically |
| Google (special crawlers) | `https://developers.google.com/static/search/apis/ipranges/special-crawlers.json` | AdsBot, Vertex AI |
| Google (user triggered fetchers) | `https://developers.google.com/static/search/apis/ipranges/user-triggered-fetchers.json` | Google-Read-Aloud, etc |
| Bing | `https://www.bing.com/toolbox/bingbot.json` | All Bing crawlers |
| OpenAI (GPTBot, OAI-SearchBot, ChatGPT-User) | `https://openai.com/gptbot.json`, `https://openai.com/searchbot.json`, `https://openai.com/chatgpt-user.json` | Separate files per crawler |
| Perplexity (PerplexityBot, Perplexity-User) | `https://www.perplexity.ai/perplexitybot.json`, `https://www.perplexity.ai/perplexity-user.json` | Separate files |
| Common Crawl (CCBot) | (uses AWS ranges, not published cleanly) | Verify via UA only, IP varies |
| Apple (Applebot) | (use reverse DNS, `*.applebot.apple.com`) | No published range |
| Amazon (Amazonbot) | (use reverse DNS, `*.amazonbot.amazon`) | No published range |
| **Anthropic (ClaudeBot, Claude-SearchBot, Claude-User)** | **Not published** | **Reverse DNS only; check for hostnames in claude.ai or anthropic.com domains** |

The notable gap: Anthropic does not publish IP ranges as of late 2025. Verification of ClaudeBot must use reverse DNS or rely on UA pattern matching combined with reasonable IP heuristics (data center ASNs).

### 6.2 The Reverse Plus Forward DNS Pattern (Manual)

```bash
# Step 1: get the IP from the request log
REQUEST_IP="66.249.66.1"

# Step 2: reverse DNS lookup (PTR)
PTR=$(host "$REQUEST_IP" | awk '{print $NF}' | sed 's/\.$//')
echo "Reverse: $REQUEST_IP -> $PTR"
# Expected for Googlebot: crawl-66-249-66-1.googlebot.com

# Step 3: check the hostname against expected domain
case "$PTR" in
    *.googlebot.com|*.google.com|*.googleusercontent.com)
        echo "Claims to be Google. Verifying forward DNS..."
        ;;
    *)
        echo "FAIL: hostname does not match Google patterns"
        exit 1
        ;;
esac

# Step 4: forward DNS lookup
FORWARD_IP=$(host "$PTR" | awk '/has address/ {print $NF}' | head -1)
echo "Forward: $PTR -> $FORWARD_IP"

# Step 5: confirm forward IP matches original
if [ "$FORWARD_IP" = "$REQUEST_IP" ]; then
    echo "VERIFIED: $REQUEST_IP is genuine Googlebot"
else
    echo "FAIL: forward IP $FORWARD_IP does not match original $REQUEST_IP"
fi
```

### 6.3 The Verification Pattern As A FastAPI Helper

```python
import socket
import asyncio
from functools import lru_cache

# Hostnames that indicate verified crawler ownership
VERIFIED_HOSTS = {
    "googlebot": ["googlebot.com", "google.com", "googleusercontent.com"],
    "bingbot": ["search.msn.com"],
    "applebot": ["applebot.apple.com"],
    "amazonbot": ["amazonbot.amazon"],
    "yandexbot": ["yandex.com", "yandex.net", "yandex.ru"],
}

@lru_cache(maxsize=10000)
def verify_crawler_by_dns(ip: str, claimed_operator: str) -> bool:
    """Verify a crawler IP using reverse plus forward DNS."""
    expected_suffixes = VERIFIED_HOSTS.get(claimed_operator, [])
    if not expected_suffixes:
        return False

    try:
        # Reverse DNS (PTR)
        hostname, _, _ = socket.gethostbyaddr(ip)
        hostname = hostname.lower().rstrip(".")

        # Check hostname matches expected suffix
        if not any(hostname.endswith(suffix) for suffix in expected_suffixes):
            return False

        # Forward DNS check
        try:
            forward_ips = socket.gethostbyname_ex(hostname)[2]
        except socket.gaierror:
            return False

        return ip in forward_ips
    except (socket.herror, socket.gaierror):
        return False


# Usage in middleware
@app.middleware("http")
async def verify_crawler(request, call_next):
    ua = request.headers.get("user-agent", "").lower()
    client_ip = request.client.host

    claimed_operator = None
    if "googlebot" in ua:
        claimed_operator = "googlebot"
    elif "bingbot" in ua:
        claimed_operator = "bingbot"
    elif "applebot" in ua:
        claimed_operator = "applebot"

    if claimed_operator:
        verified = await asyncio.get_event_loop().run_in_executor(
            None, verify_crawler_by_dns, client_ip, claimed_operator
        )
        request.state.crawler_verified = verified
        request.state.crawler_operator = claimed_operator
    else:
        request.state.crawler_verified = False
        request.state.crawler_operator = None

    return await call_next(request)
```

### 6.4 The IP Range Matching Pattern (For Operators Who Publish)

```python
import asyncio
import json
import ipaddress
import time
import httpx

# Cache of IP ranges, refreshed periodically
ip_range_cache = {}

async def refresh_googlebot_ranges():
    """Refresh Google's published IP ranges."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://developers.google.com/static/search/apis/ipranges/googlebot.json"
        )
        data = response.json()

    networks = []
    for entry in data.get("prefixes", []):
        if "ipv4Prefix" in entry:
            networks.append(ipaddress.ip_network(entry["ipv4Prefix"]))
        if "ipv6Prefix" in entry:
            networks.append(ipaddress.ip_network(entry["ipv6Prefix"]))

    ip_range_cache["googlebot"] = {
        "networks": networks,
        "fetched_at": time.time(),
    }

def is_in_googlebot_range(ip: str) -> bool:
    """Check if an IP is in Google's published Googlebot range."""
    cache = ip_range_cache.get("googlebot")
    if not cache:
        return False

    addr = ipaddress.ip_address(ip)
    return any(addr in network for network in cache["networks"])

# Refresh on startup and every 6 hours
@app.on_event("startup")
async def startup():
    await refresh_googlebot_ranges()

    async def refresher():
        while True:
            await asyncio.sleep(6 * 3600)
            try:
                await refresh_googlebot_ranges()
            except Exception:
                pass

    asyncio.create_task(refresher())
```

### 6.5 The Bubbles Practical Approach

For most Bubbles client sites, full DNS verification on every request is unnecessary overhead. The practical pattern:

* **Logging level (no privilege grant)**: UA pattern matching is sufficient. Tag crawler type in logs and move on.
* **Rate limit whitelist**: UA pattern matching is sufficient. The cost of false positives (a scraper bypassing rate limits) is small for public marketing pages.
* **Sensitive endpoint privilege grant**: full verification required. Examples: paywall bypass, admin access, "give Googlebot a peek behind login".

Document the threat model explicitly: for marketing pages, UA matching is fine; for premium content, verification is mandatory.

### 6.6 How To Test Verification

```bash
# Test with a real Googlebot IP (use one from logs)
GOOGLEBOT_IP="66.249.66.1"  # known Googlebot range
host "$GOOGLEBOT_IP"
# Expected: 1.66.249.66.in-addr.arpa domain name pointer crawl-66-249-66-1.googlebot.com.

host crawl-66-249-66-1.googlebot.com
# Expected: crawl-66-249-66-1.googlebot.com has address 66.249.66.1

# Test with a spoofed UA from a non Google IP
# (Use your own IP for testing)
MY_IP=$(curl -s ifconfig.me)
host "$MY_IP"
# Should NOT show googlebot.com hostname

# Bash one liner to verify
verify_googlebot() {
    local ip=$1
    local ptr=$(host "$ip" 2>/dev/null | awk '/pointer/ {print $NF}' | sed 's/\.$//')
    case "$ptr" in
        *.googlebot.com|*.google.com)
            local forward=$(host "$ptr" 2>/dev/null | awk '/has address/ {print $NF}' | head -1)
            if [ "$forward" = "$ip" ]; then
                echo "VERIFIED: $ip is Googlebot ($ptr)"
                return 0
            fi
            ;;
    esac
    echo "NOT VERIFIED: $ip (PTR: ${ptr:-none})"
    return 1
}

verify_googlebot 66.249.66.1
```

---

## 7. ACCEPT (CONTENT TYPE NEGOTIATION)

### 7.1 What It Does

`Accept` tells the server which content types the client can process. Defined in RFC 9110. The server uses it to pick the best representation when multiple are available.

```
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept: application/json
Accept: */*
```

The format is a comma separated list of media types, each optionally followed by a quality factor (`q=`). Higher q is more preferred. Absent q means q=1.0.

### 7.2 Typical Values

| Client | Common Accept value |
|---|---|
| Modern browser navigating to a page | `text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8` |
| Browser fetching JSON API via fetch | `application/json,text/plain,*/*` |
| curl by default | `*/*` |
| Googlebot rendering page | `text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8` |
| Image fetcher | `image/avif,image/webp,image/png,image/svg+xml,image/*,*/*;q=0.8` |

### 7.3 How To Access In Nginx

```nginx
# Variable: $http_accept
map $http_accept $client_wants_json {
    default 0;
    ~*application/json 1;
}

map $http_accept $client_wants_webp {
    default 0;
    ~*image/webp 1;
}
```

### 7.4 How To Access In FastAPI

```python
@app.get("/data")
async def data(request: Request):
    accept = request.headers.get("accept", "")
    if "application/json" in accept:
        return JSONResponse({"data": "..."})
    else:
        return HTMLResponse("<html>...</html>")
```

FastAPI's automatic content negotiation handles common cases; for complex negotiation use the `accept-types` library:

```python
from accept_types import get_best_match

@app.get("/data")
async def data(request: Request):
    accept = request.headers.get("accept", "*/*")
    best = get_best_match(accept, ["application/json", "text/html", "application/xml"])
    if best == "application/json":
        return JSONResponse({...})
    elif best == "application/xml":
        return Response(content="<xml>...</xml>", media_type="application/xml")
    else:
        return HTMLResponse("<html>...</html>")
```

### 7.5 The Vary: Accept Requirement

If the response varies based on Accept (different representation for JSON vs HTML), set `Vary: Accept`:

```nginx
add_header Vary "Accept" always;
```

Without this, caches may serve the JSON variant to a browser asking for HTML, or vice versa.

### 7.6 The Modern Image Format Negotiation Pattern

For sites serving AVIF or WebP to capable browsers and falling back to JPEG/PNG for others:

```nginx
map $http_accept $image_format {
    default                  "jpg";
    ~*image/avif             "avif";
    ~*image/webp             "webp";
}

location ~* /images/(.+)\.(jpg|png)$ {
    set $base $1;
    set $ext $2;

    # Try the modern format first based on Accept, fall back to original
    try_files /images/$base.$image_format /images/$base.$ext =404;

    add_header Vary "Accept" always;
}
```

When a browser supports AVIF and requests `/images/hero.jpg`, this serves `/images/hero.avif` if it exists, falling back to `/images/hero.jpg`.

### 7.7 Troubleshooting

**Symptom: cache serves JSON to a browser navigation request.**
The endpoint varies on Accept but `Vary: Accept` is missing. Add it.

**Symptom: crawler always gets HTML even when requesting JSON.**
Verify the crawler is actually sending `Accept: application/json`. Some crawlers send `*/*` and rely on URL extension to distinguish.

**Symptom: AVIF served to a browser that does not support it, image broken.**
The browser is sending an outdated or compatibility Accept header. Or the AVIF file is corrupt. Check both.

---

## 8. ACCEPT-LANGUAGE (LANGUAGE AND LOCALE NEGOTIATION)

### 8.1 What It Does

`Accept-Language` tells the server which natural languages the client prefers. Defined in RFC 9110. Uses BCP 47 language tags. Used by browsers to pick the right localized version and by some crawlers (Bingbot, Baiduspider) for language signal.

```
Accept-Language: en-US,en;q=0.9
Accept-Language: es-MX,es;q=0.9,en;q=0.8
Accept-Language: fr-FR,fr;q=0.9,en;q=0.5
```

### 8.2 Format

Comma separated list of language tags with optional quality factors:

* `en-US`: US English
* `en`: any English
* `es-MX`: Mexican Spanish
* `q=0.9`: 90% preference (lower than absent which is 1.0)

The client typically lists their browser language first at high q, then fallback languages at lower q.

### 8.3 How To Access In Nginx

```nginx
# Variable: $http_accept_language

# Simple language detection
map $http_accept_language $preferred_language {
    default                  "en";
    ~*^es                    "es";
    ~*^fr                    "fr";
    ~*^de                    "de";
    ~*^zh                    "zh";
}

# Redirect to language specific path
location = / {
    return 302 /$preferred_language/;
}
```

### 8.4 How To Access In FastAPI

```python
@app.get("/")
async def index(request: Request):
    accept_lang = request.headers.get("accept-language", "")
    preferred = parse_preferred_language(accept_lang)
    return RedirectResponse(f"/{preferred}/")

def parse_preferred_language(header: str) -> str:
    """Parse Accept-Language and return the top language code."""
    if not header:
        return "en"
    # Get first entry, strip quality
    first = header.split(",")[0].split(";")[0].strip().lower()
    # Get language part (en, es, fr)
    return first.split("-")[0] if "-" in first else first
```

### 8.5 The Vary: Accept-Language Requirement

If the response varies based on Accept-Language, set `Vary: Accept-Language`:

```nginx
add_header Vary "Accept-Language" always;
```

### 8.6 Why Hreflang Is Preferred Over Accept-Language Routing

Google's official recommendation for multilingual sites is `hreflang` (Link header or HTML element), not Accept-Language based redirects. Reasons:

* Googlebot does not send Accept-Language matching a target locale.
* Accept-Language based redirects can confuse the user (clicking a French link gets you redirected to Spanish based on browser settings).
* Hreflang provides explicit URLs Google can crawl independently.

The Bubbles convention: use Accept-Language to suggest a language switcher in the UI but do not auto redirect. Use hreflang annotations in HTML and Link header for crawlers.

### 8.7 Troubleshooting

**Symptom: User typed example.com/page but got redirected to example.com/es/page they did not want.**
Auto redirect based on Accept-Language. Bad UX. Remove and use hreflang plus a language switcher UI instead.

**Symptom: Googlebot only crawls English pages.**
Hreflang is not set up. Add hreflang annotations. Googlebot will discover and crawl other language variants.

**Symptom: Bingbot crawls Spanish pages from a Spanish IP.**
Bingbot does send Accept-Language sometimes. This is correct behavior.

---

## 9. ACCEPT-ENCODING (COMPRESSION NEGOTIATION, THE GZIP PLUS BROTLI PLUS ZSTD STACK)

### 9.1 What It Does

`Accept-Encoding` tells the server which compression algorithms the client can decompress. Defined in RFC 9110. The server picks the best supported algorithm and sets `Content-Encoding` on the response.

```
Accept-Encoding: gzip, deflate, br, zstd
Accept-Encoding: gzip, deflate, br
Accept-Encoding: gzip
Accept-Encoding: identity
```

### 9.2 The 2026 Compression Landscape

| Algorithm | Identifier | Browser Support | Compression Ratio | Decompression Speed |
|---|---|---|---|---|
| **gzip** | `gzip` | Universal (since 1990s) | Baseline (good) | Fast |
| **brotli** | `br` | Chrome, Firefox, Safari, Edge (since ~2017) | 15-25% better than gzip | Medium |
| **zstd** | `zstd` | Chrome 123+, Firefox 126+, Edge 123+ | Similar to brotli, faster decompression | Very fast |
| **deflate** | `deflate` | Universal but discouraged | Baseline | Fast |
| **identity** | `identity` | Always supported | No compression | Trivial |

Modern browsers in 2026 send `Accept-Encoding: gzip, deflate, br, zstd`. The server should support gzip plus at least one of br or zstd.

Safari notable gap: as of late 2025, Safari does NOT advertise zstd. iOS users still fall back to brotli or gzip. Brotli is the most universal modern format; zstd is the most aggressive choice.

### 9.3 How To Access In Nginx

```nginx
# Variable: $http_accept_encoding

# Modules required for server side compression:
# - gzip: built into nginx (always available)
# - brotli: requires libnginx-mod-brotli (apt install)
# - zstd: requires nginx-module-zstd (third party)

http {
    # gzip (universal fallback)
    gzip on;
    gzip_vary on;
    gzip_min_length 256;
    gzip_comp_level 6;
    gzip_types
        text/plain text/css text/xml application/json application/javascript
        application/xml application/atom+xml application/rss+xml
        image/svg+xml font/ttf font/otf font/woff font/woff2;

    # brotli (modern browsers)
    brotli on;
    brotli_static on;       # serve pre compressed .br files when available
    brotli_comp_level 5;
    brotli_min_length 256;
    brotli_types
        text/plain text/css text/xml application/json application/javascript
        application/xml application/atom+xml application/rss+xml
        image/svg+xml font/ttf font/otf font/woff font/woff2;

    # zstd (cutting edge, Chrome 123+)
    zstd on;
    zstd_static on;
    zstd_comp_level 6;
    zstd_min_length 256;
    zstd_types
        text/plain text/css text/xml application/json application/javascript
        application/xml image/svg+xml;
}
```

Nginx automatically negotiates the best supported algorithm based on the request's Accept-Encoding header.

### 9.4 The Vary: Accept-Encoding Requirement

When compression is in use, `Vary: Accept-Encoding` is mandatory:

```nginx
gzip_vary on;  # nginx adds Vary: Accept-Encoding automatically when gzipping
```

Without this, caches may serve a gzipped response to a client that requested no compression, breaking the response.

### 9.5 The Pre Compressed Static Files Pattern

For maximum efficiency, pre compress static files at build time and let nginx serve them directly:

```bash
# Build time: pre compress with highest levels (no runtime CPU cost)
cd /var/www/sites/example.com
for file in *.html *.css *.js; do
    gzip -k -9 "$file"          # creates file.gz
    brotli -k -q 11 "$file"     # creates file.br
    zstd -k -19 "$file"         # creates file.zst
done
```

```nginx
# nginx serves the pre compressed version when supported
gzip_static on;
brotli_static on;
zstd_static on;
```

For a typical HTML page, this means: brotli level 11 (slow at build time, but only done once) yields about 20% smaller files than gzip level 6 (fast but less efficient). The savings compound across many requests.

### 9.6 Troubleshooting

**Symptom: client sends `Accept-Encoding: gzip, br, zstd` but receives uncompressed response.**
Possible causes:
1. Response Content-Type not in `gzip_types` / `brotli_types` / `zstd_types`. Add it.
2. Response shorter than `gzip_min_length`. Lower the threshold or accept.
3. nginx compression modules not loaded. Verify with `nginx -V 2>&1 | grep -E "gzip|brotli|zstd"`.
4. Already compressed (image, video). nginx correctly skips these.

**Symptom: Content-Encoding header on response but content is double encoded or broken.**
Two compression layers are interfering. Check if a proxy or upstream is also compressing. Disable one.

**Symptom: cache serves gzipped response to client that requested no compression.**
Missing `Vary: Accept-Encoding`. Add `gzip_vary on;`.

**Symptom: Safari user reports site is slow; brotli not negotiated.**
Safari might be sending only `gzip, deflate` (older Safari versions). Brotli was added later. Verify in Safari's Network panel that `br` is in the Accept-Encoding.

**Symptom: Chrome 123+ user complains about broken downloads.**
nginx is serving zstd to a client whose handling broke for some reason. Disable zstd temporarily and observe. Common when third party tools (antivirus, corporate proxies) get in the middle and do not understand zstd yet.

### 9.7 Crawler Encoding Behavior

Googlebot sends `Accept-Encoding: gzip` (only gzip historically). Bingbot sends `gzip, deflate`. ClaudeBot and GPTBot also typically advertise gzip. None of the major crawlers advertise brotli or zstd as of 2026.

For crawlers, gzip is sufficient. The brotli and zstd savings benefit human visitors using modern browsers, which is most traffic.

---

## 10. ACCEPT-CHARSET (LEGACY CHARSET NEGOTIATION)

### 10.1 What It Does

`Accept-Charset` was the original character encoding negotiation header. Defined in RFC 9110 but deprecated in practice. Modern HTTP assumes UTF-8 everywhere.

```
Accept-Charset: utf-8, iso-8859-1;q=0.5
Accept-Charset: utf-8
```

### 10.2 The Reality In 2026

Modern browsers and major HTTP libraries do NOT send Accept-Charset. The MDN documentation states: "Browsers omit this header, as it is no longer needed for content negotiation; servers ignore it."

For Bubbles deployments: ignore Accept-Charset entirely. Always serve UTF-8. The Content-Type header (from framework-http-content-headers.md) declares charset=utf-8 explicitly:

```
Content-Type: text/html; charset=utf-8
```

### 10.3 Edge Cases Where Accept-Charset Might Matter

* Legacy enterprise software still parsing HTTP 1.0 era headers.
* Some older Asian language localizations (Shift_JIS, Big5).
* Embedded systems with limited UTF-8 support.

These are rare. For typical web traffic, treat Accept-Charset as inert.

### 10.4 If You Must Handle It

```nginx
# Default to UTF-8 unless the client explicitly says otherwise
map $http_accept_charset $response_charset {
    default                  "utf-8";
    ~*iso-8859-1             "iso-8859-1";  # legacy support if needed
    ~*shift_jis              "shift_jis";   # Japanese legacy
}
```

In practice, do not do this. Serve UTF-8 always. Convert content at the application layer if a specific edge case requires another charset.

---

## 11. FROM (LEGACY CONTACT IDENTIFIER SOME BOTS USE)

### 11.1 What It Does

`From` is a request header containing an email address identifying the human controlling the requesting agent. Defined in RFC 9110. Largely unused in 2026 but still appears in some crawler traffic.

```
From: googlebot(at)googlebot.com
From: webmaster@example.com
```

### 11.2 The Reality In 2026

Browsers do NOT send this header. Some courteous crawlers and scrapers include it as a "if you have a problem, contact this email" gesture. It is informational, not actionable.

Examples of crawlers that may include From:

* Custom academic crawlers and research bots.
* Older versions of Googlebot (rare in 2026).
* Polite scrapers that want to be reachable if blocked.

### 11.3 How To Access

```nginx
# Variable: $http_from

# Log it for crawler accountability
log_format with_from '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     '"$http_referer" "$http_user_agent" '
                     'from="$http_from"';
```

```python
@app.get("/")
async def index(request: Request):
    from_email = request.headers.get("from", "")
    # Log for accountability if it is a crawler
```

### 11.4 What To Do With From

* **Log it.** If a crawler causes problems, the From header gives you an email address.
* **Do not authenticate by it.** It is freely settable to any value.
* **Do not use for access control.** A spoofed From email grants no real identity.

### 11.5 Practical Bubbles Use

For sites Joseph operates, the From header is essentially never actionable. Log it for completeness; do not invest in tooling around it.

---

## 12. HOST (VIRTUAL HOST SELECTION)

### 12.1 What It Does

`Host` tells the server which virtual host the request is for. Defined in RFC 9110. Mandatory for HTTP/1.1 and later. Allows multiple sites to share a single IP.

```
Host: example.com
Host: www.example.com
Host: api.example.com
Host: example.com:8443
```

For Bubbles, the Host header is what nginx uses to pick which `server_name` matches and which `server` block to route to. With 181 hostnames sharing 169.155.162.118, the Host header is the critical routing key.

### 12.2 How nginx Uses Host

```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;
    # ...
}

server {
    listen 443 ssl;
    server_name api.example.com;
    # ...
}

# The default server catches any unmatched Host header
server {
    listen 443 ssl default_server;
    server_name _;
    return 444;  # close connection without response
}
```

nginx compares the Host header against `server_name` directives. The first match (with priority: exact match > leading wildcard > trailing wildcard > regex > default_server) wins.

### 12.3 The default_server Requirement

Without a `default_server`, nginx falls back to the first server block defined for the listen address. This can produce wrong routing for unmatched Host headers (cache poisoning, information disclosure, accidental SSL certificate use).

**Mandatory pattern:** every Bubbles deployment has a default_server returning 444 (connection close) for unmatched Hosts:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    listen 443 quic reuseport default_server;

    server_name _;

    # Use a snake oil certificate for default SSL
    ssl_certificate /etc/ssl/snakeoil/cert.pem;
    ssl_certificate_key /etc/ssl/snakeoil/key.pem;

    # Reject any request that did not match a known server_name
    return 444;
}
```

444 is a non standard nginx code that closes the connection without sending a response. The client gets a connection reset.

### 12.4 How To Access In Nginx

```nginx
# Variable: $http_host (the raw Host header value)
# Variable: $host (Host header value, falls back to server_name if not set)

log_format with_host '$remote_addr - [$time_local] '
                     '"$request" $status host="$host"';
```

`$host` is preferred over `$http_host` for most uses because it handles edge cases (very old HTTP/1.0 clients without Host header) gracefully.

### 12.5 How To Access In FastAPI

```python
@app.get("/")
async def index(request: Request):
    host = request.headers.get("host", "")
    # When behind nginx with proxy_set_header Host $host:
    # this is the actual client requested Host
```

In FastAPI behind nginx, ensure `proxy_set_header Host $host;` is set so the upstream sees the real Host value, not the upstream's internal hostname.

### 12.6 The Host Header Injection Threat

Host header injection occurs when an application uses the Host header value to construct URLs (in emails, redirects, etc) without validation:

```python
# DANGEROUS
@app.post("/api/reset-password")
async def reset_password(request: Request, email: str):
    host = request.headers.get("host")
    reset_url = f"https://{host}/reset?token={token}"
    send_email(email, reset_url)
```

An attacker sends a request to `example.com` with `Host: evil.com`. The application sends `evil.com/reset?token=...` to the user. User clicks, the token goes to attacker.

**The fix:** use a configured value, not the request's Host:

```python
# SAFE
ALLOWED_HOSTS = ["example.com", "www.example.com"]
CANONICAL_HOST = "example.com"

@app.post("/api/reset-password")
async def reset_password(request: Request, email: str):
    reset_url = f"https://{CANONICAL_HOST}/reset?token={token}"
    send_email(email, reset_url)
```

Or validate the Host header before using:

```python
host = request.headers.get("host", "")
if host not in ALLOWED_HOSTS:
    raise HTTPException(status_code=400, detail="invalid host")
```

### 12.7 The Bubbles Pattern: One Server Block Per Tenant

Joseph's Bubbles serves 27 custom domains plus 40 dedicated subdomains plus 119 wildcard subdomains. Each gets its own routing:

```nginx
# Custom domain
server {
    listen 443 ssl;
    server_name client1.com www.client1.com;
    # ...
}

# Wildcard subdomain (catch all *.thatwebhostingguy.com)
server {
    listen 443 ssl;
    server_name *.thatwebhostingguy.com thatwebhostingguy.com;
    # Logic to map subdomain to client directory
    set $subdomain "";
    if ($host ~ "^([^.]+)\.thatwebhostingguy\.com$") {
        set $subdomain $1;
    }
    root /var/www/sites/$subdomain;
}

# Default (catches everything else, 444s it)
server {
    listen 443 ssl default_server;
    server_name _;
    return 444;
}
```

The Host header is the decision point. nginx does the routing based on it; the application sees a normalized Host via `proxy_set_header Host $host`.

### 12.8 Troubleshooting

**Symptom: requests landing on wrong server block.**
1. server_name patterns conflict. Check priority order (exact > wildcard > regex).
2. SSL certificate does not match Host. Different cert per server is fine; mismatched certs are not.
3. No default_server, fallback is wrong.
4. Host header malformed (port mismatch, IP literal instead of hostname).

**Symptom: cache poisoning via Host header.**
Application is using Host without validation. Restrict to configured canonical hostnames.

**Symptom: SSL handshake error for some clients.**
SNI (Server Name Indication) failure. Very old clients without SNI cannot reach name based virtual hosts. They get the default_server's certificate, fail validation. Few clients affected in 2026.

---

## 13. IF-MODIFIED-SINCE (CONDITIONAL GET BY DATE)

### 13.1 What It Does

`If-Modified-Since` is sent by clients (especially crawlers and caches) to say "I have a copy from this date; only send the body if it has changed since". Defined in RFC 9110. Pairs with the `Last-Modified` response header from framework-http-caching-headers.md.

```
If-Modified-Since: Wed, 22 May 2026 14:30:00 GMT
```

If the resource has not changed since that date, the server returns `304 Not Modified` with no body. The client uses its cached copy. Massive bandwidth savings for both server and client.

### 13.2 How It Works End To End

1. Initial request: client GETs `/page`. Server responds 200 with body and `Last-Modified: Wed, 22 May 2026 14:30:00 GMT`.
2. Subsequent request: client sends `If-Modified-Since: Wed, 22 May 2026 14:30:00 GMT`.
3. Server compares to file's mtime or content fingerprint:
   * **Unchanged**: returns 304 Not Modified, no body. Client uses cached copy.
   * **Changed**: returns 200 with new body and new Last-Modified.

Googlebot specifically requests this behavior. From Google's documentation: "Googlebot's crawlers don't send the headers with all crawl attempts; it depends on the use case. AdsBot is more likely to set the If-Modified-Since and If-None-Match HTTP request headers."

### 13.3 How nginx Handles It

For static files, nginx handles If-Modified-Since automatically. The file's mtime becomes the Last-Modified value, and incoming If-Modified-Since is compared automatically.

```nginx
# Default behavior: nginx handles conditional GET on static files
location /assets/ {
    root /var/www/sites/example.com;
    # nginx compares mtime against If-Modified-Since
    # Returns 304 if unchanged
}
```

For dynamic content, the upstream must handle it:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:9090;
    proxy_cache_revalidate on;  # nginx honors upstream's revalidation
}
```

### 13.4 How FastAPI Handles It

FastAPI does not automatically handle conditional GET; the upstream code must:

```python
from email.utils import formatdate, parsedate_to_datetime
from datetime import datetime, timezone
from fastapi import FastAPI, Request, Response

app = FastAPI()

@app.get("/dynamic-page")
async def dynamic_page(request: Request):
    # Compute or look up the resource's Last-Modified time
    last_modified = datetime(2026, 5, 22, 14, 30, 0, tzinfo=timezone.utc)

    # Check If-Modified-Since
    ims = request.headers.get("if-modified-since")
    if ims:
        try:
            ims_date = parsedate_to_datetime(ims)
            if last_modified <= ims_date:
                return Response(status_code=304)
        except (TypeError, ValueError):
            pass  # malformed header, ignore

    # Generate full response with Last-Modified
    return Response(
        content=generate_content(),
        headers={
            "Last-Modified": formatdate(last_modified.timestamp(), usegmt=True),
            "Cache-Control": "public, max-age=300",
        }
    )
```

### 13.5 Why It Saves Crawler Budget

Googlebot's crawl budget for a site is finite. Each unchanged page that returns 200 with full body wastes:

* Server CPU and bandwidth.
* Crawler bandwidth and processing time.
* Crawl budget that could be spent on new or changed pages.

A site that returns 304 for unchanged pages can effectively double or triple its crawl coverage. New pages get indexed faster because the crawler is not wasting time on unchanged old ones.

### 13.6 The Date Format Strictness

The HTTP date format is strict (RFC 7231 IMF-fixdate):

```
Wed, 22 May 2026 14:30:00 GMT
```

Common errors:

* Missing comma after day name.
* Day not zero padded (`22` correct, `2` wrong).
* Time zone other than GMT (UTC is wrong even though equivalent).
* Year not four digits.
* Day name extension like `Wednesday`.

The Python `email.utils.formatdate(stamp, usegmt=True)` produces the correct format.

### 13.7 Troubleshooting

**Symptom: Googlebot never sends If-Modified-Since.**
Googlebot does not always send it (depends on use case). Not a misconfiguration on your side.

**Symptom: server returns 200 to revisits despite content unchanged.**
1. Server is not comparing If-Modified-Since against Last-Modified.
2. Dynamic content with no Last-Modified concept (always considered "fresh").
3. Application overrode 304 logic.
Fix: in FastAPI, add the conditional GET check (Section 13.4).

**Symptom: 304 returned with body (bug).**
304 must have no body per RFC. Some buggy frameworks send 304 with content. Verify by:

```bash
curl -sI -H "If-Modified-Since: Wed, 22 May 2026 14:30:00 GMT" https://example.com/
# Expected: HTTP/2 304 with no body
```

**Symptom: Clock skew causes incorrect 304s.**
Client and server clocks differ. Use ETag instead (Section 14) which is independent of clock.

---

## 14. IF-NONE-MATCH (CONDITIONAL GET BY ETAG)

### 14.1 What It Does

`If-None-Match` is sent by clients to say "I have content with this ETag value; only send body if the current ETag differs". Defined in RFC 9110. Pairs with the `ETag` response header from framework-http-caching-headers.md.

```
If-None-Match: "abc123"
If-None-Match: "abc123", "def456"
If-None-Match: W/"weak-tag-123"
If-None-Match: *
```

Unlike If-Modified-Since (date based), ETag based revalidation is unaffected by clock skew and can represent any content fingerprint (hash, version number, opaque ID).

### 14.2 How It Works End To End

1. Initial request: client GETs `/page`. Server responds 200 with body and `ETag: "abc123"`.
2. Subsequent request: client sends `If-None-Match: "abc123"`.
3. Server compares to current resource's ETag:
   * **Match**: returns 304 Not Modified, no body.
   * **No match**: returns 200 with new body and new ETag.

### 14.3 Strong vs Weak ETags

```
ETag: "abc123"         (strong: byte for byte identical)
ETag: W/"abc123"       (weak: semantically equivalent, possibly different bytes)
```

* **Strong ETag**: matches only if the content is byte for byte identical. Used for partial range requests, byte level caching.
* **Weak ETag**: matches if content is semantically equivalent. Common for dynamic responses where minor variations exist (timestamps, ad rotations) but the substance is unchanged.

The `W/` prefix marks a weak ETag.

### 14.4 How nginx Handles It

nginx auto generates strong ETags for static files from the file's size and mtime:

```nginx
# Default behavior on static files
# nginx auto generates ETag and compares If-None-Match

location /assets/ {
    root /var/www/sites/example.com;
    etag on;  # default
}
```

For dynamic content, the upstream handles it.

### 14.5 How FastAPI Handles It

```python
import hashlib

@app.get("/dynamic-page")
async def dynamic_page(request: Request):
    content = generate_content()
    etag = '"' + hashlib.md5(content.encode()).hexdigest()[:16] + '"'

    # Check If-None-Match
    inm = request.headers.get("if-none-match", "")
    if etag in [tag.strip() for tag in inm.split(",")]:
        return Response(status_code=304, headers={"ETag": etag})

    return Response(
        content=content,
        headers={
            "ETag": etag,
            "Cache-Control": "public, max-age=300",
        }
    )
```

### 14.6 If-Modified-Since And If-None-Match Together

A client can send both. The RFC specifies that **ETag takes precedence**:

```python
def is_not_modified(request, etag, last_modified):
    inm = request.headers.get("if-none-match")
    if inm is not None:
        # ETag check takes precedence
        return etag in inm.split(",")

    ims = request.headers.get("if-modified-since")
    if ims:
        # Fall back to date check
        try:
            ims_date = parsedate_to_datetime(ims)
            return last_modified <= ims_date
        except (TypeError, ValueError):
            return False

    return False
```

### 14.7 The Wildcard If-None-Match

```
If-None-Match: *
```

The wildcard `*` matches any existing resource. Used for "only respond if the resource does not exist" semantics (e.g., conditional PUT to create only if absent). Rare in GET contexts.

### 14.8 Troubleshooting

**Symptom: 304 returned but the client still seems to make a full request.**
The 304 response should be tiny (headers only). If the client is downloading body, something is wrong:
1. Check Content-Length is 0 on the 304 response.
2. Check that nginx is not adding a body to 304s.

**Symptom: every request gets a new ETag despite content unchanged.**
ETag is being generated from variable data (current timestamp, request ID, random nonce). It must be derived from content. Use a hash of the content body.

**Symptom: 304 served when content has changed.**
ETag is too coarse or based on stale data. Regenerate ETag on every content update.

---

## 15. REFERER (REQUEST ORIGIN, OFTEN ABSENT FOR CRAWLERS)

### 15.1 What It Does

`Referer` (note the historical misspelling preserved in the spec) tells the server which page the request came from. Defined in RFC 9110. Used for analytics, security checks, and content adaptation.

```
Referer: https://example.com/products
Referer: https://www.google.com/
(absent: no Referer sent)
```

The Referer value is the full URL of the referring page (or just the origin, depending on the Referrer-Policy on the source page, see framework-http-security-headers.md).

### 15.2 What Crawlers Send

Most crawlers send Referer when crawling links. The Referer is typically the URL where the link was found:

```
Referer: https://example.com/products
Request: GET /product/widget HTTP/2
```

This tells you the crawler followed a link from `/products` to `/product/widget`. Useful for analytics: see what entry points drive crawl traffic.

Some crawlers do NOT send Referer:

* Direct robots.txt or sitemap fetches.
* Many user initiated AI fetches (Claude-User, ChatGPT-User, Perplexity-User often have empty Referer).
* Privacy aware browsers behind a strict Referrer-Policy.

### 15.3 How To Access

```nginx
# Variable: $http_referer
log_format with_referer '$remote_addr - [$time_local] '
                        '"$request" $status '
                        'ref="$http_referer"';
```

```python
@app.get("/page")
async def page(request: Request):
    referer = request.headers.get("referer", "")
    # Use for analytics, not security
```

### 15.4 Common Server Side Uses

**Acceptable:**

* Analytics: track inbound traffic sources.
* Identifying broken links: if Referer is your own site and the request is a 404, you have a broken internal link.
* Content adaptation hints: tweak content based on entry point (e.g., highlight a specific feature if visitor came from a feature page).

**Unacceptable (Referer cannot be trusted):**

* Security checks ("only allow if Referer is our own page"). Referer can be omitted, spoofed, or set to anything.
* Authentication or authorization.
* CSRF protection (use proper CSRF tokens instead).

### 15.5 The Hotlink Protection Antipattern

Historically, sites used Referer to block hotlinking (other sites linking directly to images):

```nginx
# DANGEROUS: easy to bypass and breaks legitimate uses
location /images/ {
    valid_referers none blocked example.com *.example.com;
    if ($invalid_referer) {
        return 403;
    }
}
```

Problems:
1. Trivially bypassed (curl with custom Referer).
2. Breaks legitimate uses: bookmark managers, archive.org, password managers, IDE previews.
3. Privacy aware browsers send empty Referer; this breaks them.

If you genuinely need to prevent hotlinking, use signed URLs or token based access instead.

### 15.6 Logging Referer For Analytics

Most useful pattern: log Referer and analyze in batch:

```nginx
log_format ref '$remote_addr [$time_local] "$request" $status '
               '"$http_referer" "$http_user_agent"';
access_log /var/log/nginx/access.log ref;
```

Analyze:

```bash
# Top referring domains
awk -F'"' '{print $4}' /var/log/nginx/access.log | \
    grep -oE "https?://[^/]+" | sort | uniq -c | sort -rn | head -20
```

### 15.7 The Referer Field Type

Even though "referrer" is the correct English spelling, the HTTP header is `Referer` (with the misspelling). The CSS property `referrer` (no `Referer`), the JavaScript property `document.referrer`, and the security header `Referrer-Policy` all use the correct spelling. Only the HTTP request header retains the historical typo.

### 15.8 Troubleshooting

**Symptom: Referer often empty for legitimate browser traffic.**
The source page has `Referrer-Policy: no-referrer` or `same-origin`. The visitor's browser is following the policy. Cannot fix from your side.

**Symptom: hotlink protection breaks for legitimate users.**
Remove the Referer check. Use signed URLs or token based protection instead.

**Symptom: analytics shows huge volume of empty Referer traffic.**
Mix of direct visits, privacy browsers, and crawlers. Normal. Treat empty Referer as a category in its own right.

---

## 16. CONNECTION (TRANSPORT DISPOSITION)

### 16.1 What It Does

`Connection` controls what happens to the TCP connection after the response. Defined in RFC 9110. In HTTP/2 and HTTP/3, much of its function is absorbed by the protocol itself.

```
Connection: keep-alive
Connection: close
Connection: upgrade
```

### 16.2 Values

| Value | Meaning |
|---|---|
| `keep-alive` | Reuse this connection for subsequent requests. HTTP/1.1 default. Allows pipelining and avoids TCP setup overhead |
| `close` | Close the connection after this response. Forces a new TCP setup for the next request |
| `upgrade` | Switch protocol (WebSocket: `Connection: Upgrade, Upgrade: websocket`) |

In HTTP/2 and HTTP/3, connections are persistent by default and Connection header is largely irrelevant (or required to be ignored).

### 16.3 How To Access In Nginx

```nginx
# Variable: $http_connection

# Log it (rarely necessary)
log_format with_conn '$remote_addr [$time_local] "$request" $status '
                     'conn="$http_connection"';

# Required for WebSocket upgrade
location /ws {
    proxy_pass http://127.0.0.1:9090;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

### 16.4 The WebSocket Upgrade Pattern

When the client wants to upgrade an HTTP connection to WebSocket, it sends:

```
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
```

nginx must explicitly proxy this:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ""      close;
}

location /ws {
    proxy_pass http://127.0.0.1:9090;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_read_timeout 86400;  # 24 hours for long lived WebSocket
}
```

Without this `Connection: upgrade` passthrough, WebSocket negotiation fails.

### 16.5 Crawler Connection Behavior

Most crawlers prefer `keep-alive` to amortize TCP and TLS setup costs across many requests:

* Googlebot uses HTTP/2 (when supported) which is keep-alive by default.
* ClaudeBot and GPTBot use HTTP/1.1 with keep-alive.
* Older crawlers may explicitly send `Connection: close` for each request.

Either way, nginx handles connection management automatically. No action required.

### 16.6 Troubleshooting

**Symptom: WebSocket connection fails to establish.**
nginx is not passing Connection: upgrade to upstream. Add the `proxy_set_header Connection "upgrade"` configuration.

**Symptom: long lived WebSocket disconnects after 60 seconds.**
nginx default `proxy_read_timeout` is 60 seconds. Increase for WebSocket:

```nginx
proxy_read_timeout 86400;  # 24 hours
proxy_send_timeout 86400;
```

**Symptom: HTTP/2 client sends Connection header.**
HTTP/2 spec says this is invalid but some clients still send. nginx ignores per spec. No action needed.

---

## 17. HOW THESE HEADERS INTERACT

Several specific interactions matter.

### 17.1 Accept-Encoding And Vary

When compression is applied, `Vary: Accept-Encoding` is mandatory on the response (covered in framework-http-caching-headers.md). Caches that ignore this serve gzipped responses to clients that did not request compression, breaking them.

### 17.2 If-None-Match And ETag (Strong/Weak Mode)

Range requests require strong ETags. Weak ETags cannot be used with `If-Range` for partial content negotiation.

### 17.3 If-Modified-Since And If-None-Match On Same Request

When both are present, ETag (If-None-Match) takes precedence. If-Modified-Since is ignored. Document this clearly when implementing conditional GET logic.

### 17.4 User-Agent And Accept-Encoding (Crawler Compression Behavior)

Different crawlers advertise different Accept-Encoding:

| Crawler | Accept-Encoding |
|---|---|
| Googlebot | `gzip, deflate, br` (recent) or `gzip` (older) |
| Bingbot | `gzip, deflate` |
| ClaudeBot | `gzip` |
| GPTBot | `gzip` |
| Modern browser | `gzip, deflate, br, zstd` |

For Bubbles: brotli and zstd savings are for human visitors. Crawlers get gzip and that is fine.

### 17.5 Host And Routing

The Host header is the routing key. SNI (Server Name Indication) over TLS is a separate concept: the client tells the server which hostname during the TLS handshake. Most clients send the same value for both. Mismatches indicate misconfigured proxies or crafted attack traffic.

### 17.6 Referer And Accept-Language (Inferring Locale)

Sometimes Accept-Language is generic (`en`) but Referer reveals a specific market (e.g., from a Spanish language partner site). Cannot be relied on but useful as a hint.

### 17.7 User-Agent Verification And Rate Limit Whitelist

The rate limit whitelist from framework-http-rate-control-headers.md is UA pattern based for simplicity. For sensitive endpoints, layer DNS verification on top.

---

## 18. ASSET CLASS AND USE CASE RECIPES

Paste ready blocks per scenario.

### 18.1 Standard Bubbles request handling

```nginx
# /etc/nginx/nginx.conf (http context)

http {
    # Crawler classification
    map $http_user_agent $crawler_type {
        default                  "other";
        ~*googlebot              "search-google";
        ~*bingbot                "search-bing";
        ~*duckduckbot            "search-ddg";
        ~*applebot               "search-apple";
        ~*claudebot              "ai-train-claude";
        ~*gptbot                 "ai-train-openai";
        ~*ccbot                  "ai-train-cc";
        ~*oai-searchbot          "ai-search-openai";
        ~*chatgpt-user           "ai-user-openai";
        ~*claude-searchbot       "ai-search-claude";
        ~*claude-user            "ai-user-claude";
        ~*perplexitybot          "ai-search-perplexity";
        ~*perplexity-user        "ai-user-perplexity";
        ~*facebookexternalhit    "social-fb";
        ~*twitterbot             "social-twitter";
        ~*linkedinbot            "social-linkedin";
    }

    # Log with crawler type
    log_format request_aware
        '$remote_addr - $remote_user [$time_local] '
        '"$request" $status $body_bytes_sent '
        'host="$host" '
        '"$http_referer" "$http_user_agent" '
        'enc="$http_accept_encoding" '
        'lang="$http_accept_language" '
        'crawler=$crawler_type '
        'rt=$request_time';

    access_log /var/log/nginx/access.log request_aware;

    # Compression stack
    gzip on;
    gzip_vary on;
    gzip_min_length 256;
    gzip_comp_level 6;
    gzip_types
        text/plain text/css text/xml application/json application/javascript
        application/xml application/atom+xml application/rss+xml
        image/svg+xml font/ttf font/otf font/woff font/woff2;

    brotli on;
    brotli_static on;
    brotli_comp_level 5;
    brotli_min_length 256;
    brotli_types
        text/plain text/css text/xml application/json application/javascript
        application/xml application/atom+xml application/rss+xml
        image/svg+xml font/ttf font/otf font/woff font/woff2;

    # Default server: reject unmatched Host headers
}
```

### 18.2 The mandatory default_server

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    listen 443 quic reuseport default_server;

    server_name _;

    # Snake oil cert (not user facing)
    ssl_certificate /etc/ssl/snakeoil/cert.pem;
    ssl_certificate_key /etc/ssl/snakeoil/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    return 444;  # close connection without response
}
```

### 18.3 Modern image negotiation (AVIF then WebP then JPG)

```nginx
map $http_accept $image_format {
    default                  "jpg";
    ~*image/avif             "avif";
    ~*image/webp             "webp";
}

location ~* /images/(.+)\.(jpg|jpeg|png)$ {
    set $base $1;
    set $ext $2;

    try_files
        /images/$base.$image_format
        /images/$base.$ext
        =404;

    add_header Vary "Accept" always;
    add_header Cache-Control "public, max-age=2592000" always;
}
```

### 18.4 Conditional GET on dynamic responses (FastAPI)

```python
from email.utils import formatdate, parsedate_to_datetime
from datetime import datetime, timezone
import hashlib
from fastapi import FastAPI, Request, Response

app = FastAPI()

async def conditional_get_response(request: Request, content: bytes, last_modified: datetime):
    """Return 304 if client has fresh copy, else 200 with content."""
    etag = '"' + hashlib.md5(content).hexdigest()[:16] + '"'

    # ETag check takes precedence
    inm = request.headers.get("if-none-match", "")
    if etag in [tag.strip() for tag in inm.split(",") if tag.strip()]:
        return Response(
            status_code=304,
            headers={
                "ETag": etag,
                "Last-Modified": formatdate(last_modified.timestamp(), usegmt=True),
                "Cache-Control": "public, max-age=300",
            }
        )

    # Date check as fallback
    ims = request.headers.get("if-modified-since")
    if ims:
        try:
            ims_date = parsedate_to_datetime(ims)
            if last_modified <= ims_date:
                return Response(
                    status_code=304,
                    headers={
                        "ETag": etag,
                        "Last-Modified": formatdate(last_modified.timestamp(), usegmt=True),
                    }
                )
        except (TypeError, ValueError):
            pass

    return Response(
        content=content,
        media_type="text/html",
        headers={
            "ETag": etag,
            "Last-Modified": formatdate(last_modified.timestamp(), usegmt=True),
            "Cache-Control": "public, max-age=300",
        }
    )
```

### 18.5 Crawler aware logging for AI visibility analytics

```nginx
# Separate access log for crawler traffic only
map $crawler_type $is_crawler {
    default 1;
    "other" 0;
}

access_log /var/log/nginx/crawlers.log request_aware if=$is_crawler;
```

Then aggregate:

```bash
#!/usr/bin/env bash
# /usr/local/bin/crawler-report.sh

LOG=/var/log/nginx/crawlers.log

echo "=== Crawler hits today ==="
grep "$(date '+%d/%b/%Y')" "$LOG" | \
    grep -oE "crawler=[a-z-]+" | sort | uniq -c | sort -rn

echo ""
echo "=== AI crawler unique URLs today ==="
grep "$(date '+%d/%b/%Y')" "$LOG" | grep "crawler=ai-" | \
    awk '{print $7}' | sort -u | wc -l

echo ""
echo "=== Top URLs hit by AI crawlers ==="
grep "$(date '+%d/%b/%Y')" "$LOG" | grep "crawler=ai-" | \
    awk '{print $7}' | sort | uniq -c | sort -rn | head -20
```

This gives Joseph daily visibility into AI crawler traffic per client site.

### 18.6 The hostnames JSON for Bubbles

For Joseph's reverse DNS verification on the Bubbles server, a config file mapping crawler names to expected hostname suffixes:

```python
# /opt/bubbles/crawler_hostnames.json
{
    "googlebot": ["googlebot.com", "google.com", "googleusercontent.com"],
    "bingbot": ["search.msn.com", "msn.com"],
    "applebot": ["applebot.apple.com", "applebot.cdn-apple.com"],
    "amazonbot": ["amazonbot.amazon", "compute.amazonaws.com"],
    "yandexbot": ["yandex.com", "yandex.net", "yandex.ru"],
    "duckduckbot": ["duckduckgo.com"],
    "facebookexternalhit": ["facebook.com", "fbsv.net"],
    "twitterbot": ["twitter.com", "twttr.com"],
    "linkedinbot": ["linkedin.com"]
}
```

Loaded by the FastAPI sidecar to verify any incoming UA claim.

### 18.7 WebSocket upgrade proxy

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ""      close;
}

location /ws {
    proxy_pass http://127.0.0.1:9090;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 86400;
    proxy_send_timeout 86400;
}
```

### 18.8 Language switcher hint (do NOT auto redirect)

```python
@app.get("/")
async def home(request: Request):
    accept_lang = request.headers.get("accept-language", "")

    # Detect preferred language but do NOT redirect automatically
    preferred = "en"
    if accept_lang.lower().startswith("es"):
        preferred = "es"
    elif accept_lang.lower().startswith("fr"):
        preferred = "fr"

    # Inject hint into the rendered HTML for a switcher banner
    html = render_template("home.html", preferred_language=preferred)
    return HTMLResponse(html, headers={"Vary": "Accept-Language"})
```

Then in the HTML template, if `preferred_language != current_language`, show a switcher banner: "Looks like you might prefer Spanish. Switch to es.example.com?"

### 18.9 Pre compressed assets at build time

```bash
# Build time script (run after each site update)
cd /var/www/sites/example.com

# Find all compressible files
find . -type f \( -name "*.html" -o -name "*.css" -o -name "*.js" -o -name "*.svg" -o -name "*.json" \) | while read -r file; do
    # gzip (level 9, maximum)
    gzip -k -9 "$file"

    # brotli (level 11, maximum)
    if command -v brotli >/dev/null; then
        brotli -k -q 11 "$file"
    fi

    # zstd (level 19, maximum)
    if command -v zstd >/dev/null; then
        zstd -k -19 -q "$file"
    fi
done

echo "Pre compression complete. nginx will serve .gz, .br, .zst variants automatically."
```

With `gzip_static on`, `brotli_static on`, `zstd_static on` in nginx, the pre compressed variants are used directly without runtime compression.

### 18.10 Crawler aware rate limiting (combining with framework-http-rate-control-headers.md)

```nginx
# Use the existing crawler_type map for rate limit decisions
map $crawler_type $rate_limit_skip {
    default          0;  # not a crawler, apply rate limit
    "search-google"  1;  # search crawlers: skip
    "search-bing"    1;
    "search-ddg"     1;
    "search-apple"   1;
    "ai-train-claude" 1;
    "ai-train-openai" 1;
    "ai-train-cc"     1;
    "ai-search-openai" 1;
    "ai-user-openai"   1;
    "ai-search-claude" 1;
    "ai-user-claude"   1;
    "ai-search-perplexity" 1;
    "ai-user-perplexity"   1;
    "social-fb"      1;
    "social-twitter" 1;
    "social-linkedin" 1;
    # "other" stays at 0 (rate limited)
}

map $rate_limit_skip $rate_limit_key {
    1 "";
    0 $binary_remote_addr;
}

limit_req_zone $rate_limit_key zone=site:10m rate=20r/s;
limit_req_status 429;
```

Whitelist via UA pattern; sensitive endpoints layer DNS verification on top.

---

## 19. BUBBLES NGINX REFERENCE BLOCK (PASTE READY)

The complete request side stanza, layered with the previous seven frameworks.

```nginx
# /etc/nginx/nginx.conf (http context)

http {
    # ============= CRAWLER CLASSIFICATION =============
    map $http_user_agent $crawler_type {
        default                  "other";
        ~*googlebot              "search-google";
        ~*bingbot                "search-bing";
        ~*duckduckbot            "search-ddg";
        ~*applebot               "search-apple";
        ~*amazonbot              "search-amazon";
        ~*claudebot              "ai-train-claude";
        ~*gptbot                 "ai-train-openai";
        ~*ccbot                  "ai-train-cc";
        ~*meta-externalagent     "ai-train-meta";
        ~*oai-searchbot          "ai-search-openai";
        ~*chatgpt-user           "ai-user-openai";
        ~*claude-searchbot       "ai-search-claude";
        ~*claude-user            "ai-user-claude";
        ~*perplexitybot          "ai-search-perplexity";
        ~*perplexity-user        "ai-user-perplexity";
        ~*youbot                 "ai-search-you";
        ~*facebookexternalhit    "social-fb";
        ~*twitterbot             "social-twitter";
        ~*linkedinbot            "social-linkedin";
        ~*slackbot               "social-slack";
        ~*discordbot             "social-discord";
    }

    map $crawler_type $rate_limit_skip {
        default          0;
        "search-google"  1;
        "search-bing"    1;
        "search-ddg"     1;
        "search-apple"   1;
        "search-amazon"  1;
        "ai-train-claude" 1;
        "ai-train-openai" 1;
        "ai-train-cc"     1;
        "ai-train-meta"   1;
        "ai-search-openai" 1;
        "ai-user-openai"   1;
        "ai-search-claude" 1;
        "ai-user-claude"   1;
        "ai-search-perplexity" 1;
        "ai-user-perplexity"   1;
        "ai-search-you"  1;
        "social-fb"      1;
        "social-twitter" 1;
        "social-linkedin" 1;
        "social-slack"   1;
        "social-discord" 1;
    }

    # ============= CONTENT NEGOTIATION =============
    # Image format negotiation
    map $http_accept $image_format {
        default                  "jpg";
        ~*image/avif             "avif";
        ~*image/webp             "webp";
    }

    # ============= UPGRADE NEGOTIATION =============
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ""      close;
    }

    # ============= COMPRESSION =============
    gzip on;
    gzip_vary on;
    gzip_min_length 256;
    gzip_comp_level 6;
    gzip_proxied any;
    gzip_types
        text/plain text/css text/xml application/json application/javascript
        application/xml application/atom+xml application/rss+xml
        image/svg+xml font/ttf font/otf font/woff font/woff2;

    brotli on;
    brotli_static on;
    brotli_comp_level 5;
    brotli_min_length 256;
    brotli_types
        text/plain text/css text/xml application/json application/javascript
        application/xml application/atom+xml application/rss+xml
        image/svg+xml font/ttf font/otf font/woff font/woff2;

    # zstd if module available
    # zstd on;
    # zstd_static on;
    # zstd_comp_level 6;
    # zstd_min_length 256;
    # zstd_types text/plain text/css application/json application/javascript;

    # ============= LOGGING =============
    log_format request_aware
        '$remote_addr - $remote_user [$time_local] '
        '"$request" $status $body_bytes_sent '
        'host="$host" '
        '"$http_referer" "$http_user_agent" '
        'enc="$http_accept_encoding" '
        'lang="$http_accept_language" '
        'crawler=$crawler_type '
        'rt=$request_time '
        '$server_protocol "$http3"';

    access_log /var/log/nginx/access.log request_aware;

    # Separate log for crawlers
    map $crawler_type $is_crawler {
        default 1;
        "other" 0;
    }
    access_log /var/log/nginx/crawlers.log request_aware if=$is_crawler;
}
```

```nginx
# ============= DEFAULT SERVER (mandatory) =============
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    listen 443 quic reuseport default_server;

    server_name _;

    ssl_certificate /etc/ssl/snakeoil/cert.pem;
    ssl_certificate_key /etc/ssl/snakeoil/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    return 444;
}

# ============= EXAMPLE.COM (canonical client site) =============
server {
    listen 443 ssl;
    listen 443 quic;
    http2 on;
    http3 on;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_early_data on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    # Security headers
    include snippets/common-security-headers.conf;

    root /var/www/sites/example.com;
    index index.html;

    # ============= IMAGE NEGOTIATION =============
    location ~* /images/(.+)\.(jpg|jpeg|png)$ {
        set $base $1;
        set $ext $2;
        try_files /images/$base.$image_format /images/$base.$ext =404;
        add_header Vary "Accept" always;
        add_header Cache-Control "public, max-age=2592000" always;
    }

    # ============= STATIC ASSETS =============
    location ~* \.(css|js|woff2)$ {
        # nginx auto generates ETag, handles If-None-Match and If-Modified-Since
        add_header Cache-Control "public, max-age=31536000, immutable" always;
    }

    # ============= API ENDPOINTS =============
    location /api/ {
        # Rate limit applied via framework-http-rate-control-headers.md
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ============= WEBSOCKET =============
    location /ws {
        proxy_pass http://127.0.0.1:9090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    # ============= ROOT =============
    location / {
        try_files $uri $uri/ $uri.html =404;
    }
}
```

After deploying:

```bash
nginx -t && systemctl reload nginx
```

Verify request side handling:

```bash
# 1. Default server rejects unknown Host
curl -sI -H "Host: random.example.com" https://169.155.162.118/ -k 2>&1 | head -3
# Expected: connection reset (444)

# 2. Crawler classification works
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1)" https://example.com/ \
    && sudo tail -1 /var/log/nginx/crawlers.log

# 3. Image format negotiation
curl -sI -H "Accept: image/avif" https://example.com/images/hero.jpg
# Expected: served as AVIF if hero.avif exists

# 4. Conditional GET on static
ETAG=$(curl -sI https://example.com/style.css | grep -i etag | awk '{print $2}' | tr -d '\r')
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -1
# Expected: HTTP/2 304

# 5. Compression negotiation
curl -sI -H "Accept-Encoding: gzip, br, zstd" https://example.com/ | grep -i content-encoding
# Expected: content-encoding: br (or zstd if enabled)
```

---

## 20. AUDIT CHECKLIST

Run through these 50 items for any production request handling configuration.

### User-Agent and crawler classification

1. [ ] UA pattern map in nginx covers all major search crawlers (Googlebot, Bingbot, DuckDuckBot, Applebot, Amazonbot).
2. [ ] UA pattern map covers AI training crawlers (ClaudeBot, GPTBot, CCBot, meta-externalagent).
3. [ ] UA pattern map covers AI search/retrieval crawlers (OAI-SearchBot, ChatGPT-User, Claude-SearchBot, Claude-User, PerplexityBot, Perplexity-User, YouBot).
4. [ ] UA pattern map covers social crawlers (facebookexternalhit, Twitterbot, LinkedInBot).
5. [ ] Deprecated `Claude-Web` pattern removed (replaced by `ClaudeBot`).
6. [ ] UA value logged in access log for analytics.
7. [ ] Crawler verification (reverse DNS) implemented for sensitive endpoints (admin, paywall, premium content).

### Host and routing

8. [ ] `default_server` block exists for every listen address (80, 443 SSL, 443 QUIC).
9. [ ] Default server returns 444 (or 421 Misdirected) for unmatched Host.
10. [ ] FastAPI sidecar receives correct Host via `proxy_set_header Host $host`.
11. [ ] No application code uses raw `request.headers["host"]` for URL construction without validation.
12. [ ] All 27 custom domains have their own server block with proper server_name.
13. [ ] Wildcard server (*.thatwebhostingguy.com) routes to subdomain specific directories.

### Accept and content negotiation

14. [ ] Image format negotiation map (avif/webp/jpg) configured.
15. [ ] Vary: Accept set on responses that vary by Accept.
16. [ ] Content-Type negotiation handles JSON vs HTML correctly for API endpoints.
17. [ ] Default content type is text/html with charset=utf-8.

### Accept-Language

18. [ ] Accept-Language logged for analytics.
19. [ ] No auto redirect based on Accept-Language (use hreflang plus UI switcher).
20. [ ] If multilingual: Vary: Accept-Language set on language variant responses.

### Accept-Encoding and compression

21. [ ] gzip enabled with appropriate gzip_types.
22. [ ] brotli module loaded and brotli enabled (if available).
23. [ ] zstd module loaded if maximum compression desired (optional).
24. [ ] gzip_vary on; (ensures Vary: Accept-Encoding).
25. [ ] gzip_min_length set (256 or similar) to avoid compressing tiny responses.
26. [ ] Static assets pre compressed at build time (.gz, .br, .zst variants exist).
27. [ ] gzip_static on; brotli_static on; for pre compressed serving.
28. [ ] Compression negotiated correctly with curl tests against modern UAs.

### Accept-Charset

29. [ ] All responses use UTF-8 (Content-Type includes charset=utf-8).
30. [ ] No legacy charset negotiation code (Accept-Charset should be ignored).

### From

31. [ ] From header logged where present (low priority, informational only).
32. [ ] No authentication uses From for identity.

### If-Modified-Since and If-None-Match

33. [ ] Static assets return 304 when If-Modified-Since matches mtime.
34. [ ] Static assets return 304 when If-None-Match matches generated ETag.
35. [ ] Dynamic FastAPI endpoints implement conditional GET where appropriate.
36. [ ] ETag takes precedence over If-Modified-Since when both present.
37. [ ] 304 responses have no body.
38. [ ] 304 responses include ETag and/or Last-Modified for next revalidation.

### Referer

39. [ ] Referer logged for analytics.
40. [ ] No security checks rely on Referer alone.
41. [ ] No hotlink protection via Referer (use signed URLs instead).

### Connection

42. [ ] WebSocket location has proper upgrade headers (proxy_set_header Upgrade $http_upgrade; Connection $connection_upgrade).
43. [ ] Long lived WebSocket has appropriate proxy_read_timeout.

### Logging and analytics

44. [ ] log_format includes user agent, referer, host, accept-encoding.
45. [ ] Separate access log for crawlers (or crawler tag in main log).
46. [ ] Crawler analytics script runs periodically (daily aggregation).
47. [ ] AI crawler visibility tracked over time.

### Cross cutting

48. [ ] `nginx -t` passes without warnings.
49. [ ] `nginx -T | grep -E "map|default_server|listen"` shows expected configuration.
50. [ ] Curl tests with major crawler UAs return expected behavior.

A site that passes all 50 has correctly configured request handling.

---

## 21. COMMON PITFALLS

Fifteen patterns to recognize and avoid.

**Pitfall 1: Trusting User-Agent for paywall bypass.**
Symptom: scraper accesses paywalled content by spoofing Googlebot UA.
Why it breaks: UA alone is unverifiable. Reverse DNS verification is the actual identity check.
Fix: implement DNS verification (Section 6) for any UA based privilege grant.

**Pitfall 2: Using deprecated Claude-Web in robots.txt.**
Symptom: site believes it is blocking Anthropic but ClaudeBot still crawls.
Why it breaks: Claude-Web was deprecated; ClaudeBot is the current pattern.
Fix: update robots.txt to use `User-agent: ClaudeBot`.

**Pitfall 3: No default_server block.**
Symptom: unmatched Host headers route to whichever server block parses first.
Why it breaks: nginx falls back to first server block defined.
Fix: add a default_server that returns 444 for all listen addresses.

**Pitfall 4: Host header used in password reset URL without validation.**
Symptom: attacker triggers password reset with spoofed Host, victim's reset link points to attacker.
Why it breaks: Host can be set to anything; using it as authority is unsafe.
Fix: use a configured canonical hostname for outbound URLs.

**Pitfall 5: Ignoring conditional GET headers.**
Symptom: Googlebot crawl budget wasted on unchanged pages.
Why it breaks: server returns 200 with full body even when If-Modified-Since/If-None-Match match.
Fix: implement conditional GET (Section 13.4 and 14.5).

**Pitfall 6: ETag generated from non content data.**
Symptom: every request gets new ETag even when content unchanged.
Why it breaks: ETag derived from timestamp or random nonce, not content hash.
Fix: derive ETag from content hash (MD5 or similar of body).

**Pitfall 7: Compression negotiated for already compressed types.**
Symptom: PNG, JPG, MP4 served gzipped (CPU waste, no size savings).
Why it breaks: gzip_types includes image/video MIME types.
Fix: restrict gzip_types to text and code formats.

**Pitfall 8: Missing Vary: Accept-Encoding when compression in use.**
Symptom: cache serves gzipped response to client that did not request compression.
Why it breaks: missing Vary causes cross variant pollution.
Fix: `gzip_vary on;` (nginx auto adds Vary: Accept-Encoding when compressing).

**Pitfall 9: Auto redirect based on Accept-Language.**
Symptom: users frustrated by being redirected to wrong language version.
Why it breaks: Accept-Language reflects browser settings, not user intent.
Fix: use hreflang plus UI language switcher; do not auto redirect.

**Pitfall 10: Hotlink protection via Referer breaks legitimate uses.**
Symptom: bookmark managers, archive.org, password managers all fail to load images.
Why it breaks: Referer is unreliable; many legitimate clients send empty Referer.
Fix: use signed URLs or token based protection instead.

**Pitfall 11: Compressing very short responses.**
Symptom: response is larger after compression due to overhead.
Why it breaks: gzip_min_length too low (default 20 bytes).
Fix: set gzip_min_length 256 or higher.

**Pitfall 12: WebSocket disconnects after 60 seconds.**
Symptom: WebSocket connection drops despite being active.
Why it breaks: nginx default proxy_read_timeout is 60 seconds.
Fix: set proxy_read_timeout 86400 for WebSocket locations.

**Pitfall 13: 304 response with body.**
Symptom: 304 response includes content (bug in framework).
Why it breaks: 304 per RFC must have no body.
Fix: ensure 304 responses return only headers.

**Pitfall 14: Crawler whitelist applied to sensitive endpoints.**
Symptom: spoofed Googlebot UA accesses admin endpoint.
Why it breaks: UA whitelist is sufficient for public pages but not for sensitive ones.
Fix: layer DNS verification on top for sensitive endpoints (login, admin, payment).

**Pitfall 15: Compressing API responses but not setting Vary.**
Symptom: cross origin API client gets gzipped response but no Vary header, downstream cache poisoning.
Why it breaks: missing gzip_vary on or explicit Vary header.
Fix: ensure Vary: Accept-Encoding is set on compressed responses.

---

## 22. DIAGNOSTIC COMMANDS

Reference of every command useful for request header investigation.

### Inspect what your client sends

```bash
# See exactly what curl sends
curl -v -o /dev/null https://example.com/ 2>&1 | grep "^>"

# As a specific crawler UA
curl -v -A "Mozilla/5.0 (compatible; Googlebot/2.1)" -o /dev/null https://example.com/ 2>&1 | grep "^>"
```

### Simulate crawler requests

```bash
# Cycle through major crawler UAs
for ua in \
    "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
    "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)" \
    "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko); compatible; ClaudeBot/1.0" \
    "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko); compatible; GPTBot/1.0" \
    "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko); compatible; OAI-SearchBot/1.0" \
    "Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko); compatible; PerplexityBot/1.0"
do
    echo "=== $ua ==="
    curl -sI -A "$ua" https://example.com/ | head -3
    echo
done
```

### Test conditional GET

```bash
# Get current ETag and Last-Modified
HEADERS=$(curl -sI https://example.com/style.css)
ETAG=$(echo "$HEADERS" | grep -i "^etag:" | awk '{print $2}' | tr -d '\r')
LM=$(echo "$HEADERS" | grep -i "^last-modified:" | cut -d: -f2- | sed 's/^ //')

# Verify 304 on If-None-Match
echo "Testing If-None-Match with ETag $ETAG"
curl -sI -H "If-None-Match: $ETAG" https://example.com/style.css | head -1
# Expected: HTTP/2 304

# Verify 304 on If-Modified-Since
echo "Testing If-Modified-Since with $LM"
curl -sI -H "If-Modified-Since: $LM" https://example.com/style.css | head -1
# Expected: HTTP/2 304

# Verify 200 on stale validators
curl -sI -H "If-None-Match: \"stale\"" https://example.com/style.css | head -1
# Expected: HTTP/2 200
```

### Test compression negotiation

```bash
# What does the server actually negotiate?
echo "Plain gzip:"
curl -sI -H "Accept-Encoding: gzip" https://example.com/ | grep -i content-encoding

echo "Plus brotli:"
curl -sI -H "Accept-Encoding: gzip, br" https://example.com/ | grep -i content-encoding

echo "Plus zstd:"
curl -sI -H "Accept-Encoding: gzip, br, zstd" https://example.com/ | grep -i content-encoding

# Compare compressed sizes
for enc in gzip br zstd; do
    SIZE=$(curl -s -H "Accept-Encoding: $enc" --compressed -w "%{size_download}" -o /dev/null https://example.com/)
    echo "$enc: $SIZE bytes"
done
```

### Verify crawler identity by reverse DNS

```bash
verify_crawler() {
    local ip=$1
    local domain_pattern=$2

    local ptr=$(host "$ip" 2>/dev/null | awk '/pointer/ {print $NF}' | sed 's/\.$//')
    if [ -z "$ptr" ]; then
        echo "FAIL: no reverse DNS for $ip"
        return 1
    fi

    case "$ptr" in
        *$domain_pattern)
            local forward=$(host "$ptr" 2>/dev/null | awk '/has address/ {print $NF}' | head -1)
            if [ "$forward" = "$ip" ]; then
                echo "VERIFIED: $ip is $domain_pattern ($ptr)"
                return 0
            else
                echo "FAIL: forward $forward != $ip"
                return 1
            fi
            ;;
        *)
            echo "FAIL: $ptr does not match $domain_pattern"
            return 1
            ;;
    esac
}

# Test
verify_crawler 66.249.66.1 ".googlebot.com"  # should verify
verify_crawler 8.8.8.8 ".googlebot.com"      # should fail
```

### Analyze crawler traffic patterns

```bash
# Daily crawler hits by type
awk '/crawler=/' /var/log/nginx/access.log | \
    grep -oE "crawler=[a-z-]+" | sort | uniq -c | sort -rn

# Hourly Googlebot rate
grep "crawler=search-google" /var/log/nginx/access.log | \
    awk '{print $4}' | cut -d: -f1-2 | uniq -c

# Top URLs by AI crawler
grep "crawler=ai-" /var/log/nginx/access.log | \
    awk '{print $7}' | sort | uniq -c | sort -rn | head -20

# Unique URLs by AI crawler type
for ct in ai-train-claude ai-train-openai ai-search-openai ai-search-claude ai-search-perplexity; do
    COUNT=$(grep "crawler=$ct" /var/log/nginx/access.log | awk '{print $7}' | sort -u | wc -l)
    echo "$ct: $COUNT unique URLs"
done
```

### Test Host routing

```bash
# Verify default_server rejects unknown Host
curl -sI -H "Host: random.example.invalid" https://169.155.162.118/ -k 2>&1 | head -3
# Expected: connection reset

# Verify Host routing to correct server block
for host in example.com www.example.com api.example.com other.com; do
    echo "=== $host ==="
    curl -sI -H "Host: $host" https://169.155.162.118/ -k 2>&1 | head -3
    echo
done
```

### Server side investigation

```bash
# Show all maps related to request headers
nginx -T 2>/dev/null | grep -A 30 "map \$http_"

# Show all default_server declarations
nginx -T 2>/dev/null | grep "default_server"

# Show compression configuration
nginx -T 2>/dev/null | grep -E "gzip|brotli|zstd"

# Show all log_format definitions
nginx -T 2>/dev/null | grep -A 10 "log_format"

# Apply changes
nginx -t && systemctl reload nginx
```

### Browser DevTools quick reference

In Chrome DevTools Network panel:

1. Click any request.
2. Headers tab: "Request Headers" section shows what the browser sent.
3. Look for: User-Agent, Accept, Accept-Language, Accept-Encoding, Host, Referer, Connection.

Useful for understanding what your real users send vs what curl tests send.

```javascript
// In Console: see what fetch will send
fetch("/test").then(r => console.log(r));
// Then check Network tab > the /test request > Headers tab
```

---

## 23. CROSS-REFERENCES

* [framework-http-caching-headers.md](framework-http-caching-headers.md): Last-Modified and ETag response headers pair with If-Modified-Since and If-None-Match request headers covered here.
* [framework-http-content-headers.md](framework-http-content-headers.md): Content-Type response header is the result of Accept based negotiation. Content-Encoding response header pairs with Accept-Encoding here. Content-Language pairs with Accept-Language.
* [framework-http-seo-headers.md](framework-http-seo-headers.md): X-Robots-Tag and Link headers are responses; the User-Agent header here is how crawlers identify themselves before receiving those responses.
* [framework-http-security-headers.md](framework-http-security-headers.md): Referrer-Policy controls what Referer the browser sends out (response header that affects future requests). Cross origin headers interact with Origin (request) and Host.
* [framework-http-performance-headers.md](framework-http-performance-headers.md): Alt-Svc response header advertises HTTP/3; clients then connect via different transport. Server-Timing depends on the upstream measuring time.
* [framework-http-cors-headers.md](framework-http-cors-headers.md): Origin request header (not in this framework) triggers CORS handling. Access-Control-Request-Method/Headers are preflight request headers.
* [framework-http-rate-control-headers.md](framework-http-rate-control-headers.md): the User-Agent crawler whitelist used here directly drives rate limit bypass decisions.
* [UNIVERSAL-RANKING-FRAMEWORK.md](UNIVERSAL-RANKING-FRAMEWORK.md): the master ranking reference. AI crawler identification is core to GEO (generative engine optimization).
* [SEO-BUILD-REFERENCE.md](SEO-BUILD-REFERENCE.md) v2.4: the build playbook.
* [framework-bot-management.md](framework-bot-management.md): deeper dive on crawler verification, AS number based identification, fail2ban integration.
* [framework-robotstxt-crawlbudget.md](framework-robotstxt-crawlbudget.md): how robots.txt directives interact with the crawler taxonomy here.
* Google crawler documentation: https://developers.google.com/search/docs/crawling-indexing/overview-google-crawlers
* Google verify Googlebot: https://developers.google.com/search/docs/crawling-indexing/verifying-googlebot
* OpenAI bot documentation: https://platform.openai.com/docs/bots
* Anthropic crawler documentation: https://docs.claude.com/en/docs/agents-and-tools/web-fetch-tool (and operator portal)
* Bingbot verification: https://www.bing.com/webmasters/help/which-crawlers-does-bing-use-8c184ec0
* PerplexityBot documentation: https://docs.perplexity.ai/guides/bots
* RFC 9110 (HTTP Semantics): https://www.rfc-editor.org/rfc/rfc9110
* RFC 7234 (HTTP Caching, conditional GET): https://www.rfc-editor.org/rfc/rfc7234
* BCP 47 language tags: https://www.rfc-editor.org/info/bcp47
* MDN HTTP Headers reference: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers

---

## APPENDIX A: ONE PAGE QUICK REFERENCE

For the person who just wants the answer.

### Bubbles default request handling

```nginx
http {
    # Crawler classification (drives logs, rate limit whitelist, analytics)
    map $http_user_agent $crawler_type {
        default                  "other";
        ~*googlebot              "search-google";
        ~*bingbot                "search-bing";
        ~*claudebot              "ai-train-claude";
        ~*gptbot                 "ai-train-openai";
        ~*oai-searchbot          "ai-search-openai";
        ~*claude-searchbot       "ai-search-claude";
        ~*perplexitybot          "ai-search-perplexity";
        ~*claude-user            "ai-user-claude";
        ~*chatgpt-user           "ai-user-openai";
        ~*perplexity-user        "ai-user-perplexity";
    }

    # WebSocket upgrade
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ""      close;
    }

    # Image negotiation
    map $http_accept $image_format {
        default                  "jpg";
        ~*image/avif             "avif";
        ~*image/webp             "webp";
    }

    # Compression
    gzip on; gzip_vary on; gzip_min_length 256;
    brotli on; brotli_static on;
}

# MANDATORY default server
server {
    listen 443 ssl default_server;
    server_name _;
    ssl_certificate /etc/ssl/snakeoil/cert.pem;
    ssl_certificate_key /etc/ssl/snakeoil/key.pem;
    return 444;
}
```

### Header purpose table

| Header | Variable | One line purpose |
|---|---|---|
| User-Agent | `$http_user_agent` | Client identity claim (unverified) |
| Accept | `$http_accept` | Content types client supports |
| Accept-Language | `$http_accept_language` | Languages client prefers |
| Accept-Encoding | `$http_accept_encoding` | Compression algorithms supported |
| Accept-Charset | `$http_accept_charset` | Legacy charset (ignore in 2026) |
| From | `$http_from` | Legacy contact email (log only) |
| Host | `$host` or `$http_host` | Which virtual host (routing key) |
| If-Modified-Since | `$http_if_modified_since` | Conditional GET by date |
| If-None-Match | `$http_if_none_match` | Conditional GET by ETag |
| Referer | `$http_referer` | Where request came from |
| Connection | `$http_connection` | Transport disposition (keep-alive, close, upgrade) |

### Crawler quick reference (2026)

| Crawler | UA pattern | Operator |
|---|---|---|
| Googlebot | `googlebot` | Google |
| Bingbot | `bingbot` | Microsoft |
| ClaudeBot | `claudebot` (NOT claude-web) | Anthropic |
| GPTBot | `gptbot` | OpenAI training |
| OAI-SearchBot | `oai-searchbot` | OpenAI search |
| Claude-SearchBot | `claude-searchbot` | Anthropic search |
| Claude-User | `claude-user` | Anthropic user fetches |
| PerplexityBot | `perplexitybot` | Perplexity index |

### Verification quick reference

| Crawler | Method |
|---|---|
| Googlebot | Reverse DNS + IP range JSON |
| Bingbot | Reverse DNS + IP range JSON |
| OpenAI (GPTBot et al) | Published IP range JSON |
| Perplexity | Published IP range JSON |
| Anthropic (ClaudeBot et al) | Reverse DNS only (no IP range published) |
| Applebot, Amazonbot | Reverse DNS |

### Five rules to memorize

1. Never trust User-Agent alone. Verify by reverse DNS for sensitive endpoints.
2. Always have a default_server returning 444 for unmatched Host.
3. Always implement conditional GET (304 on If-Modified-Since or If-None-Match match).
4. Always negotiate Accept-Encoding correctly (gzip plus brotli at minimum).
5. Use hreflang plus UI switcher for languages; never auto redirect on Accept-Language.

### Five commands every operator should know

```bash
# 1. Verify a request IP belongs to claimed crawler
host 66.249.66.1 && host crawl-66-249-66-1.googlebot.com

# 2. Simulate a crawler
curl -sI -A "Mozilla/5.0 (compatible; Googlebot/2.1)" https://example.com/

# 3. Test conditional GET
ETAG=$(curl -sI https://example.com/file.css | grep -i etag | awk '{print $2}' | tr -d '\r')
curl -sI -H "If-None-Match: $ETAG" https://example.com/file.css | head -1

# 4. Test compression negotiation
curl -sI -H "Accept-Encoding: gzip, br, zstd" https://example.com/ | grep -i content-encoding

# 5. Apply nginx changes
nginx -t && systemctl reload nginx
```

### Three end to end tests

```bash
# 1. Default server rejects unknown Host
curl -sI -H "Host: nonexistent.example" https://169.155.162.118/ -k 2>&1 | head -3
# Expected: connection reset

# 2. Major crawlers classified correctly
for ua in Googlebot bingbot ClaudeBot GPTBot OAI-SearchBot PerplexityBot; do
    curl -sI -A "Mozilla/5.0 (compatible; $ua/1.0)" https://example.com/ > /dev/null
done
sudo tail -6 /var/log/nginx/crawlers.log | grep -oE "crawler=[a-z-]+"
# Expected: six different crawler= tags

# 3. 304 returned for unchanged content
ETAG=$(curl -sI https://example.com/style.css | grep -i etag | awk '{print $2}' | tr -d '\r')
STATUS=$(curl -so /dev/null -w "%{http_code}" -H "If-None-Match: $ETAG" https://example.com/style.css)
[ "$STATUS" = "304" ] && echo "Conditional GET works" || echo "FAIL: got $STATUS"
```

If all three pass AND the daily crawler analytics show expected AI crawler visibility, the request handling stack is correctly wired.

---

End of framework-http-request-headers.md.
