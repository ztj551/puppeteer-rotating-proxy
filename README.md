# puppeteer rotating proxy: how to rotate IPs without 407s, session leaks, or surprise bandwidth bills

Puppeteer has no rotation feature. There's no `rotate()`, no proxy pool, no round-robin helper. There's a launch flag, `--proxy-server`, and it holds one endpoint for the entire life of that browser process. Set it once, expect a fresh IP per request, and you'll get one IP and a wall of 403s — then spend an evening reading GitHub issues about it.

That's the annoying part, but it isn't hopeless. Rotation in Puppeteer happens in one of three places: you relaunch the browser, you point the browser at a gateway that hands out a new exit IP per connection, or you put a local relay in front of Chrome. Pick wrong and you either burn through RAM launching browsers or leak sessions across accounts that should never share an IP. What follows is all three approaches, the authentication step Chrome forces on you, the sticky-vs-rotating decision most tutorials skip, and what it costs per page once billing is part of the equation.

## Why the launch flag doesn't rotate anything on its own

`--proxy-server` goes into `puppeteer.launch({ args: [...] })`. Every page in that browser — tabs, workers, service workers — goes out through that one proxy, and it stays that way until you close the browser. Chrome does not re-pick your exit node between navigations.

The flag also can't carry credentials. This is the single most common reason a proxy "works in curl but not in Puppeteer":

js
// This does nothing useful. Chromium drops the user:pass part
// without throwing an error.
args: ['--proxy-server=http://user:pass@gw.dataimpulse.com:823']


You won't get an exception. You'll get a 407, or a `net::ERR_NO_SUPPORTED_PROXIES`, or in some setups just a hang. Credentials are a separate mechanism (more on that below), and they have to be set before the first navigation runs.

The practical consequence: **rotation is not a Puppeteer setting, it's a property of what sits in front of Chrome.** Once you accept that, the options get clearer.

## Three ways to actually rotate

**1. Browser-per-proxy.** You keep a list of proxy URLs, launch a new browser for each one, and close it when the job ends. Full isolation: separate cookies, cache, connections, fingerprint state. It's also the slowest and the hungriest — each Chromium instance is real memory, and launching one per page is a good way to find your container's memory limit. Works well for a handful of long jobs, poorly for thousands of short requests.

**2. Rotating gateway.** You point `--proxy-server` at a single hostname. The provider decides which exit IP serves each connection, based on the username you authenticate with. New connection, new IP — no relaunching, no pool to maintain. If you don't send a session identifier, most gateways rotate automatically. This is the default choice for crawling.

**3. Local relay via `proxy-chain`.** Apify's `proxy-chain` starts a tiny local proxy that holds your upstream credentials and forwards to the provider, which lets you skip `page.authenticate()` entirely and swap the upstream when you want to change identity:

js
const proxyChain = require('proxy-chain');

const upstream = 'http://YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823';
const localProxy = await proxyChain.anonymizeProxy(upstream);

const browser = await puppeteer.launch({ args: [`--proxy-server=${localProxy}`] });
// ... scraping ...

await browser.close();
await proxyChain.closeAnonymizedProxy(localProxy, true); // don't skip this


That last line matters. Every `anonymizeProxy()` call opens a listener on a random local port. Skip the close and you leak ports until the process runs out of file descriptors — usually somewhere in the middle of the night, on a schedule you didn't supervise.

What you *won't* do is set a different proxy per page. Puppeteer's request interception is built for filtering and rewriting requests, not for changing the network transport mid-flight. Per-context proxy support in Puppeteer is limited compared to Playwright, so the workable substitute is one browser process per identity.

## Rotating vs sticky: the decision that prevents most session failures

Rotation is the right default and the wrong answer for anything stateful. If your target tracks session state server-side — a login, a multi-step checkout, a paginated API that remembers where you are — an IP change mid-flow looks exactly like what it is. The server re-challenges or invalidates, and your retry logic makes it worse by hammering the same broken session.

| Job | Session setup | Why |
| --- | --- | --- |
| Crawling many pages of one site | No session id — rotate per connection | Spreads load across the pool |
| Logged-in session | Fixed session id + TTL | The site sees one stable location |
| Parallel workers | One session id per worker | Workers never share an exit IP |
| Checkout / payment flow | Session id, long TTL | An IP change mid-flow reads as a hijack |
| Isolated scraping jobs | New browser per job | Clean cookies, cache, connections |

One rule worth writing on a sticky note: never reuse a session id across two accounts. Two accounts arriving from one IP link themselves together far more reliably than any fingerprint ever will, and that's true whether you're scraping or managing profiles.

## Wiring it up against a rotating gateway

The concrete example below uses DataImpulse's gateway, because it's a rotating endpoint with free country targeting and per-GB billing, which makes the cost side easy to reason about. The host is the same for residential and mobile traffic: `gw.dataimpulse.com`, port `823` for HTTP/HTTPS. (SOCKS5 runs on a different port — check the dashboard — and Chromium can't do username/password auth over SOCKS5 anyway, so authenticated residential goes on the HTTP endpoint.)

js
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch({
    headless: 'new',
    args: ['--proxy-server=http://gw.dataimpulse.com:823', '--no-sandbox'],
  });

  const page = await browser.newPage();

  // Runs BEFORE page.goto(). Reverse the order and your first
  // request goes out unauthenticated and dies with a 407.
  await page.authenticate({
    username: 'YOUR_LOGIN__cr.us;sid.job01;sessttl.600',
    password: 'YOUR_PASSWORD',
  });

  await page.goto('https://ip-api.com/json', { waitUntil: 'domcontentloaded' });
  console.log(await page.evaluate(() => document.body.innerText));

  await browser.close();
})();


Two things are happening in that username string. `__cr.us` is country targeting, and it's part of the base price on residential. `sid.job01;sessttl.600` pins a session id for 600 seconds, which is what turns a rotating gateway into a sticky one. Drop the `sid` and you get a fresh exit IP on every connection. Keep it and the IP holds for the TTL — DataImpulse supports sticky windows from a minute up to 120 minutes, with 30 minutes as the default when you don't specify one.

For parallel workers, derive the session id from the worker index so two workers can never collide:

js
const username = `YOUR_LOGIN__cr.us;sid.w${workerId};sessttl.600`;


Always verify the exit IP before you trust the setup. Send ten requests through the same session id and assert you see exactly one address; send ten without one and assert you see several. That single check catches credential mistakes, misconfigured targeting, and a sticky session that isn't actually sticky. You can also see the exact username format your account expects, plus the error codes for this gateway, in 👉 [the official DataImpulse Puppeteer tutorial](https://dataimpulse.com/tutorials/how-to-set-up-proxies-in-puppeteer/?aff=86938).

## What this costs, and why headless browsing is the expensive way to fetch a page

Raw proxies bill bandwidth, not requests. That makes Puppeteer the worst-case client for your bill, because a headless Chrome visit isn't a 40 KB HTML fetch — it's HTML plus scripts, stylesheets, fonts, and images. Median pages have been over 2 MB for years, and a full asset load through a browser comfortably exceeds that. At $1/GB, a 2 MB page works out to roughly $0.002, or about $2 per 1,000 pages. Same page size at $5/GB is $10 per 1,000.

There's a second cost that rarely shows up in tutorials: you pay for bytes in a 403 exactly as you pay for bytes in a 200. A Cloudflare challenge page is 30–80 KB of traffic you bought and can't use, and if your retry loop fires three times against a blocked host, you've paid for all four responses. Failure rate is not a footnote to the price, it's part of it. Rotating is what keeps that failure rate down; blocking what you don't need is what keeps the numerator down. Both matter.

Puppeteer makes the trimming easy, because request interception is already there:

js
await page.setRequestInterception(true);
page.on('request', (req) => {
  if (['image', 'media', 'font', 'stylesheet'].includes(req.resourceType())) {
    req.abort();
  } else {
    req.continue();
  }
});


Blocking those resource types removes the large majority of a page's payload while leaving the DOM you actually parse intact. On a $1/GB plan that's the difference between a pipeline that costs a few dollars a month and one that costs a few hundred.

## DataImpulse plans, compared

DataImpulse prices by traffic under a pay-as-you-go model. There's no subscription, no monthly commitment, and purchased traffic doesn't expire — which matters for rotation workloads, because scrapers are lumpy. You run a big crawl, then nothing for three weeks, then another one. Billing that resets monthly charges you for the quiet weeks anyway.

| Plan | Best for | Entry package | Rate | Billing | Purchase |
| --- | --- | --- | --- | --- | --- |
| Residential | Rotating general-purpose crawling; the default for Puppeteer | $5 / 5 GB | $1/GB (from $0.80/GB at the 1 TB tier) | Pay as you go, traffic never expires | [Start with residential](https://bit.ly/dataimPulse) |
| Datacenter | Fast, cheap targets with weak bot protection | $5 / 10 GB | $0.50/GB (from $0.45/GB at 1 TB) | Pay as you go, traffic never expires | [Start with datacenter](https://bit.ly/dataimPulse) |
| Mobile | 4G/5G targets that block residential ranges | $5 / 2.5 GB | $2/GB (from $1.60/GB at 1 TB) | Pay as you go, traffic never expires | [Start with mobile](https://bit.ly/dataimPulse) |
| Premium Residential | High-trust traffic, dedicated account manager, all targeting included | $5 / 1 GB | $5/GB | Pay as you go, traffic never expires | [Start with premium residential](https://bit.ly/dataimPulse) |

The network sits at 90M+ ethically sourced IPs across 195 countries, supports HTTP(S) and SOCKS5, and publishes a 99.51% success rate. Datacenter plans advertise 99.9% uptime. Country targeting is included in the base rate on every plan; state, city, ZIP, and ASN filters are a paid add-on on standard residential traffic, billed at double the per-GB rate, which is worth knowing before you write `ci.newyork` into a username template and wonder why the invoice doubled.

Two caveats on this pricing. Mobile and premium residential volume discounts only kick in at the 1 TB tier, so small mobile jobs pay the headline $2/GB with no ladder down. And if you need static ISP proxies, a fully managed scraping API, or access to banking and government sites, this isn't the right vendor — the product is rotating residential, mobile, and datacenter infrastructure for collecting public data. The $5 entry package is enough to measure your own cost per successful page before committing to anything larger.

For most Puppeteer work, the residential line is the correct starting point: datacenter IPs are cheaper and faster but get flagged quickly on anything with real bot protection, and mobile is the escalation you reach for only after residential starts failing on a specific host. 👉 [See the current DataImpulse pricing and plan details here](https://bit.ly/dataimPulse).

## The failures you'll actually hit

| Symptom | Cause | Fix |
| --- | --- | --- |
| 407 Proxy Authentication Required | Credentials in the flag, or `page.authenticate()` called after `goto()` | Authenticate first, navigate second; or wrap with `proxy-chain` |
| `ERR_PROXY_CONNECTION_FAILED` | Endpoint unreachable, or the port doesn't match the protocol | Confirm host and port match the protocol you're using |
| `ERR_TUNNEL_CONNECTION_FAILED` | The proxy rejected the HTTPS CONNECT tunnel | Verify the endpoint supports CONNECT; test the same credentials with curl |
| `ERR_NO_SUPPORTED_PROXIES` | Malformed proxy string, protocol prefix mismatch | Check the string format against a working curl command |
| `TimeoutError` on navigation | Distant exit node, or the default 30s timeout is too tight | Raise the timeout to 60s and test proxy latency separately |
| 403 or a CAPTCHA wall | Flagged IP, or a fingerprint that reads as automated | Rotate to a fresh IP, set a realistic user agent and viewport, add delays between requests |
| Request "succeeds" with 200 but the data is wrong | Soft block serving a challenge or honeypot page | Validate the content, not the status code |

That last row is the one that quietly corrupts datasets. A 200 with a login wall inside it parses fine and gives you garbage. Check for expected selectors before you trust a response, and treat a missing key element as a block rather than a layout change.

One more diagnostic note: `headless: false` plus `slowMo` will show you what Chrome is actually doing, and `NODE_DEBUG="puppeteer:*"` produces verbose protocol logs. Those logs can contain credentials and target URLs in full, so don't leave them on against production accounts.

## Habits that keep rotation cheap

- **Rotate *and* back off.** Twenty requests per second from a fresh IP is still twenty requests per second. Add jittered 2–8 second delays and cap retries. An unbounded retry loop against a blocked host bills you for every attempt.
- **Send `Accept-Encoding: gzip, deflate, br`.** You pay for compressed bytes on the wire, and HTML compresses roughly 4:1.
- **Cache anything on a schedule.** Re-fetching a page whose ETag hasn't changed is bandwidth you paid for twice. Price monitoring is the usual offender.
- **Match the proxy type to the target, not to the whole job.** Most scraping work is a long tail of easy domains plus a handful of hard ones. Paying residential rates for the easy ones is how budgets disappear.
- **Don't leave `sid` in by accident.** The default rotating behaviour is what you want for broad crawling. A stray session id in a username template quietly pins every request to one IP, and the symptom — rising block rates with no obvious cause — looks nothing like the actual problem.
- **Give sticky sessions to the flows that need them, and nothing else.** Login and multi-step flows need IP continuity. Listing pages don't.

## Quick answers

**Can I use a different proxy for each page in Puppeteer?** Not natively. `--proxy-server` is browser-scoped, and per-request transport switching isn't what request interception is for. Use separate browser instances per identity, or a rotating gateway that assigns a new exit IP per connection.

**Will rotation alone stop the blocks?** No. IP reputation is one signal among several — TLS and HTTP/2 fingerprints, request cadence, referrer consistency, and header coherence all get evaluated. Rotating IPs while every request carries an identical, perfectly timed header set produces traffic that looks less human, not more.

**Do I need residential, or will datacenter do?** Test datacenter first, since it's half the price per GB. Move to residential for the specific hosts that return 403s, and to mobile only for targets that block residential ranges too. Paying $2/GB for a target that never blocks datacenter IPs is money you didn't need to spend.

**What does the minimum buy-in look like?** $5. That's 5 GB of residential traffic at $1/GB, and the traffic doesn't expire, so a failed experiment doesn't evaporate at the end of the month. 👉 [Set up a DataImpulse account and point your Puppeteer script at the gateway](https://bit.ly/dataimPulse).
