# beautifulsoup proxy: routing requests through a proxy, fixing 407s, and buying only the traffic your scraper needs

BeautifulSoup doesn't know what a proxy is. It takes a string of HTML you already have and returns a parse tree. Nothing in `bs4` opens a socket, so nothing in `bs4` can be pointed at a proxy server.

That's the actual answer behind the search, and it's the reason so many "BeautifulSoup proxy" tutorials feel sideways. The proxy belongs to whichever library makes the HTTP call, which in most scripts is `requests`. Once you internalize that split, the whole setup takes four lines, and most of the frustration that follows comes from something else entirely: authentication format, port choice, wrong country on the exit IP, or the fact that the page you want is rendered by JavaScript and never shows up in the HTML your parser receives.

Here's the practical version of all of it.

## The shortest thing that works

python
import requests
from bs4 import BeautifulSoup

proxies = {
    "http":  "http://USERNAME:PASSWORD@gw.dataimpulse.com:823",
    "https": "http://USERNAME:PASSWORD@gw.dataimpulse.com:823",
}

response = requests.get(
    "https://example.com/catalog",
    proxies=proxies,
    timeout=30,
    headers={"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"},
)
response.raise_for_status()

soup = BeautifulSoup(response.text, "html.parser")
print(soup.title.text)


Three details in those lines are worth slowing down for, because each one is a common cause of a broken setup.

The `https` value starts with `http://`. That looks like a typo and isn't. The scheme tells `requests` how to talk to the proxy server, not what the target URL uses. Writing `"https": "https://USERNAME:PASSWORD@host:port"` against a proxy that only accepts plain HTTP on that port is one of the standard ways to get a connection error that looks random.

Credentials go directly in the URL, `user:pass@host:port`. There's no separate `auth=` argument for proxy authentication in `requests`. If your password contains `@`, `:`, or `/`, URL-encode it first (`, `urlquote`) or you'll get an authentication failure with correct credentials.

Set a `timeout`. Residential IPs are real devices belonging to real people; the peer occasionally drops mid-request. Without a timeout, a single dead peer can hang a loop for minutes.

## What the proxy string actually encodes

For DataImpulse, one gateway host serves the traffic and the rest is configured through the username. The pattern, as documented in their own integration guides and confirmed in Proxyway's review, is:


USERNAME__cr.us;state.california;city.sanfrancisco;asn.7922:PASSWORD@gw.dataimpulse.com:823


Two underscores mark the start of the parameters; semicolons separate them; `cr` is country, `sid` is a session id, `sessttl` is the session lifetime in seconds. Country targeting is included in the base rate. State, city, ZIP, and ASN precision costs extra, and in practice it roughly doubles the traffic a request consumes, so treat it as a price multiplier on a per-GB plan rather than a free toggle.

Two practical warnings on the endpoint:

**Copy the host and port from your own dashboard, not from a blog.** Ports differ between the rotating HTTP(S) gateway, the SOCKS5 gateway, and the sticky-session ports (which increment from 10000, per Proxyway). Write-ups disagree on the exact numbers, including this one, and the dashboard is the only source that reflects your plan.

**SOCKS5 has to match on both sides.** If you're on a SOCKS5 port, your `proxies` dict should use `socks5://`, and you'll need `requests[socks]` installed. Mixing the protocol and the port is the second standard cause of a connection refusal.

If you'd rather skip credentials entirely, DataImpulse also supports IP whitelisting: you configure the parameters in the dashboard, save the package config, and then point `requests` at the gateway with no login at all. A package holds one saved configuration at a time.

## Where a working proxy still leaves you blocked

Fixing the proxy is maybe 30% of "my BeautifulSoup scraper keeps failing." The rest is here:

**The page is JavaScript-rendered.** If the content appears in DevTools but not in `response.text`, no proxy will change that. `requests` receives the initial HTML only. You need Playwright/Puppeteer, or a target that serves server-side HTML.

**Your headers look like a script.** A residential IP with a default `python-requests/2.x` User-Agent is still an obvious bot. Send a normal browser UA, and let the session reuse cookies.

**You're hitting one URL hundreds of times in parallel.** Rotating the IP doesn't help if the request pattern is a burst. Throttle, and check the site's `robots.txt` and terms before you scale up.

**You're parsing before checking the response.** `soup.find_all("a")` returns an empty list on a 403 page, and an empty list is easy to misread as "selector is wrong." Call `raise_for_status()` first and save the raw HTML when it fails.

## Rotating vs sticky, and the bit reviewers keep flagging

Per-request rotation is the default on DataImpulse's residential pool: every connection comes out of a different IP, which is what you want for crawling a category page or a list of product URLs.

Sticky sessions are for the cases where the target sets a cookie, paginates against a server-side session, or flags a mid-flow IP change. Configure them with a `sid` and a `sessttl`. The honest caveat, which DataImpulse support gave HostAdvice directly, is that the configured interval is not a guarantee:

> "Our sticky sessions last for 30 minutes on average, could be less or could be more. You can specify up to 120 minutes for the rotation interval, but we can not guarantee that the session will last for so long… Once the user whose IP you are connected to goes offline, the proxy will rotate automatically, replacing it with the next available IP."

That's just how a peer-sourced residential pool behaves. It's worth designing for: assume your sticky session may end early, and don't tie state you can't rebuild to a single `sid`. Also note the concurrency ceiling, 2,000 threads by default, and the blocked domain groups (government, banking and payment domains, bandwidth-sharing platforms, mailing services).

## Error table, in the order you'll meet them

| What you see | What it usually is | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Wrong password, a trailing space in the password, or the account has no traffic left | Re-copy the password from the dashboard, check the balance |
| Connection refused / failed | Protocol and port don't match, or the host includes `http://` | Strip the scheme from the host, verify the port |
| Test passes, wrong country | Targeting written into the wrong field | Parameters belong on the username after two underscores |
| IP changes mid-session | No `sid`, or `sessttl` expired | Add `sid` and `sessttl` (the value is seconds) |
| Empty soup, no exception | You got a block page | Log `response.status_code` and save the HTML before parsing |
| Works for one URL, fails for the next | One target is JS-rendered | Check whether the data exists in the raw HTML at all |

A quick sanity check that costs almost nothing in traffic: request `http://api.ipify.org/` through the proxy and print what comes back. If the country doesn't match your targeting, nothing downstream is trustworthy. DataImpulse's own docs use exactly this call, and it's the right first test.

## How much traffic a BeautifulSoup crawl actually burns

This is where per-GB pricing stops being abstract. You're billed for the bytes that cross the proxy, not for requests, and HTML is small.

A typical catalog or article page is roughly 50 to 150 KB of transferred HTML once compression is in play. Run the division and 1 GB lands somewhere around 7,000 to 20,000 page fetches. Scraping a 5,000-page site is usually a single-digit-dollar job, not a scary one.

The number that blows up the estimate is everything you didn't intend to download: images, fonts, tracking pixels, and API calls made by page scripts. Avoid it by fetching HTML only. `requests.get()` without `stream=True` still loads the body, but it won't pull assets the way a real browser does, which is one place where a plain requests + bs4 stack is genuinely cheaper than a headless browser. If you do move to Playwright for JS-heavy pages, budget several times more traffic per page. DataImpulse's plan widgets let you block hosts, which works as a crude ad-filter and saves real megabytes over a long crawl.

So: start with the $5 intro package, measure your true cost per successful request against your own targets, and scale after that. The 5 GB doesn't expire, so nothing is wasted while you figure it out.

👉 [Get 5 GB of residential traffic for $5 and test it against your own targets](https://bit.ly/dataimPulse)

## Which proxy pool to point BeautifulSoup at

Four pools, and the price difference maps almost exactly onto how much of a fight the target puts up. Community testing is worth reading before the price list, because the cheap pool is not the cheap pool for every job.

| Pool | Rate | Good for | Watch out for |
| --- | --- | --- | --- |
| Datacenter | $0.50/GB | High-volume jobs on sites that don't inspect IP reputation | Proxyway found best-in-class IP uniqueness but a poor success rate and requests three to four times slower than the fastest competitors |
| Residential | $1/GB | The default for scraping: SERP-style checks, e-commerce listings, regional pricing | In Proxyway's April 2026 benchmarks, close to 15% of the US regular-pool IPs resolved to server-type addresses; the premium pool was cleaner |
| Mobile | $2/GB | Logged-in flows, app-adjacent data, platforms that distrust anything else | A meaningful share of the Brazil pool was identified as non-mobile; the cheapest place to burn budget if you don't need it |
| Premium residential | $5/GB | High-stakes targets where a failed request costs more than the traffic | Starts at $5/GB with no discount until very large volumes, so it's hard to justify against enterprise pools for general work |

The sensible default for a first BeautifulSoup project is residential, and the honest reason is that you don't yet know your target's blocking behaviour. Datacenter traffic is five times cheaper, so the moment you confirm a site doesn't care about IP type, move the crawl there and pocket the difference. Mobile only earns its rate on targets that have already rejected residential.

Be clear about what these proxies are not. DataImpulse's own documentation frames the product as rotating residential, mobile, and datacenter proxies for collecting publicly available data. Static ISP addresses and a fully managed scraping API aren't part of the offering, and their own FAQ notes that banking and government sites are off the table.

## Plans and pricing in full

DataImpulse runs a pay-per-traffic model rather than a subscription. You pick a proxy type, type a GB amount, and pay; the traffic sits in your account until you use it. Prices below are the standard buckets per product, and the volume discount at 1 TB applies to residential and mobile.

| Proxy type | Plan | Traffic | Total price | Effective rate | Buy |
| --- | --- | --- | --- | --- | --- |
| Residential | Intro | 5 GB | $5.00 | $1.00/GB | [Start with the residential intro package](https://bit.ly/dataimPulse) |
| Residential | Basic | 50 GB | $50.00 | $1.00/GB | [Order the 50 GB residential block](https://bit.ly/dataimPulse) |
| Residential | Advanced | 1,000 GB | $800.00 | $0.80/GB | [Get the 1 TB residential rate](https://bit.ly/dataimPulse) |
| Residential | Custom | 5 TB+ | From $4,000 | Negotiable | [Ask about residential volume pricing](https://bit.ly/dataimPulse) |
| Datacenter | Intro | 10 GB | $5.00 | $0.50/GB | [Try the datacenter pool for $5](https://bit.ly/dataimPulse) |
| Datacenter | Basic | 100 GB | $50.00 | $0.50/GB | [Buy 100 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Datacenter | Advanced | 1,000 GB | $450.00 | $0.45/GB | [Take the 1 TB datacenter tier](https://bit.ly/dataimPulse) |
| Datacenter | Custom | 5 TB+ | From $2,250 | Negotiable | [Request datacenter volume pricing](https://bit.ly/dataimPulse) |
| Mobile | Intro | 2.5 GB | $5.00 | $2.00/GB | [Test the mobile pool for $5](https://bit.ly/dataimPulse) |
| Mobile | Basic | 25 GB | $50.00 | $2.00/GB | [Order 25 GB of mobile traffic](https://bit.ly/dataimPulse) |
| Mobile | Advanced | 1,000 GB | $1,600.00 | $1.60/GB | [Get the 1 TB mobile tier](https://bit.ly/dataimPulse) |
| Mobile | Custom | 5 TB+ | From $8,000 | Negotiable | [Ask about mobile volume pricing](https://bit.ly/dataimPulse) |
| Premium residential | Intro | 1 GB | $5.00 | $5.00/GB | [See the premium residential entry plan](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |
| Premium residential | Basic | 10 GB | $50.00 | $5.00/GB | [Order 10 GB of premium residential traffic](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |
| Premium residential | Advanced / Custom | 1 TB+ | From $4,000 | $4.00/GB | [Review the premium residential tiers](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

A few conditions that matter more than the headline rate. The entry point across all four product types is $5, and beyond that the standard buckets start at $50; Proxyway notes a $50 minimum transaction whether you're buying or topping up. There's no free tier, so $5 is genuinely the cost of testing. Intro plans carry a 7-day money-back guarantee on card payments if you've used less than 80% of the traffic, while crypto purchases on intro plans aren't refundable. Country-level targeting is included; city, ZIP, and ASN precision costs more.

Payment options are broader than most providers at this price: card via Stripe, PayPal, wire transfer, Alipay, Apple Pay, Google Pay, and four cryptocurrencies through Cryptomus.

## What independent testers found

DataImpulse has been reviewed enough now that you don't have to guess. Proxyway has handed it Newcomer of the Year, Greatest Progress, and Most Flexible Provider awards, and their April 2026 benchmarks found the infrastructure well configured, with high success rates and low latency against a nearby CDN, though the regular residential pool carried that non-residential share noted above and the datacenter pool traded performance for unique IPs. HostAdvice scored it 9.1/10 overall, calling the $1/GB rate "real" and highlighting a live-chat response from a named human agent within roughly seven minutes. DataImpulse's own comparison pages cite a 4.8/5 G2 score, and secondary listings describe an "Excellent" Trustpilot rating around 4.6/5, both of which are worth reading at the source rather than taking secondhand.

The consistent criticism is scale and depth: younger company (founded 2022), thinner coverage in less common geographies, and no static ISP product. If your BeautifulSoup crawler targets US, UK, German, or Brazilian sites, none of that will bite.

## Do you actually need a proxy here?

Worth asking once before you spend anything. A one-off script pulling 40 pages from one site, respecting `robots.txt`, and getting a 200 back needs no proxy. A loop that has already started returning 403s, or that needs to see prices as a local visitor sees them, does. The trigger isn't "scraping," it's "the target started refusing, geo-restricting, or rate-limiting me."

When that happens, the fix is four lines of `requests` config, a clean session that looks like a browser, and a pool sized to the job. Datacenter first if the site allows it, residential as the default, mobile only for the targets that leave you no choice.

👉 [Create a DataImpulse account and pull your gateway credentials from the dashboard](https://bit.ly/dataimPulse)
