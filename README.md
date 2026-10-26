<p align="center"><a href="https://apify.com/DevilScrapes/hacker-news-scraper"><img src="assets/banner.svg" alt="Hacker News Scraper — Python example" width="100%"></a></p>

# Hacker News Scraper in Python: front-page and Show HN snapshots for pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/hacker-news-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/hacker-news-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/hacker-news-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Hacker News publishes its data openly, but a feed endpoint hands you nothing more than a list of up to 500 numeric IDs; titles, points and comment counts each sit behind a separate request. This repository is a short Python client for the **[Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper)** on Apify, which does that stitching and returns one finished row per story. Choose the front page, the newest submissions, the best-of list, Ask HN, Show HN or the jobs board, and each row comes back with its position in that feed, title, link, author handle, points, comment total and posting time in UTC. Run it once for a quick look or every half hour from cron to build a history of what reached the front page and how fast it climbed. Results land in `results.json` and `results.csv`, ready for a DataFrame.

> **Actor:** [Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [hacker-news-scraper-javascript](https://github.com/devilscrapes/hacker-news-scraper-javascript)

## 🔥 What you get

- **Six feeds behind one switch**: `feed` picks the front page (`top`), the newest submissions, the best-of ranking, Ask HN, Show HN or the YC jobs board.
- **Front-page position captured**: every row carries `rank`, the story's place in the chosen feed at the moment of the run, a number the official item records do not include.
- **Feed and items stitched together**: the feed only lists IDs, so the Actor fetches each item record in parallel (`concurrency`, 1 to 32) and returns finished rows.
- **Two time formats**: the raw Unix `time` the site stores plus an ISO 8601 UTC `posted_at`, and a `scraped_at` stamp so repeated snapshots can be ordered.
- **A ready discussion link**: `permalink` points at the item's `news.ycombinator.com` thread, while `url` holds the outbound article and is `null` for text posts.
- **Self-post bodies on request**: with `includeText` on, Ask HN and Show HN rows include the post body as the site's HTML.
- **Dead and deleted items dropped**: flagged or removed stories never reach your dataset, and an item that fails to load is logged and left out instead of stopping the run.

## 🎯 Use cases

### Measure what climbs the front page

Snapshot the `top` feed every 30 minutes and append each CSV to a growing table. Because every row has `rank`, `score` and `scraped_at`, a pivot on `id` shows the path each story took through the ranking, how long it stayed in the top ten and how many points it had when it dropped off.

### Find which domains Hacker News rewards

Parse the hostname out of `url` and aggregate points and comments per domain over a few weeks of snapshots. Content marketers and developer-relations teams use this kind of table to see whether long-form blogs, GitHub repositories or news outlets draw more discussion.

### Study Show HN launches

Point the Actor at the `show` feed daily and keep every row. Over a month you have a dataset of launches with their points and comment counts, enough to compare weekday against weekend posting or to check whether self-hosted tools outperform SaaS announcements.

### Label stories for a topic model

Titles are short, dense and written by practitioners, which makes them good training text. Collect the `best` feed over time, deduplicate on `id` and you have a growing corpus of headlines with an engagement score attached to each.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/hacker-news-scraper-python.git
cd hacker-news-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

With the CLI parsing and file writing removed, what remains is the snippet below. It launches the Actor, blocks until the run is over and walks the resulting dataset one story at a time.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/hacker-news-scraper").call(run_input={'feed': 'top', 'maxResults': 10, 'includeText': True, 'concurrency': 8, 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` hands `input.json` to the Actor exactly as written. The bundled file grabs the first ten front-page stories with their text bodies, which makes a cheap and quick first run. Change `feed` to `show`, `ask` or `jobs` for the dedicated lists, or raise `maxResults` toward 500; `0` means the whole feed.

```json
{
  "feed": "top",
  "maxResults": 10,
  "includeText": true,
  "concurrency": 8,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `feed` | string | no | `"top"` | `top`, `new`, `best`, `ask`, `show`, `jobs` |
| `maxResults` | integer | no | `100` | — |
| `includeText` | boolean | no | `true` | — |
| `concurrency` | integer | no | `8` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Each row stands for one item from the chosen feed. Rows may arrive in a different order from the ranking because items are fetched in parallel, so sort by `rank` when position matters. Job posts may have no score, and link posts have no `text`.

| Field | Type | Always present |
|---|---|---|
| `id` | integer | yes |
| `type` | string | no |
| `title` | string | yes |
| `url` | string or null | no |
| `permalink` | string | yes |
| `by` | string | yes |
| `score` | integer or null | no |
| `descendants` | integer or null | no |
| `text` | string or null | no |
| `time` | integer | yes |
| `posted_at` | string | no |
| `scraped_at` | string | no |
| `rank` | integer | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.002 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/hacker-news-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

The pandas sketch below (`pip install pandas`) works out points per hour since posting, extracts each story's domain and prints the domains that collected the most points in the snapshot.

```python
import json
from urllib.parse import urlparse

import pandas as pd

stories = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
posted = pd.to_datetime(stories["posted_at"], utc=True, errors="coerce")
seen = pd.to_datetime(stories["scraped_at"], utc=True, errors="coerce")
hours = ((seen - posted).dt.total_seconds() / 3600).clip(lower=0.5)
stories["points_per_hour"] = (stories["score"].fillna(0) / hours).round(1)


def domain(link):
    return urlparse(link).netloc.removeprefix("www.") if isinstance(link, str) else "(text post)"


stories["domain"] = stories["url"].map(domain)
ranked = stories.sort_values("rank")
print(ranked[["rank", "title", "score", "descendants", "points_per_hour"]].to_string(index=False))

by_domain = stories.groupby("domain")["score"].agg(["count", "sum"]).sort_values("sum", ascending=False)
print(by_domain.head(10).to_string())
ranked.to_csv("front_page_snapshot.csv", index=False)
```

## ❓ FAQ

### Why not query the Hacker News API directly?

You can, and for a few items it is fine. A full feed, though, means one request for the ID list and up to 500 more for the items, plus your own concurrency control, timestamp conversion and rank bookkeeping. The Actor does that work and hands a notebook a flat table, which is the shape pandas wants.

### Can I get the comments of a story?

No. Each row reports the comment total in `descendants`, but the comment tree itself is not collected. Use `permalink` to open the thread when you need the discussion.

### Does it search Hacker News by keyword?

Not on the server side. The Actor reads the six ranked feeds. To track a keyword, run the `new` feed on a schedule and filter titles in pandas with `str.contains`, since fresh submissions show up there first.

### How do I build a history from repeated runs?

Run `python main.py --out snapshots/$(date +%Y%m%dT%H%M)` from cron and concatenate the CSVs later. `id` identifies a story across snapshots, and `scraped_at` orders them.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/hacker-news-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Hacker News Scraper on Apify Store](https://apify.com/DevilScrapes/hacker-news-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/hacker-news-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [DEV.to Scraper — Articles by Tag & Author](https://apify.com/DevilScrapes/dev-to-articles-scraper) — developer blog posts with tags and reactions from DEV Community.
- [Lemmy Scraper — Posts, Comments & Community Data](https://apify.com/DevilScrapes/lemmy-community-scraper) — posts and votes from federated Lemmy communities.
- [GitHub Repo Scraper](https://apify.com/DevilScrapes/github-repo-scraper) — stars, forks and topics for the repositories that Show HN posts link to.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/hacker-news-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Hacker News or Y Combinator.
