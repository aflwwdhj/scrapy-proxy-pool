# scrapy proxy pool: 3 ways to set one up, and when you can skip building one entirely

If you're searching for "scrapy proxy pool," chances are your spider got blocked after a few requests and you're now deciding between three paths: building a pool of free proxies over a weekend, buying rotating proxies and wiring them into middleware, or paying a scraping API like ScraperAPI to handle the whole thing. The short version: all three work, and they sit at three different points on a cost curve — time spent versus monthly budget versus how often your crawl stalls. For anything beyond a throwaway project, the hosted route wins on time, and ScraperAPI is the simplest to bolt onto an existing Scrapy project because nothing gets refactored.

Here's the quick map before the details:

> **Zero budget and willing to maintain things:** build your own pool from free proxies (Scrapy + Redis). It works, but free proxies die within hours, so maintenance is constant.
>
> **Already paying for proxies:** use a rotating middleware and let Scrapy distribute requests.
>
> **Want the pool, headers, CAPTCHAs, and retries handled for you:** route requests through ScraperAPI. The free plan gives you 5 concurrent threads to test with, and paid plans scale up from there. 👉 [Check ScraperAPI's plans and current pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Three ways to build a proxy pool in Scrapy

First, a quick clarification on what "proxy pool" means in a Scrapy context. Scrapy supports HTTP proxies natively through its built-in `HttpProxyMiddleware` (part of the framework since version 0.8), so using a proxy is already handled — you just need to tell it which proxy per request. A proxy pool is simply the automation layer that answers that question: pick a working IP, rotate it, and weed out the dead ones.

You can solve that at three levels:

1. **Self-built pool from free proxies** — a scraper constantly harvests proxies from list sites, tests them, and keeps the alive ones in Redis; a middleware injects them into your requests.
2. **Middleware for your own paid proxies** — you buy a list of IPs from a provider, and an open-source or custom middleware rotates them and retries failures.
3. **A scraping API that manages the pool for you** — you send requests to an endpoint, and the service handles IP rotation, headers, geotargeting, retries, and CAPTCHA solving.

ScraperAPI is a useful benchmark for the third route, because if you've already decided to outsource, you can point your Scrapy project at it instead of maintaining anything else. 👉 [The official integration guide shows the three connection options with runnable code](https://www.scraperapi.com/quick-start-guides/python-scrapy-integration/?fp_ref=coupons). Let's take the three options in order.

## Option 1: build a free proxy pool with Scrapy + Redis

This is the closest thing to what people picture when they search "scrapy proxy pool": a fully self-hosted system where a scraper process collects proxies from list sites, tests them, and stores the working ones in Redis. Several open-source projects on GitHub follow this pattern — one well-known example is the Time1ess ProxyPool project, which pulls proxies from four sources, runs a web console on port 5000, and lets you define rules with XPath expressions to match the page structure of any proxy list site you want to add.

Once the pool is populated, a Scrapy middleware distributes the proxies across your requests. The existing projects expose a surprising number of tuning knobs; typical settings from these codebases include:

python
PROXY_POOL_ENABLED = True
PROXY_POOL_FILTER_CODE = 'us'          # country filter
PROXY_POOL_REFRESH_INTERVAL = 900      # re-check proxies every 15 min
PROXY_POOL_PAGE_RETRY_TIMES = 5        # retries before counting a page failure


Two behaviors worth knowing before you commit to this route. First, ban detection is site-specific: the common heuristic treats a proxy as dead when the response status isn't 200/301/302/404/500, the body is empty, or an exception occurs — and you can override it with a custom policy, for instance by flagging any response body containing the word "captcha." Second, when every proxy in the pool is dead, the pool can either stop the spider entirely or, in some configurations, fall back to sending requests from your own host IP — which quietly defeats the purpose of the pool, so it's worth checking which behavior your setup has.

The honest trade-off: public proxies are shared, slow, and frequently die within hours — a complaint that shows up in every scraping forum, and it's a property of the free-proxy ecosystem, not of any particular project. A free pool is a container you constantly repair. That's fine for learning and fine for a low-volume hobby crawler. For anything with a deadline, the hours spent babysitting dead IPs cost more than a paid setup does.

## Option 2: rotate paid proxies with middleware

The middle path is buying real proxies and letting a middleware rotate them. The classic solution is the `scrapy-proxies` project, which rotates proxies, filters out bad ones, or pins all requests to a single proxy. The core mechanism is deliberately simple: a middleware sets `request.meta['proxy']` on each request, like this:

python
request.meta['proxy'] = 'user:pass@host:port'


or at the request level:

python
yield scrapy.Request(url, callback=self.parse, meta={'proxy': 'user:pass@host:port'})


What you still own yourself: header rotation, retry logic, and CAPTCHAs. Those either become your code or additional middleware, and that's the hidden cost of this option. On pricing, proxy providers vary widely; Proxy Rotator, for example, starts at $24.95/mo according to its pricing page, and providers like WebShare offer rotating endpoints that plug into the same middleware pattern. This setup is reasonable when your project is already deeply tied to raw HTTP proxy infrastructure — specific exit IPs, custom tooling, existing contracts.

One thing to check in your `settings.py` if you go this route: middleware ordering. Custom proxy middlewares typically register around priority 100, and `HttpProxyMiddleware` sits at 110 — if your custom middleware is registered after it and things silently don't work, ordering is the usual suspect. When you pass authentication through the proxy, the `user:pass@host:port` format handles it, or you can base64-encode credentials into a `Proxy-Authorization: Basic` header.

## Option 3: use ScraperAPI and make the proxy pool someone else's problem

ScraperAPI takes a different approach: instead of maintaining proxies, you send every request through ScraperAPI, which rotates IPs from its own pool, picks the headers most likely to succeed, retries failures, and solves CAPTCHAs. You send a URL, and you get HTML back. Its stated success rate runs 97% or higher, and requests that do fail come back as a 500 status code without being charged — after three retries, almost everything succeeds unless the target site itself is down.

There are three ways to wire it into Scrapy, and all three have identical functionality — the official repo's phrasing is that they "just offer you the flexibility to integrate Scraper API in the way that is easiest for you":

**1. API endpoint.** Rewrite the target URL into a ScraperAPI request:

python
from urllib.parse import urlencode

API_KEY = 'YOUR_API_KEY'

def get_scraperapi_url(url):
    payload = {'api_key': API_KEY, 'url': url}
    return 'http://api.scraperapi.com/?' + urlencode(payload)


**2. Official SDK.** `pip install scraperapi-sdk`, then:

python
from scraperapi_sdk import ScraperAPIClient
client = ScraperAPIClient(API_KEY)

yield scrapy.Request(url=client.scrapyGet(url=url), callback=self.parse)


**3. Proxy mode.** If your codebase already uses `meta['proxy']` everywhere, this is the least intrusive path — ScraperAPI behaves like an ordinary proxy:

python
meta = {
    "proxy": "https://scraperapi:YOUR_API_KEY@proxy-server.scraperapi.com:8001"
}
for url in urls:
    yield scrapy.Request(url=url, callback=self.parse, meta=meta)


One detail that matters from a pool perspective: `session_number=123` keeps the same IP across multiple requests, which you need when a site should see one consistent visitor across a paginated session. Omit it, and you get a rotating IP per request — the behavior most people actually want from a pool. Other request parameters include `render=true` (JavaScript rendering), `country_code=us` (geotargeting), `device_type=desktop|mobile`, `keep_headers=true`, and `autoparse=true` for structured JSON output on supported sites.

If you want to try this, you can start without a credit card: signing up gets you a 7-day trial with 5,000 API credits, and the permanently free plan adds 1,000 free credits with up to 5 concurrent connections. 👉 [Get the free ScraperAPI plan and test it against your target sites](https://www.scraperapi.com/?fp_ref=coupons)

## ScraperAPI plans: pricing, concurrency, and the credit system

Here's the full plan lineup from the current official pricing page. Paid plans are billed monthly or annually, and the annual billing takes **10% off** — the annual prices below are the discounted ones:

| Plan | Monthly | Annual (10% off) | Credits / month | Concurrent threads | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | $0 | 1,000 free credits | 5 | [ Start Free](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [ See Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [ See Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [ See Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [ See Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [ See Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [ See Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | custom quote | custom quote | 22,000,000+ | 500+ | [ Contact Sales](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Beyond what's in the table, three plan-level restrictions don't show up in the price columns but do affect which plan makes sense:

- **Geotargeting:** Hobby and Startup are limited to **US and EU** proxies. Country-level geotargeting for anywhere else starts at Business.
- **Analytics dashboard:** Hobby only keeps 30 days of history; Business and up have no limit.
- **Concurrency overages:** if you exceed your thread limit, requests get a 429 and are turned away rather than queued — there's no extra charge, but no extra throughput either.

Every paid plan includes the same core feature set: JS rendering, premium residential and mobile IPs, rotating proxy pools, custom header and session support, CAPTCHA and anti-bot bypassing, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee.

## The credit system most reviews don't explain

A standard HTML page costs 1 credit — but not every page is standard. ScraperAPI prices harder targets according to its official pricing page:

| Target type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon | 5 |
| Google and Bing (including subdomains) | 25 |
| LinkedIn | 30 |
| Sites behind Cloudflare / Datadome / PerimeterX (when bypassed) | +10 |
| Premium residential/mobile proxies (`premium=true`) | 10 (25 combined with JS rendering) |

Three practical consequences. First, check the cost of *your actual target* before estimating volume: there's a Domain Cost Estimator in the dashboard that tells you exactly what any URL costs, and you can set a `max_cost` per scrape so an Amazon crawl can't silently eat your quota. Second, two features multiply costs — JavaScript rendering plus premium proxies together run 25 credits per request — so apply them selectively. Third, there are safety nets: when credits run out, **auto-upgrade** (opt-in, available on Hobby, Startup, Business, Scaling, and Professional) moves you to the next plan automatically, while **pay-as-you-go** (available on Scaling, Professional, Advanced, and Enterprise only — not Hobby or Startup) lets you keep scraping past your limit at a fixed per-credit rate up to a monthly cap you set.

If your monthly volume is anywhere near a plan boundary, the annual-billing discount plus the 10% difference matters less than picking the right tier — going one tier up at annual rates usually beats auto-upgrade charges on the tier below. 👉 [See the annual-billing prices and pick the tier that matches your monthly volume](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Scrapy settings to change when using ScraperAPI

Dropping ScraperAPI into a default Scrapy project works, but runs slower than it needs to. Four settings from the integration guide need changing, and one of them contradicts what most tutorials tell you:

python
CONCURRENT_REQUESTS = 10        # match your plan's thread count
DOWNLOAD_DELAY = 0
RANDOMIZE_DOWNLOAD_DELAY = False
RETRY_TIMES = 3
ROBOTSTXT_OBEY = False          # only when routing through the API endpoint


The reasoning: download delays normally exist to protect *your* IPs and proxies from bans. When every request goes through ScraperAPI's rotation, there's nothing to rate-limit, and `DOWNLOAD_DELAY = 0` with `RANDOMIZE_DOWNLOAD_DELAY = False` simply stops throttling you. Similarly, Scrapy's default robots.txt check interferes with API endpoint routing, so when you take that path the check is redundant. Set `CONCURRENT_REQUESTS` to whatever your plan allows — per the official guide, scaling from a few hundred pages a day to millions is a configuration change, not an infrastructure rebuild.

Header behavior is worth understanding too. By default, ScraperAPI picks the headers it judges most likely to succeed for each site. Only override when you actually need them — say, an auth cookie for a logged-in session — by setting `keep_headers=true`:

python
yield scrapy.Request(
    url,
    callback=self.parse,
    meta={'proxy': f'https://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
          'keep_headers': True},
    headers={'X-MyHeader': '123'}
)


One more Scrapy-specific convenience: Scrapy skips SSL verification by default, so no extra certificate handling is needed when requests go through the proxy port.

## FAQ

**Does Scrapy support proxies natively?** Yes. HTTP proxies are supported through `HttpProxyMiddleware` since Scrapy 0.8 — set `meta['proxy']` per request, or set a proxy at the environment level. A proxy pool is just the layer that automates *which* proxy per request.

**Do failed ScraperAPI requests cost credits?** No. Requests that don't succeed come back as a 500 status code and aren't charged, per the official integration guide. The 97%+ success-rate figure and the no-charge-on-failure rule both come from that guide, not from marketing copy.

**Is a free proxy pool actually viable?** For experiments, yes; for reliable scraping, no. Public proxies are shared, slow, and often die within hours — which is why every serious project built on them spends most of its time validating and replacing dead IPs.

**How do I keep the same IP across requests?** Add `session_number=123` (any number) in ScraperAPI; the same session number reuses the same proxy across consecutive requests. With a free proxy pool, pinning an IP is your own job.

**Can I send my own headers instead of letting ScraperAPI choose?** Yes — set `keep_headers=true` and pass your headers normally. Without it, the API overrides with its optimized header set, which generally gets better success rates but won't work if the target site requires specific values.

## Which one should you actually use?

A decision rule rather than a recap:

- **One-off crawl of one or two sites, and you can tolerate intermittent failures** → a free Scrapy + Redis proxy pool. Budget one evening, accept that it will break.
- **You already have a paid proxy list and need raw HTTP proxies with specific exit IPs** → a rotating middleware like scrapy-proxies. You build header rotation and retry logic yourself.
- **Your crawler targets anything actually defended — Amazon, Google, LinkedIn, or sites behind Cloudflare or DataDome** → ScraperAPI is the practical choice, because the bypass logic is part of the product rather than your weekend project, and failed requests aren't billed.

One last check before you commit: test against *your specific* target sites rather than toy pages, since credit costs vary by domain from 1 to 30 per page, and your concurrency needs depend on how aggressively you rotate. The free tier is enough to run that test. If it goes well, 👉 [open ScraperAPI's pricing page and start on the plan that matches your crawl volume](https://www.scraperapi.com/pricing/?fp_ref=coupons)
