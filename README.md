# selenium wire proxy: how to route Chrome through authenticated rotating and sticky proxies in Python (and what the traffic costs)

If you've ever tried `options.add_argument("--proxy-server=http://user:pass@host:port")` and watched Chrome quietly ignore your credentials, you already know why people end up searching for "selenium wire proxy." Chrome's proxy flag takes a host and port. It does not take a username and password. So unless your proxy gateway whitelists your IP, plain Selenium can't authenticate, and your scraper keeps exiting from your own IP while the logs insist everything is fine.

Selenium Wire solves that specific problem. It injects itself between your script and the browser, terminates the connection, and re-sends your request through an upstream proxy with auth headers attached. Same idea as a local MITM proxy, but packaged as a drop-in replacement for `selenium.webdriver`.

The rest of this is what actually matters: getting the connection verified, choosing rotating versus sticky sessions, keeping the browser alive while you swap IPs, and what the traffic behind all that costs per gigabyte.

## Why plain Selenium fails with authenticated proxies

Two separate limitations stack up here.

The first is credential handling. The `--proxy-server` flag accepts `protocol://host:port`, nothing more. No `user:pass@`. You can work around it with a Chrome extension that fills in auth, or by whitelisting your IP on the provider side — both are more moving parts than most projects want.

The second is dynamic switching. Even when a proxy works, Chrome binds it for the lifetime of the browser process. Changing IP means `driver.quit()` and a fresh browser, which is expensive if you're driving a login flow or a page that takes ten seconds to render.

Selenium Wire handles both. It supports credentials embedded in the proxy URL, and it exposes a `driver.proxy` attribute you can reassign mid-session. It also gives you request interception, which is how people block images and fonts to cut both page-load time and (if you pay per GB) bandwidth.

## Install and verify before you write any scraping logic

bash
pip install selenium-wire


Selenium Wire pulls in Selenium as a dependency, so you don't install it separately. Note that the package on PyPI has a community fork (`selenium-wire-lw`) — worth knowing if you hit dependency conflicts with newer Selenium releases.

Verification first. Don't write your target scraper until this prints an address that isn't yours:

python
from seleniumwire import webdriver

options = webdriver.ChromeOptions()
options.add_argument("--headless=new")

seleniumwire_options = {
    "proxy": {
        "http": "http://USERNAME:PASSWORD@GATEWAY_HOST:823",
        "https": "http://USERNAME:PASSWORD@GATEWAY_HOST:823",
    },
    "verify_ssl": False,   # only while testing
}

driver = webdriver.Chrome(options=options, seleniumwire_options=seleniumwire_options)

driver.get("https://httpbin.io/ip")
print(driver.find_element("tag name", "body").text)
driver.quit()


Replace `GATEWAY_HOST`, `USERNAME` and `PASSWORD` with the endpoint from your provider's dashboard. Use `https://` in the proxy URL only if the gateway itself serves TLS; most proxy endpoints expect `http://` even for HTTPS traffic, because the tunnel carries the encryption.

If the printed IP matches your own, the request never reached the proxy. Check that the gateway answers at all with a plain `requests.get(..., proxies={"https": PROXY_URL})` call. If that works and Selenium Wire doesn't, the problem is usually credentials rather than network.

## Rotating or sticky: pick based on the target site

These are not interchangeable, and choosing wrong is the most common reason a working setup still gets blocked or breaks.

Rotating sessions hand you a new IP on every request. On DataImpulse, rotating traffic goes through port 823 (HTTP/HTTPS) or 824 (SOCKS5), so rotation itself costs you nothing in code — you don't manage a list of proxies or call `random.choice()` on a pool that's half dead.

That's fine for SERP scraping, price checks, and anything where each page is independent. It's fatal for flows with state: logins, carts, multi-step forms, anything where the site ties a session cookie to an IP. Rotate halfway through and you get logged out, or worse, flagged.

Sticky sessions pin one IP to one port for a defined window. On DataImpulse, sticky connections use ports in the 10000–20000 range, sessions run from 1 to 120 minutes, and the default when you don't specify an interval is 30 minutes. For a login-and-scrape run, 10–30 minutes is usually the right length.

> Rule of thumb: if the site issues you a session cookie, you want sticky. If it doesn't, rotating will save you a lot of error handling.

## Rotating proxies with geo targeting

Where you want requests to appear from is usually part of the proxy username, not a separate API call. Country targeting is typically free; city, ZIP and ASN filters usually aren't.

python
from seleniumwire import webdriver

USER = "USERNAME"           # exact syntax depends on the provider
PASSWORD = "PASSWORD"
HOST = "GATEWAY_HOST"
PORT = 823                  # rotating HTTP/HTTPS

proxy_url = f"http://{USER}:{PASSWORD}@{HOST}:{PORT}"

driver = webdriver.Chrome(
    seleniumwire_options={"proxy": {"http": proxy_url, "https": proxy_url}}
)

driver.get("https://httpbin.io/ip")
print(driver.find_element("tag name", "body").text)


Then loop. Every new request exits through a different address, and you don't maintain a pool.

## Sticky sessions for a login flow

Same URL shape, different port, plus whatever session identifier the provider wants in the username:

python
STICKY_PORT = 15001   # pick a free port in 10000-20000
proxy_url = f"http://{USER}:{PASSWORD}@{HOST}:{STICKY_PORT}"

driver = webdriver.Chrome(
    seleniumwire_options={"proxy": {"http": proxy_url, "https": proxy_url}}
)

driver.get("https://example.com/login")
# ... log in, navigate, extract ...


Reuse the same sticky port for the next run if you want the same exit IP again, or move to a different port for a fresh one. Sessions expire on their own schedule, so don't assume an IP is still rented an hour later.

## Swapping proxies without restarting the browser

This is the feature that justifies the dependency. Reassign `driver.proxy` and the next request goes out through the new endpoint:

python
driver.proxy = {
    "http": f"http://{USER}:{PASSWORD}@{HOST}:{NEW_PORT}",
    "https": f"http://{USER}:{PASSWORD}@{HOST}:{NEW_PORT}",
}

driver.get("https://httpbin.io/ip")


Cookies, local storage and the rendered DOM survive. For a workflow where you need a cold IP between two stages of the same page, that's a restart you don't pay for.

## What actually breaks in practice

A short list, in rough order of how often it's the real cause.

**Dead free proxies.** The recurring Stack Overflow version of this question always involves an `sslproxies.org` list. Those endpoints die constantly, and a dead proxy often fails silently rather than throwing — you just see your own IP. If you're serious enough to write rotation logic, you're serious enough to pay for IPs.

**TLS verification errors.** `verify_ssl: False` gets test scripts moving past self-signed or MITM'd certificates. Keep it out of production.

**Disk filling up.** Selenium Wire stores captured requests and responses in a `.seleniumwire` folder in your home directory by default. Long crawls write a lot there. Point `storage_base_dir` elsewhere, or set `request_storage="memory"` with `request_storage_max_size` to cap what's held.

**Memory and CPU overhead.** Intercepting every request costs something, and in-memory storage with no cap will grow until the container dies.

**Capture you don't need.** `disable_capture: True` turns off storage while still routing traffic through the upstream proxy. `exclude_hosts` bypasses Selenium Wire entirely for listed addresses — useful for analytics domains you don't want eating proxy bandwidth.

**Headless detection.** Headless Chrome still fingerprints differently from a real browser. A clean residential IP helps, but it isn't a complete answer, and it never was.

## What to look for in the provider behind Selenium Wire

The library is the easy half. The IPs are where projects succeed or fail, and a handful of questions separate a workable provider from an expensive one.

- Does it support username/password auth on a single endpoint? (If not, Selenium Wire's main advantage is moot.)
- Is rotation automatic on a dedicated port, or do you manage a proxy list yourself?
- Are sticky sessions available, and for how long?
- Does traffic expire at the end of the month? Expiring GB is the single biggest hidden cost increase in this space — you buy more than you need and lose the remainder.
- Is country targeting included or billed?
- What's blocked? Every provider maintains a restricted-domain list, and finding out after you've paid is annoying.

👉 [Check DataImpulse's current per-GB rates and supported endpoints](https://bit.ly/dataimPulse)

## DataImpulse's current product lineup and pricing

DataImpulse sells on pay-as-you-go per gigabyte rather than monthly seats. Below is what's published on its product pages. All four proxy types run through the same dashboard and the same auth model, so you can test one and switch without re-writing your script.

| Proxy type | Entry plan | Price per GB | Volume tiers | Billing | Purchase |
| --- | --- | --- | --- | --- | --- |
| Residential | $5 for 5 GB | $1.00/GB | $800 for 1 TB ($0.80/GB); $0.70/GB at 5 TB | Pay-as-you-go, traffic never expires | [Buy residential traffic](https://bit.ly/dataimPulse) |
| Datacenter | $5 for 10 GB | $0.50/GB | $50 for 100 GB; $450 for 1 TB ($0.45/GB); custom from $2,250 for 5 TB+ | Pay-as-you-go, traffic never expires | [Buy datacenter traffic](https://bit.ly/dataimPulse) |
| Mobile | $5 for 2.5 GB | $2.00/GB | $50 for 25 GB; $1,600 for 1 TB ($1.60/GB); custom from $8,000 for 5 TB+ | Pay-as-you-go, traffic never expires | [Buy mobile traffic](https://bit.ly/dataimPulse) |
| Premium residential | $5 for 1 GB | $5.00/GB | $50 for 10 GB; custom pricing from $20,000 for 5 TB+ | Pay-as-you-go, traffic never expires | [Buy premium residential traffic](https://bit.ly/dataimPulse) |

A few things that matter more than the headline rate:

**Advanced targeting costs double on residential.** Country selection (and ASN exclusion) is included. State, city, ZIP and ASN selection are billed at 2× the standard per-GB rate for residential traffic. If your script passes a city filter on every request, budget twice the listed price, not the listed price. Datacenter proxies list the finer targeting as included — confirm current treatment with support before you build a cost model on it.

**Traffic doesn't expire.** Buy 5 GB for a test, use 1 GB, and the other 4 GB are still there next month. For anything experimental, this is worth more than a marginally lower per-GB rate with a monthly reset.

**Volume discounts on mobile and premium residential only start at 1 TB.** Below that you pay list.

**There's a restricted-domain list.** DataImpulse blocks a set of domains for abuse prevention — some `.gov` sites need identity verification plus $100 in spend, banking and payment domains need verification plus $1,000, and a handful of sites (openstreetmap.org, 4chan.org, and others) are simply not available through the proxy. Check the list before scoping a project around a specific target.

**New accounts get a 7-day refund window,** which is a cheaper way to test than committing to a 1 TB tier.

## The cost math for a Selenium Wire run

Bandwidth accounting here is different from an HTTP client, and this trips people up.

Fetching raw HTML for a page costs a few hundred kilobytes. Rendering that same page in a headless browser pulls the images, scripts, fonts and tracker calls too — often 2–15 MB per page once everything loads. If you're paying per GB, that difference is your entire budget.

Say a rendered page averages 4 MB. A thousand pages is roughly 4 GB. On DataImpulse residential at $1/GB that's about $4 per thousand pages; on the datacenter tier at $0.50/GB, about $2. Turn on `exclude_hosts` and block images, and you can cut that materially.

The comparison that matters is cost per *successful* request, not cost per GB. A $0.50/GB pool that gets blocked half the time on your target is more expensive than a $1/GB pool that doesn't — you paid twice for the same page. Start with the smallest plan, measure your own success rate, then scale.

## Practical setup notes

A few habits that keep Selenium Wire runs from degrading.

Keep the proxy URL out of source control. Rotating traffic through a leaked endpoint burns your balance quickly.

Set a traffic limit in the dashboard if you're sharing a plan. DataImpulse supports limits on a rolling window (1 hour, 24 hours, 7 days, 30 days) with an option to suspend usage when the cap is hit — a better failure mode than discovering the overspend later.

Don't reuse a sticky port across parallel workers. Two browsers on the same sticky port share the same exit IP, which defeats the point of running them.

Run one authenticated request through `httpbin.io/ip` before every campaign. Verification takes ten seconds and catches expired credentials, wrong ports, and provider-side changes before they contaminate a day of collected data.

## Quick answers

**Does Selenium Wire work with SOCKS5?** Yes — pass a SOCKS5 URL in the same proxy dict. DataImpulse exposes SOCKS5 rotation on port 824 alongside HTTP/HTTPS on 823.

**Do I need both Selenium and Selenium Wire installed?** No. Selenium comes in as a dependency.

**What's the default sticky session length?** If you don't specify an interval, or pass `0`, it defaults to 30 minutes. The ceiling is 120 minutes.

**Can I use residential and datacenter IPs in the same script?** If the provider runs one endpoint and one auth model, yes — swap the port or the targeting parameter and the same driver keeps working.

## Bottom line

Selenium Wire earns its place for two reasons: it's the only straightforward way to send authenticated proxies through Chrome's automation stack, and it lets you change exit IP without throwing away browser state. Plain Selenium does neither, and workarounds with extensions or IP whitelists run out of road quickly.

Set up the library once, then spend your effort on the part that decides whether the project works — the IPs. Verify the connection before you write the scraper, decide rotating versus sticky based on whether the target issues you a session cookie, and price your budget on rendered bytes rather than HTML bytes. On the provider side, non-expiring traffic and a $5 entry point make testing cheap; just remember that city-level targeting doubles the residential rate, and that a low per-GB price means nothing if the pool gets blocked on the sites you actually need.

👉 [Start with a $5 DataImpulse plan and measure your own cost per successful request](https://bit.ly/dataimPulse)
