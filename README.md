# selenium proxy authentication java: Why ChromeDriver Rejects Your Credentials, Three Fixes That Work, and the Proxy-Port Shortcut That Skips the Problem

Most people searching this have hit the same wall: you set a proxy in Selenium Java, the proxy needs a username and password, and ChromeDriver simply refuses to pass them along. The short answer is that Selenium's `Proxy` class has no credential fields for HTTP proxies, Chrome's native login dialog can't be driven by WebDriver, and every working solution lives outside the standard `ChromeOptions.setProxy()` call. This article covers the three fixes that actually work in Java, then shows a fourth option — a managed proxy port from ScraperAPI — that makes the credential problem almost disappear while also handling rotation and CAPTCHAs for you.

> **Quick pick:** If you just need Selenium Java to work behind an authenticated proxy without fighting 407 dialogs, the fastest route is a managed proxy endpoint with fixed credentials — ScraperAPI's proxy port is `proxy-server.scraperapi.com:8001` with username `scraperapi` and your API key as the password, and a 7-day trial with 5,000 credits costs nothing to try. 👉 [Start the 7-day trial with 5,000 free credits](https://www.scraperapi.com/signup?fp_ref=coupons)

## Why proxy authentication fails in Selenium Java

Selenium's `Proxy` class only carries the proxy address. It has no fields for a username and password on HTTP proxies. When your proxy returns a 407 Proxy Authentication Required response, Chrome pops up a native authentication dialog, and WebDriver cannot interact with that dialog at all — it sits outside the DOM.

A lot of older Stack Overflow answers suggest `proxy.setSocksUsername()` and `proxy.setSocksPassword()`. Those only apply to SOCKS proxies. If your proxy is HTTP (most residential and datacenter gateways are), that code compiles, runs, and does nothing useful. Another common snippet sets `proxy.setHttpProxy(ip + ":" + port)` and `proxy.setSslProxy(...)` — fine for unauthenticated proxies, useless the moment credentials are involved.

So the question is never "how do I type the password into Selenium." It's "how do I get the credentials to Chrome through a channel Selenium controls." That leads to three workarounds.

## Three fixes that actually work in Java

**1. Inject a small Chrome extension that answers the auth prompt.** You generate a tiny extension at runtime containing a `chrome.webRequest.onAuthRequired` listener that returns your credentials, zip it up, and load it with `chromeOptions.addExtension()`. This works with any authenticated proxy and is the most common approach in Java projects. Two constraints worth knowing before you build it: the extension pattern uses manifest version 2 with a minimum Chrome version of 22, and — more importantly — extensions don't work in headless mode on Chrome below 130. If you run headless, you need Chrome 130+ with the `--headless=new` flag.

**2. Use BrowserMob Proxy as a local MITM layer.** This is the approach Oxylabs documents in its own Selenium Java integration repo: BrowserMob sits between Chrome and the target, handles the upstream proxy authentication itself, and supports full man-in-the-middle interception. It's powerful — you can also capture traffic and rewrite headers — but it's another JVM dependency to manage, and it adds moving parts to your test or scraping setup.

**3. IP whitelisting.** Some proxy providers let you whitelist your server's IP so no credentials are needed at all. Then plain `--proxy-server=host:port` in Chrome options works with zero extra code. The catch is portability: whitelisting works fine on a fixed cloud server and falls apart on residential connections or CI machines with rotating egress IPs.

Each of these solves authentication. None of them solves what usually comes next: your proxies get banned, CAPTCHAs start appearing, and you're building rotation logic on top of your auth logic. That's where the fourth option changes the math.

## The proxy-port route: authentication handled once, rotation handled server-side

ScraperAPI offers a proxy front-end to its scraping API. You point Chrome at a single gateway — `proxy-server.scraperapi.com:8001` — with the username `scraperapi` and your API key as the password. Behind that endpoint, the service rotates proxies, solves CAPTCHAs, and retries failed requests automatically, with the same feature set as their REST API endpoint.

Why does this matter for the Selenium authentication problem specifically? Because the credentials never change. The whole reason the extension-injection approach is fragile with ordinary proxy pools is that usernames and passwords rotate or expire, so your generated extension breaks silently. With a fixed `scraperapi:API_KEY` pair, you inject the extension once and it keeps working. Rotation, geo-targeting, and anti-bot handling all happen on the server side — your Java code never sees them.

Turning on features works through dot-notation in the username. JS rendering is `scraperapi.render=true`, and you can stack parameters: `scraperapi.render=true.country_code=us` routes through US IPs with rendering enabled. Sticky sessions — reusing the same IP across a multi-page flow — are controlled via a session parameter, and sessions expire 15 minutes after their last use, which matters if your Selenium flow has long pauses between steps.

One Java-specific caveat that trips people up: in proxy mode, your client must not verify SSL certificates against the standard trust store, because the gateway re-terminates TLS. The documented options are either to skip certificate verification or to download ScraperAPI's CA certificate and import it into your trusted roots. In Java/Selenium terms, people commonly pass `--ignore-certificate-errors` as a Chrome argument for the quick path; for production, importing the provided `proxyca.pem` into the JVM or OS trust store is the cleaner route. The docs include step-by-step import instructions for Windows (via `mmc`), macOS (Keychain Access), and Linux.

Here's what the setup looks like in Java — a generated auth extension pointed at the proxy port:

java
// Extension background.js core (built at runtime, zipped, loaded via addExtension):
// chrome.proxy.settings.set({ value: {
//   mode: "fixed_servers",
//   rules: { singleProxy: { scheme: "http",
//     host: "proxy-server.scraperapi.com", port: 8001 } }
// }});
// chrome.webRequest.onAuthRequired.addListener(
//   details => ({ authCredentials: { username: "scraperapi.render=true",
//                                   password: API_KEY } }),
//   {urls: ["<all_urls>"]}, ['blocking']);

ChromeOptions options = new ChromeOptions();
options.addExtension(buildProxyAuthExtension()); // the zip from above
options.addArgument("--ignore-certificate-errors");
WebDriver driver = new ChromeDriver(options);
driver.get("https://httpbin.org/ip"); // verify the exit IP


With ordinary proxies this same extension breaks whenever the provider rotates credentials. Here it doesn't, because there's nothing to rotate. If that sounds like the trade you want to make, 👉 [check current plan prices and start a trial](https://www.scraperapi.com/pricing?fp_ref=coupons).

## ScraperAPI plans and pricing

Pricing is credit-based, and every plan includes the full feature set — JS rendering, premium proxies, CAPTCHA handling, custom headers, session support, automatic retries, and unlimited bandwidth. Here's the full lineup as shown on the official pricing page (monthly USD; annual billing takes 10% off):

| Plan | Price (monthly billing) | Credits | Concurrency | Geo targeting | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | 1,000 | 5 | US & EU only | [Sign up free](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | 100,000 | 20 | US & EU only | [Start Hobby](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | $149 | 1,000,000 | 50 | US & EU only | [Start Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | $299 | 3,000,000 | 100 | Country-level | [Start Business](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | $475 | 5,000,000 | 200 | Country-level | [Start Scaling](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | $975 | 10,500,000 | 300 | Country-level | [Start Professional](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | $1,975 | 21,500,000 | 500 | Country-level | [Start Advanced](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Custom | 22,000,000+ | 500+ | Country-level | [Contact sales](https://www.scraperapi.com/?fp_ref=coupons) |

New accounts get a 7-day trial with 5,000 credits and no credit card required; after that you drop to the 1,000-credit free plan with 5 concurrent connections. Prices are the ones published on the official pricing page and confirmed by independent reviews through mid-2026, but slider-based entry tiers do change, so confirm the current figure before checkout.

## The credit math most tutorials skip

This is the part that catches new users off guard, and it's rarely explained in the Selenium integration guides: one credit equals one standard page request, but everything else costs more.

- Standard GET: **1 credit**
- JS rendering (`render=true`): **10 credits**
- Premium proxies (`premium=true`): **10 credits**
- Premium + rendering: **25 credits**
- Ultra-premium (`ultra_premium=true`): **30 credits**
- Ultra-premium + rendering: **75 credits**

Certain domains have their own pricing regardless of parameters: Amazon pages cost 5 credits, Google and Bing (including subdomains) cost 25, LinkedIn costs 30. Sites behind Cloudflare, Datadome, or PerimeterX add 10 credits per request when the protection is bypassed.

Run your own numbers before picking a plan. If your Selenium jobs render JavaScript on premium proxies, each page costs 25 credits, so Hobby's 100,000 credits is roughly 4,000 rendered pages per month — not 100,000 requests. Professional's 10.5M credits works out to about 1.05M rendered requests, or 140,000 ultra-premium-plus-render requests. ScraperAPI provides a Domain Cost Estimator in the dashboard so you can price any URL exactly, and you can set a `max_cost` per scrape to keep a single request from burning your balance.

Two policies worth knowing. First, failed requests don't cost you: the API retries internally through different proxies for up to 60 seconds, and only successful responses deduct credits. Second, when you exceed your plan's concurrency limit you get a 429 response telling you to retry — the request is rejected, not queued, and you aren't charged. When credits run out, scraping pauses by default until renewal; Hobby through Professional plans can opt into auto-upgrade to the next tier, while Scaling and above can use pay-as-you-go at a fixed per-credit rate up to a monthly cap you set.

## Which plan fits a Selenium workload

Hobby at $49/month handles light testing and small scheduled jobs — 100,000 standard credits or about 10,000 rendered pages. Startup's 1M credits at $149 covers a modest production scraper running a few thousand rendered pages daily. Business ($299) is the first tier with full country-level geo-targeting across the standard country list; if your Selenium tests need IPs outside the US and EU, this is the real floor. Scaling and above are for sustained volume, and they're also the tiers where pay-as-you-go is available, which acts as a safety valve against hard pauses mid-job.

If you're comparing against raw proxy providers instead of a managed API, the trade is different: providers like Oxylabs sell proxies from around $30/month and give you full control, but you own the rotation, CAPTCHA, and auth plumbing — the BrowserMob setup being a case in point. A managed endpoint costs more per page and buys back your engineering time. Neither is wrong; it depends on whether your bottleneck is budget or developer hours.

## FAQ

**Does the proxy port work with headless Chrome?**
Yes, with the extension method, as long as you're on Chrome 130 or newer and use `--headless=new`. Older headless mode doesn't load extensions, which is why many Java developers hit failures only in CI environments.

**What happens if my Selenium script pauses between steps?**
Sticky sessions expire 15 minutes after the last use. If your flow waits longer than that between requests (waiting for an email, a manual review step), assume a new IP and plan accordingly.

**Is there a free way to test the Java setup?**
Yes — the 7-day trial includes 5,000 credits without a credit card, enough for roughly 500 rendered pages or 5,000 plain requests. 👉 [Claim the free trial and test the proxy port](https://www.scraperapi.com/signup?fp_ref=coupons)

**Do I need to handle CAPTCHAs myself?**
No. The proxy port handles CAPTCHA detection and retries automatically, and requests keep retrying across different proxies for up to 60 seconds before the API returns a 500.

## Before you commit

Match the plan to your actual request profile, not the credit headline: count your rendered and premium requests separately, since they burn 10x to 75x credits. If your targets are mostly US/EU and unrendered, Hobby is plenty. If you need country-level targeting outside US/EU, start at Business. And if your "scraping" is really a handful of Amazon or Google lookups, remember those cost 5 and 25 credits per page — a small plan depletes faster than the credit count suggests.

One last practical check: verify your exit IP with a request to `httpbin.org/ip` before assuming the proxy is live, and decide upfront whether you'll skip SSL verification or import the CA certificate, because mixing both approaches across your codebase creates hard-to-debug inconsistency. When you're ready to stop managing proxy credentials yourself, 👉 [pick a plan and get your API key](https://www.scraperapi.com/pricing?fp_ref=coupons).
