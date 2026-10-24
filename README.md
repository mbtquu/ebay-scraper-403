# ebay scraper python: Why Your Script Gets a 403 (and the Setup That Actually Pulls Listings, Prices and Seller Data)

Most ebay scraper python tutorials start the same way: `pip install requests BeautifulSoup`, hit `https://www.ebay.com/sch/i.html?_nkw=...`, parse `.s-item__title`. That code is fine. The problem is the first request, which in mid-2026 returns a **403** with a 1,832-byte "Error Page | eBay" instead of listings — even with a full Chrome User-Agent and complete browser headers. Parsing was never the hard part on eBay. Getting any 200 response at all is.

This guide covers why the DIY approach stalls, what fields you can realistically pull from eBay, when a free open-source scraper is enough, and how a scraping API like ScraperAPI handles the blocking layer for you — with its current plan prices, credit math, and refund policy so you know what it actually costs before signing up.

> **Quick take:** For a side project or occasional price checks, ScraperAPI's free plan (1,000 credits/month, plus 5,000 bonus credits during your first 7 days) is genuinely enough. If you're tracking prices daily, Hobby at **$49/month** (or **$44.10** billed annually) with 100,000 credits is the realistic starting point. 👉 [See all plans and start the 7-day free trial](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Why your Python eBay scraper gets blocked

eBay fingerprints the whole request, not just the User-Agent string. Python's `requests` library produces a TLS and HTTP/2 handshake that looks nothing like a real Chrome session — different JA3/JA4 fingerprints, different HTTP/2 frame ordering. eBay can reject the bot at the network layer before it ever reads your headers. That's why stuffing your script with realistic `Accept`, `Accept-Language`, and `Sec-Fetch-*` headers doesn't help: the verdict is already in.

A documented test from June 2026 makes the point concretely. A GET request to the eBay search page for "mechanical keyboard," sent with a Chrome 124 User-Agent and full browser headers, returned:


STATUS: 403
BYTES: 1832
TITLE: Error Page | eBay
ITEMS: 0


Twice. Same result both times. There's no cookie-free `requests` path that returns a real listing page.

Beyond the TLS-level block, three practical traps catch almost everyone writing an eBay scraper:

1. **Rate-based blocking.** A handful of requests from one IP is fine. Hundreds in a short window gets you rate-limited or served CAPTCHA pages instead of listings.
2. **Pagination quirks.** eBay's `_pgn` parameter controls pagination, and it silently caps out — go past the limit and eBay redirects you to the last valid page without any error. If your scraper doesn't detect that, you'll happily save the same page of results forty times.
3. **Regional domains.** ebay.com, ebay.co.uk, and ebay.de show different pricing and different listings. If your price tracking spans regions, you need geotargeting per request, which means rotating exit IPs in the right country.

One thing working in your favor: eBay's descriptive fields and prices sit in static HTML. Once you get a 200 response, parsing is straightforward. The retrieval layer is where scripts die.

## What you can extract, and the selectors that currently work

An eBay listing page gives you a lot once you're past the block. Here's the field map that third-party scraping guides are using right now:

| Field | Where | Typical selector |
| --- | --- | --- |
| Listing title | Search + item page | `.s-item__title` / `h1.x-item-title__mainTitle` |
| Price | Search + item page | `.s-item__price` / `.x-price-primary` |
| Buying format (auction / Buy It Now) | Search | `.s-item__purchase-options` |
| Bids / time left | Search + item page | `.s-item__bids` / `.x-bid-count` |
| Shipping cost | Search + item page | `.s-item__shipping` |
| Item condition | Search + item page | `.s-item__subtitle` / `.x-item-condition` |
| Seller name + feedback % | Item page | `.x-sellercard-atf__info__about-seller` |
| Item specifics (brand, model) | Item page | `.ux-labels-values` rows |
| Item ID | URL | `/itm/{itemId}` |
| Image URLs | Search + item page | `img` in `.s-item__image` |

The sold/completed listings data deserves a special mention. Active listings show what a seller *hopes* to get; sold data shows what buyers actually paid. If you're doing resale arbitrage or dropshipping price research, that distinction is the whole game.

Two caveats. First, class names aren't guaranteed stable — item pages occasionally render differently by category, so a selector that works for electronics can fail on collectibles. Second, eBay has been actively tightening its stance: the current robots.txt (version `v28_COM_June_2026`) opens with a policy statement, not just directives, and explicitly prohibits automated scraping, buy-for-me agents, and LLM-driven bots. In other words, the parsing layer you write today may need maintenance tomorrow, and eBay's tolerance for automated access is only moving in one direction.

## Three routes to eBay data, honestly compared

Broadly, there are three paths, and they suit very different people.

**Route 1: DIY with requests + BeautifulSoup.** Free, and fine for learning. Several open-source eBay scrapers on GitHub (including a Scrapy-based one from ScrapeOps that exports to CSV/JSON and a no-API-key CLI scraper) give you working parsing code for the `s-item` structure. The catch: same-IP bursts get throttled within minutes, and any class-name change breaks your script. You end up maintaining proxy pools and retry logic instead of analyzing data.

**Route 2: eBay's official Browse API.** Free with a developer key, and the right first choice for catalog data. The limitation is coverage — it doesn't expose everything you might want (sold-listing depth, for instance), and it can't render arbitrary search pages the way a scraper can.

**Route 3: A scraping API.** You send your target eBay URL to an endpoint like `api.scraperapi.com` with your API key, and the service handles IP rotation, JS rendering, CAPTCHA handling, and retries. You get back HTML (or pre-parsed output) that you parse locally. This is the middle path: no proxy infrastructure to maintain, and it costs credits.

A dedicated eBay scraping setup — like [ScraperAPI's eBay-specific solution](https://www.scraperapi.com/solutions/ecommerce-data-collection/ebay-scraper/?fp_ref=coupons) — addresses all four eBay problems directly: it rotates across a large IP pool, applies geotargeting per request, and automatically retries failed calls. If regional pricing accuracy matters for your use case, that per-request geotargeting is the feature doing the real work.

The honest framing: if you need five search pages once a month, don't pay anything — use the official API or a free scraper. If you need thousands of listings daily without babysitting a proxy stack, this is what you're paying for.

## ScraperAPI plans and current pricing

All seven paid tiers share the same core feature set: JS rendering, premium residential and mobile proxies, CAPTCHA and anti-bot handling, JSON auto-parsing, custom headers and sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Here's the full lineup from the official pricing page:

| Plan | Monthly price | Annual (billed) | Credits | Concurrency |
| --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000/mo | 5 |
| Hobby | $49 | $44.10/mo | 100,000 | 20 |
| Startup | $149 | $134.10/mo | 1,000,000 | 50 |
| Business | $299 | $269.10/mo | 3,000,000 | 100 |
| Scaling | $475 | $427.50/mo | 5,000,000 | 200 |
| Professional | $975 | $877.50/mo | 10,500,000 | 300 |
| Advanced | $1,975 | $1,777.50/mo | 21,500,000 | 500 |
| Enterprise | Custom | Custom | 22M+ | 500+ |

Two limits worth knowing before you pick: **Hobby and Startup only include US & EU geotargeting** (country-level targeting worldwide starts at Business), and on Hobby/Startup your analytics dashboard only shows the last 30 days.

| Plan | Where to get it |
| --- | --- |
| Hobby | [Start Hobby at $44.10/mo billed annually](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | [Check Startup plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | [View Business plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | [See Scaling plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | [View Professional plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | [See Advanced plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | [Discuss a custom Enterprise plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Every new account also gets a **7-day trial with 5,000 free credits** — no credit card required — and a **7-day no-questions-asked refund** after any purchase. If the service doesn't fit your eBay use case, contact support within a week and you get your money back. That makes the trial-to-paid decision genuinely low-risk: you can benchmark your exact scraping target before committing.

## The real cost: credits, not just dollars

This is the part most comparison posts skip. ScraperAPI bills in credits, and a credit is not always a page.

A standard page costs 1 credit. Heavier targets cost more: Amazon is 5, Google and Bing are 25, LinkedIn is 30, and sites behind Cloudflare, Datadome, or PerimeterX protection add 10 credits per request when the bypass succeeds. eBay isn't on the published multiplier list, so it typically bills at the standard rate — but don't take that on faith. The dashboard includes a **Domain Cost Estimator** where you can check the exact cost of any URL before you burn through credits, and you can set a `max_cost` per request so a single expensive scrape can't blow your budget.

Do the math for Hobby at the annual rate: $44.10/month for 100,000 credits works out to roughly **$0.44 per 1,000 standard pages**. At the monthly rate it's $0.49. For tracking a few hundred eBay listings a day, 100,000 credits covers a lot of search pages and item lookups before you'd need to upgrade.

A few billing details that only show up in the docs:

- **Credits don't roll over** between billing cycles. Use them or lose them.
- **Exceeding concurrency doesn't cost anything.** You get a 429 and are asked to retry within your plan's limit — requests are rejected outright, not queued and billed.
- **Running dry doesn't hard-stop you on higher tiers.** On Scaling and above, Pay-As-You-Go continues at a fixed per-credit rate up to a monthly spending cap you set — a cap, not a pre-charge. On Hobby through Professional, you can opt into auto-upgrade, which bumps you to the next tier (usually a better price-per-credit) when credits run out.
- **Cancel anytime** from the dashboard at no charge, and card details can be changed anytime on the Billing page.

👉 [Run the credit estimator on your target eBay URLs and start free](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## What a working setup looks like

The code side stays simple because the hard part is outsourced. The pattern that third-party guides use:

python
import requests
from bs4 import BeautifulSoup

API_KEY = "your_key_here"
target = "https://www.ebay.com/sch/i.html?_nkw=airpods+pro"

response = requests.get(
    "https://api.scraperapi.com",
    params={"api_key": API_KEY, "url": target},
)
soup = BeautifulSoup(response.text, "lxml")

for item in soup.select(".s-item__info"):
    title = item.select_one(".s-item__title")
    price = item.select_one(".s-item__price")
    print(title.get_text(strip=True) if title else None,
          price.get_text(strip=True) if price else None)


Useful extras worth knowing about: add `output_format=markdown` to get the page back pre-structured for LLM pipelines, and use custom session support if a multi-page flow needs consistent cookies. There's also a DataPipeline product for bulk crawls if you outgrow request-by-request calls.

One caution that applies regardless of tooling: eBay's User Agreement prohibits bots accessing the site without permission, and *eBay v. Bidder's Edge* (N.D. Cal. 2000) established that aggressive automated querying can be enjoined on a trespass-to-chattels theory. US case law (*hiQ v. LinkedIn*, *Van Buren*) generally treats scraping public pages as outside CFAA, but stay on public pages, never script a logged-in account, avoid personal data, and get legal advice before any commercial-scale operation. For heavy commercial needs, eBay's official API under its license agreement is the sanctioned route.

## FAQ

**How many listings can I scrape with the free plan?**
1,000 credits per month at roughly one credit per standard eBay page, with 5 concurrent connections. During your first 7 days you get 5,000 bonus credits for larger-scale testing. Enough for periodic checks; not enough for daily tracking across hundreds of SKUs.

**Why does my scraper work for a few requests and then get CAPTCHAs?**
Rate-based blocking. eBay tolerates a small number of requests per IP, then escalates to rate limits and CAPTCHA pages. The fix is IP rotation across a large pool, which is exactly what a scraping API provides.

**Do I need JS rendering for eBay?**
eBay's listing fields sit in static HTML, so JS rendering isn't strictly required to parse them — but it matters if a page variant loads content client-side, and it can change the credit cost of a request. Check the Domain Cost Estimator.

**What happens if my credits run out mid-month?**
By default scraping pauses until credits renew. On Hobby through Professional you can enable auto-upgrade to the next tier; on Scaling and above, Pay-As-You-Go keeps you running at a fixed rate up to your own spending cap.

**Which plan for tracking thousands of SKUs daily?**
Business or Scaling, mainly for concurrency — 100 and 200 threads respectively — plus country-level geotargeting worldwide if you're comparing ebay.com against ebay.de prices.

## Where to start

Match the plan to what you're actually doing:

- **Learning or a one-off project:** the free plan's 1,000 monthly credits, plus the 7-day 5,000-credit trial, will cover it. Spend nothing.
- **Regular price checks or a resale side project:** Hobby at $44.10/month billed annually. 100,000 credits is a lot of search pages at standard rates.
- **Regional price tracking across thousands of SKUs:** Business or Scaling, for the concurrency and worldwide geotargeting. If you routinely blow through credits, Scaling's built-in Pay-As-You-Go with a spending cap beats getting 429'd mid-run.

Before you buy anything, do one thing: sign up, grab the 5,000 trial credits, and run your actual target URLs through the Domain Cost Estimator and a few live scrapes. Success rates vary by target, and your specific eBay category — electronics, collectibles, sold listings — is the only benchmark that matters. 👉 [Start the 7-day free trial and test your eBay targets](https://www.scraperapi.com/signup?fp_ref=coupons)
