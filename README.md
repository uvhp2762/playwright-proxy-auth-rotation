# playwright proxy authentication: fix the 407, rotate one exit IP per browser context, and keep the scrape running

Nine times out of ten, the code looks right and the request still comes back from your own IP address. You passed a proxy URL, Playwright accepted it, the browser launched, and nothing raised an error. That silence is the whole problem.

Playwright does not read credentials out of a proxy URL. It reads credentials from separate fields. Once you know that, most of the confusing behaviour around authenticated proxies — 407s, "it works in requests but not here", IPs that never seem to change — turns into a short list of specific things to check.

Here's what actually goes wrong, in the order people hit it.

## Playwright drops credentials that live inside the proxy URL

This is the single biggest cause of proxy authentication failures in Playwright, and it fails quietly:

python
# Wrong: the username and password are ignored
browser.new_context(proxy={"server": "http://user:pass@gw.example.com:823"})


The `server` value is parsed as a host and port. The userinfo section is not extracted, so the browser makes an unauthenticated connection and the proxy answers with `407 Proxy Authentication Required` — if you're lucky enough to get an error at all.

The form Playwright expects splits the URL apart:

python
# Right
browser.new_context(proxy={
    "server": "http://gw.example.com:823",
    "username": "user",
    "password": "pass",
})


If you're used to `requests` or Scrapy, where credentials in the URL work fine, this is the assumption that bites you. Libraries that hand you a single `scheme://user:pass@host:port` string are giving you a format Playwright will not use as-is. Either split it yourself or run it through a small parser at the point where you build the config.

One related mix-up worth clearing up while we're here: `httpCredentials` is a different option and does a different job. It answers the *target site's* HTTP Basic auth challenge (a 401 from the page itself). It does nothing for the proxy's 407. If you've set `httpCredentials` and your proxy still rejects you, you've solved a problem you didn't have.

## Set the proxy at the context, not the page

`proxy` is a browser-level or context-level option. It is not a page argument, and there is no per-page override. Pages created inside a context share that context's exit IP, which is what makes rotation predictable:

python
context = browser.new_context(proxy=PROXY_CONFIG)
page = context.new_page()          # uses the context's proxy


Two levels are available. Passing `proxy` to `chromium.launch()` sets it for every context in that browser. Passing it to `browser.new_context()` sets it for one context, which is what you want the moment you care about running different exit IPs in the same script:

python
browser = p.chromium.launch(headless=True)

for i in range(3):
    context = browser.new_context(proxy=proxy_for_session(f"job-{i}"))
    page = context.new_page()
    page.goto("https://api.ipify.org/")
    print(page.inner_text("body"))
    context.close()


Same shape in Node:

javascript
const context = await browser.newContext({
  proxy: {
    server: 'http://gw.example.com:823',
    username: 'user',
    password: 'pass',
  },
});


If you're running the Playwright test runner, the same object goes in `playwright.config.ts` under `use: { proxy: {...} }`, and `test.use({ proxy: ... })` overrides it per file. Keep credentials in environment variables rather than the config file so a repo push doesn't leak them — `process.env.PROXY_USER` on the Node side, `os.environ["PROXY_USER"]` in Python.

## Never trust a page load as proof the proxy works

The failure mode of a dropped credential is not always an exception. On some combinations, the request simply goes direct. A page that returns 200 tells you nothing about which IP returned it.

So test the exit address, and compare it against your own:

python
page.goto("https://api.ipify.org/")
print(page.inner_text("body"))


`https://httpbin.org/ip` works just as well. If the printed address matches your own connection, the proxy is not in play, regardless of what the code says. Build that check into the first run of any new proxy config, and keep it in your debugging bookmarks.

## SOCKS5 plus credentials is the trap that costs people an afternoon

Playwright's proxy option accepts HTTP, HTTPS and SOCKS servers on the `server` field. The `username` and `password` fields are documented for **HTTP proxies only**. Those are two different lists, and SOCKS5 authentication sits in the gap.

Pass credentials to a `socks5://` server and Chromium fails with `net::ERR_PROXY_AUTH_UNSUPPORTED`. The upstream feature request for SOCKS5 authentication (microsoft/playwright#10567) has been open since November 2021 and is still unmerged, so this is not something a version bump is about to fix.

The routes that do work:

1. **Use an HTTP/HTTPS endpoint instead.** Almost every provider that speaks SOCKS5 also exposes an HTTP port, and credentials are supported on that path. If your provider gives you both ports, pick the HTTP one.
2. **Run a tiny local relay.** A small proxy on `127.0.0.1` accepts unauthenticated connections from Playwright and injects the `Proxy-Authorization` header upstream. Playwright then points at `socks5://127.0.0.1:1080` with no credentials, which is a configuration it does support. The cost is one extra process to supervise.
3. **Configure Firefox directly.** Firefox has its own proxy preferences, independent of the automation layer, settable through `firefox_user_prefs`. Stock Firefox gives you unauthenticated SOCKS5 this way; credentials need a patched proxy service, so this route is really for projects that already maintain one.

There's also an older workaround circulating for Chromium: pass the proxy without credentials to the `proxy` option and hand the credentials to Chromium itself through `--proxy-server=socks5://user:pass@host:port` in `args`. It shows up in Stack Overflow answers and it does work in some setups, but it isn't documented behaviour and it breaks as Chromium changes. Treat it as a last resort, not a first move.

## Choosing a proxy that survives past the login page

Getting authentication right is only half of it. A correctly authenticated request from a datacenter ASN still gets blocked on sites that score IP reputation before your page finishes loading — the handshake succeeds, the target simply refuses. That's a provider question, not a code question, and it's why proxy type matters more than proxy count in most Playwright work.

The practical split: datacenter exits for internal endpoints, staging environments, and pages with no bot defence; residential exits for anything that filters on ASN. Residential bandwidth costs more per gigabyte, so route selectively rather than piping every request through it.

For Playwright specifically, what matters in a provider is whether the credential format survives a username-based configuration — because that's the shape you need for per-context rotation. DataImpulse is a reasonable example to work through, since its residential product is priced at $1/GB pay-as-you-go and its gateway uses exactly that username-parameter style.

The gateway is `gw.dataimpulse.com`, port 823 for HTTPS and 824 for SOCKS5. Targeting and session control are appended to the login with a double underscore, keys and values separated by dots, parameters separated by semicolons:

python
from playwright.sync_api import sync_playwright

PROXY = {
    "server": "http://gw.dataimpulse.com:823",
    "username": "your_login__cr.us;sessid.job42",
    "password": "your_password",
}

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context(proxy=PROXY)
    page = context.new_page()
    page.goto("https://api.ipify.org/")
    print(page.inner_text("body"))
    browser.close()


`__cr.us` pins the country to the United States; `sessid.job42` holds the same exit IP for roughly 30 minutes, which is enough for a multi-step flow that needs a stable address. Drop the `sessid` part and you get rotation instead — a new exit IP as requests come in. Multiple countries are comma-separated, and there's an equivalent async version if your pipeline is already `async_playwright`.

👉 [👉 Get DataImpulse's $5 / 5 GB residential intro pack here](https://bit.ly/dataimPulse)

Two billing details worth knowing before you scale a crawler on it. Country-level targeting is included; city, ZIP and ASN targeting are charged as add-ons on the standard residential product, which quietly changes your effective cost per gigabyte if you're filtering to a metro area. And bandwidth is metered per gigabyte, so full page loads with images and fonts burn through credit far faster than the HTML you actually wanted. Blocking them in Playwright is three lines and pays for itself:

javascript
await context.route('**/*.{png,jpg,jpeg,woff,woff2}', route => route.abort());


## DataImpulse plans and pricing

Four proxy products, all pay-as-you-go, all with traffic that does not expire and no subscription attached. The minimum purchase is $5, and intro packs carry a 7-day money-back window on card payments (crypto purchases are excluded, and the refund assumes you haven't consumed most of the traffic). Custom enterprise tiers start at 5 TB across all four products.

| Proxy type | What you get | Entry pack | Standard rate | Larger tiers | Billing | Buy |
| --- | --- | --- | --- | --- | --- | --- |
| Residential proxies | 90M+ residential IPs across 195 countries, rotating and sticky sessions, HTTP(S) + SOCKS5, country targeting included | $5 / 5 GB | $1.00/GB | $50 / 50 GB; $800 / 1 TB (≈$0.80/GB) | One-time PAYG, traffic never expires | [ Start with the residential intro pack](https://bit.ly/dataimPulse) |
| Datacenter proxies | Datacenter IPs with 99.9% uptime, randomized subnet access | $5 / 10 GB | $0.50/GB | $50 / 100 GB; $450 / 1 TB ($0.45/GB) | One-time PAYG, traffic never expires | [ Check the $0.50/GB datacenter pack](https://bit.ly/dataimPulse) |
| Mobile proxies | Real 4G/5G/LTE carrier IPs for the hardest targets | $5 / 2.5 GB | $2.00/GB | $50 / 25 GB; $1,600 / 1 TB ($1.60/GB) | One-time PAYG, traffic never expires | [ See the mobile proxy packs](https://bit.ly/dataimPulse) |
| Premium residential proxies | Higher-trust residential pool, dedicated account manager, all targeting options at no surcharge | $5 / 1 GB | $5.00/GB | $50 / 10 GB; custom pricing from 5 TB | One-time PAYG, traffic never expires | [ Compare the premium residential pool](https://bit.ly/dataimPulse) |

Which one lands in your Playwright script depends on the target, not the price list. Datacenter at $0.50/GB will handle a staging site or an unprotected public dataset for half the cost. Residential at $1/GB is the default for anything that filters on IP reputation. Mobile at $2/GB only makes sense when the target genuinely distinguishes cellular traffic — paying carrier rates for a job datacenter IPs can finish is money set on fire. Premium residential is for the cases where the standard pool isn't clearing a target and a dedicated manager to escalate to is worth the multiple.

On coupons: there isn't a public promo code doing the rounds. The $5 intro pack is the entry deal, and the 20% volume discount at 1 TB is the real price break. If you find a code on a coupon site, treat it as unverified until the checkout page confirms a discount.

## Decoding the errors you'll actually see

| Symptom | Usual cause | What to change |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Credentials were inside the proxy URL and got dropped | Move `username` and `password` into their own fields |
| `net::ERR_PROXY_AUTH_UNSUPPORTED` | SOCKS5 with credentials on Chromium | Use the provider's HTTP port, or a local relay |
| `net::ERR_PROXY_CONNECTION_FAILED` | Wrong host or port, gateway down | Confirm the port matches the protocol (823 HTTPS / 824 SOCKS5 at DataImpulse) and retest |
| Page loads, IP is yours | Proxy not applied to that context | Check the proxy is passed to `new_context()`, and that the page came from that context |
| Auth fine, requests blocked anyway | Datacenter ASN reputation | Switch to residential exits or rotate the session ID |
| Routing error when targeting is narrow | No IP available for that city/ZIP filter | Widen to country level |
| Everything worked yesterday, now nothing does | Metered balance exhausted or a concurrency cap hit | Top up, or reduce parallel contexts |

That last row is worth pinning to your monitor. On pay-per-gigabyte providers, a drained balance and a concurrency ceiling both look like sudden, total failure rather than a gradual slowdown, and it's easy to spend an hour hunting for a code bug that's actually a billing state.

## A short checklist before you blame the proxy

- Credentials are in `username` and `password`, never inside `server`.
- `httpCredentials` is set only if the *site* needs Basic auth.
- Proxy is attached at `new_context()`, and you verified the exit IP rather than the page title.
- Rotating flows create a new context per session, not a new page.
- SOCKS5 paths use a relay or the HTTP port.
- Credentials come from environment variables, not a committed config.
- Image and font requests are aborted, since every megabyte is metered.

Get those seven right and authenticated proxies in Playwright stop being mysterious. The remaining variable is the IP quality behind the gateway — which is the part $1/GB residential, or $0.50/GB datacenter if your targets are forgiving, is there to solve.

👉 [👉 Buy your first traffic pack and test it against your own target](https://bit.ly/dataimPulse)
