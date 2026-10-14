Vinted Deal Monitor
A small self-hosted tool that watches Vinted for underpriced clothing and sends the best finds to a Discord channel, so I can buy, relist and sell on for a profit.
This repository contains documentation only. The source code is private. It is a personal, read-only tool for my own use, and the write-up covers what it does, how it is built and what I learned building it.

Why I built it
I flip clothes on Vinted as a side project. Finding deals meant refreshing search pages by hand, and the good ones (a £20 jacket that normally sells for £60) are usually gone within the hour. I wanted something that does the watching for me and only taps me on the shoulder when a listing is worth looking at.
It was also a chance to build something properly end to end: scraping, data handling, a web UI, containers and running it as a service on my homelab.
What it does
·	Checks a set of saved Vinted searches on a slow schedule (Timberland boots, The North Face jackets, Carhartt, Patagonia and so on).
·	Works out whether each new listing is cheap compared with similar listings, estimates the profit after selling costs, and alerts me if it passes my rules.
·	Sends a Discord message with the photo, price, discount, estimated profit, a suggested offer, and anything worth a warning.
·	Marks the very best deals as priority so they stand out.
·	Records what it has seen, so over time it can show which kinds of items actually sell.
How it decides what is a deal
The main idea is that a price only means something next to comparable items.
1.	Baseline price. For each new listing it finds a median price from comparable listings, using the most specific group that has enough data: same item type, same gender (men's or women's) and same size first, widening step by step if there are too few. Rare sizes cost less than common ones, and men's and women's items price differently, so comparing across everything gives misleading answers.
2.	Discount and profit. It compares the asking price with that baseline and subtracts selling costs to get an estimated profit and return.
3.	Rules. A listing alerts only if it clears my minimum discount and minimum profit, in a condition I'd buy.
4.	Priority. The strongest deals are flagged separately, but never if the seller looks risky (for example a brand-new account with no reviews).
Filters that cut down false alarms
Cheap does not mean good. Most of the work was learning what makes a "deal" a dud:
·	Fakes and dupes. Titles and descriptions mentioning replicas, dupes or "not genuine" are dropped, with care taken not to trip on harmless phrases like "fake fur".
·	Damage and problems. Descriptions are read for words like "broken" or "needs repair", while allowing for "no damage" style wording.
·	Mislabelled items. For example, fleeces listed as jackets are skipped so they don't look like bargains against jacket prices.
·	Wrong sizes. A warning is raised when the description contradicts the listed size.
·	Repeats. The same item reposted at the same price is alerted once, not three times.
·	Children's sizes are left out of comparisons so they do not drag the median down.
·	Fabric and line tags. Mentions of things like Gore-Tex, Polartec or HyVent are highlighted on the alert, because they usually mean a better resale.
Architecture
flowchart LR
    V[Vinted public search pages] -->|slow, logged-out reads| M[Monitor]
    M --> F[Filters and comparison]
    F -->|worthwhile deals| D[Discord webhook]
    M --> H[(History and alerts files)]
    H --> P[Control panel - view only]
    P -->|SSH tunnel| Me[My laptop]

·	Monitor. A long-running Python process. It reads public search pages, parses the listings, applies the filters and posts alerts.
·	Storage. Plain files: what it has already seen, a history of listings, alerts, and later the outcome (sold or not) of items it checked again.
·	Control panel. A small web dashboard for performance, prices by size, what sells most, a profit tracker and settings. It only reads the files, so it can't change the monitor when run in view-only mode.
·	Deployment. Both parts run as Docker containers on a Proxmox container in my homelab, so it stays on while my main PC is off. The panel is only reachable through an SSH tunnel, never exposed to the internet.
Being a good citizen
This was a deliberate design choice, not an afterthought:
·	Read-only and logged out. It only reads public search pages. It never logs in, never messages anyone and never buys anything.
·	Slow on purpose. Roughly one search request every few minutes across all searches. Early on I polled too fast and Vinted pushed back with rate-limit responses, so the monitor now backs off and pauses when it sees them, and I keep the schedule conservative.
·	Recovers from failures. If Vinted starts returning empty pages, it resets its session and retries once before giving up.
·	Secrets stay private. The webhook and settings live in a private data folder that is never committed anywhere.
Vinted's terms discourage automated access, which is why this is a private, personal tool and not something I distribute or run at scale.
Things that went wrong (and what I learned)
·	Getting blocked by polling too aggressively taught me to treat the site as a shared resource and build backoff in from the start.
·	The page layout and data changed under me. Parsing HTML is fragile, so I added checks that notice when a page yields no listings and log it clearly instead of failing silently.
·	A monitor that silently stops finding listings looks identical to a quiet market. I added a heartbeat and a "no listings found" warning, and later found that a long-lived web session had gone stale and returned empty pages for hours. A fresh session fixed it, so the monitor now does that automatically.
·	"Cheap" is not "good". My first version flagged lots of fleeces, fakes and damaged items as bargains. Most improvements since have been about removing false positives, not finding more deals.
·	Prices are asking prices, not sale prices. The median is only a guide. Tracking which alerted items actually sold is how I check the tool against reality instead of trusting it.
·	Deploying to Docker on a homelab meant dealing with DNS (Tailscale rewriting resolver settings), file permissions, container restarts on config changes and getting logs out of containers.
Tech stack
·	Python (standard library plus requests)
·	Docker and Docker Compose
·	Proxmox (LXC container) on a MikroTik-networked homelab
·	Discord webhooks for alerts
·	Tailscale and SSH tunnelling for remote access
Limitations
·	It estimates profit from asking prices, so the numbers are a guide, not a promise.
·	Gender and item type are inferred from titles, brands and sizes, so some listings are misclassified.
·	It only sees the newest listings on each search, so very fast-moving searches can still slip a few through.
·	It depends on Vinted's page structure, which can change without warning.
What I'd do next
·	Use the sold-item history to adjust the baseline from real sale data instead of asking prices.
·	Add simple image checks to help catch mislabelled items.
·	Add tests that replay saved pages so a layout change is caught before it silently breaks parsing.

Built by Daniel Scott as a personal project alongside my BSc Computing (Networks, Cyber Security & Forensics) degree.
