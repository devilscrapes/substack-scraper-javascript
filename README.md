<p align="center"><a href="https://apify.com/DevilScrapes/substack-newsletter-scraper"><img src="assets/banner.svg" alt="Substack Newsletter Scraper — JavaScript example" width="100%"></a></p>

# Substack Scraper for JavaScript and Node.js: post feeds for apps, webhooks and backends

<p align="center">
<a href="https://apify.com/DevilScrapes/substack-newsletter-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/substack-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/substack-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository shows how to drive the **[Substack Newsletter Scraper](https://apify.com/DevilScrapes/substack-newsletter-scraper)** from Node.js with the official `apify-client` package. Your service hands over one or more Substack addresses, custom domains included, and receives plain objects describing each post: identifier, slug, canonical link, publish time, bylines, paywall flag, engagement totals and the HTML of free issues. Apify does the paging and the waiting, which keeps a serverless function or a queue worker to a few lines. The included `index.js` also saves the rows to `results.json` and `results.csv` when you run it from a terminal.

> **Actor:** [Substack Newsletter Scraper](https://apify.com/DevilScrapes/substack-newsletter-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [substack-scraper-python](https://github.com/devilscrapes/substack-scraper-python)

## 🔥 What you get

- **Archive walking, newest post first**: the Actor pages through a publication 50 posts at a time until it reaches your cap, your date cutoff or the oldest post. A cap of `0` means keep going to the very first issue.
- **Custom domains accepted as-is**: `https://www.thefp.com` works the same way as a `name.substack.com` address, and a trailing slash in your input is trimmed before any request.
- **Paid or free, decided by one rule**: `is_paywalled` is true whenever the raw `audience` value is anything other than `everyone`, and the raw value ships beside it so you can apply your own reading.
- **Engagement as integers**: `reaction_count` adds up every reaction type Substack records for the post, and `comment_count` is the public comment tally.
- **HTML for free posts only**: `body_html` is filled for posts open to everyone when `includeBody` is on. Gated posts always come back with `null` there.
- **Date windows that save requests**: `postedAfter` stops paging a publication as soon as older posts begin, while `postedBefore` skips newer posts without cutting the walk short.
- **Partial failure stays partial**: if one publication in a batch errors out, it is logged and skipped while the others finish. A run that writes zero posts ends as failed, with a status message that names the cause.

## 🎯 Use cases

### Push new issues to Slack, Discord or any webhook

Run `node index.js` every hour with `postedAfter` set to today, compare `post_id` values with the ones you already sent and post the fresh ones to a webhook URL. The recipe further down implements that loop without a single npm dependency.

### Back a reading app or newsletter aggregator

Store rows keyed by `post_id` in Postgres or SQLite and serve them from an Express or Fastify route. The `is_paywalled` flag lets the UI badge paid issues and link out to Substack instead of rendering a body you do not have.

### Feed an embeddings service from a queue

A worker takes a publication URL from the queue, calls the Actor, then forwards each free post's `body_html` and `url` to your chunking and embedding step. Since the Actor gives `null` for gated posts, the worker can skip them with a single check.

### Watch competitor publications from a serverless timer

A Lambda, Cloud Function or Vercel cron can start a run for the newsletters you track, then store the reaction and comment numbers per post. Re-reading the same posts a week later shows which topics kept drawing replies.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/substack-scraper-javascript.git
cd substack-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Substack Newsletter Scraper](https://apify.com/DevilScrapes/substack-newsletter-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

This is all your backend needs. Awaiting `call()` waits out the whole run; a follow-up `listItems()` on its dataset returns the posts as an array.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/substack-newsletter-scraper').call({"publications": ["https://www.thefp.com"], "maxPostsPerPublication": 10, "includeBody": true, "proxyConfiguration": {"useApifyProxy": true}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` passes `input.json` through unchanged. The sample targets The Free Press, a company-run publication served from its own domain, and limits it to ten posts, so the first call returns fast and costs little. Put more publications in the array, or switch `includeBody` off when your app only needs titles and counts.

```json
{
  "publications": [
    "https://www.thefp.com"
  ],
  "maxPostsPerPublication": 10,
  "includeBody": true,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `publications` | array | yes | `["https://www.lennysnewsletter.com"]` | — |
| `maxPostsPerPublication` | integer | no | `50` | — |
| `includeBody` | boolean | no | `true` | — |
| `postedAfter` | string | no | — | — |
| `postedBefore` | string | no | — | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Every object stands for one post. `post_id` is numeric and stable, which makes it the natural primary key for upserts. `audience` is passed through as Substack reports it, `is_paywalled` is the derived boolean, and nullable fields such as `subtitle`, `cover_image` and `word_count` deserve a fallback in your UI.

| Field | Type | Always present |
|---|---|---|
| `publication` | string | yes |
| `post_id` | integer | yes |
| `title` | string | yes |
| `subtitle` | string or null | no |
| `slug` | string | yes |
| `url` | string | yes |
| `post_date` | string | yes |
| `post_type` | string | yes |
| `authors` | array | yes |
| `audience` | string | yes |
| `is_paywalled` | boolean | yes |
| `reaction_count` | integer | no |
| `comment_count` | integer | no |
| `word_count` | integer or null | no |
| `cover_image` | string or null | no |
| `body_html` | string or null | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.0025 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/substack-newsletter-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This ES module reads `results.json`, keeps a list of already-announced post IDs in `seen.json`, and sends anything new to the URL in `WEBHOOK_URL` (or prints it when that variable is unset). It relies on Node 20 built-ins only, including the global `fetch`.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const posts = JSON.parse(await readFile('results.json', 'utf8'));
const seen = new Set(JSON.parse(await readFile('seen.json', 'utf8').catch(() => '[]')));

const fresh = posts
    .filter((post) => !seen.has(post.post_id))
    .sort((a, b) => a.post_date.localeCompare(b.post_date));

for (const post of fresh) {
    const lock = post.is_paywalled ? ' (paid)' : '';
    const text = `${post.title}${lock} by ${post.authors.join(', ') || 'staff'}: ${post.url}`;
    if (process.env.WEBHOOK_URL) {
        await fetch(process.env.WEBHOOK_URL, {
            method: 'POST',
            headers: { 'content-type': 'application/json' },
            body: JSON.stringify({ text }),
        });
    } else {
        console.log(text);
    }
    seen.add(post.post_id);
}

await writeFile('seen.json', JSON.stringify([...seen]));
console.log(`${fresh.length} new post(s) announced`);
```

## ❓ FAQ

### Does Substack offer an official API I could call instead?

Substack does not publish a documented public API for reading other people's newsletters. This Actor reads the post listing that every publication's own website uses, and your code only needs an Apify token to call it.

### Can I use this from TypeScript?

Yes. `apify-client` includes type definitions, so the run object is typed. For the posts themselves, declare a small interface with the fields you use, such as `post_id: number` and `is_paywalled: boolean`.

### What happens if one of several publication URLs is wrong?

That publication is logged and skipped, and the rest still return rows. Only when every publication fails, or none has posts in your window, does the run end as failed; in that case `index.js` exits with code 1 and prints the reason.

### How do I keep test calls cheap?

Hold `maxPostsPerPublication` at a handful and start the script as `node index.js --max-charge 0.5`; the run then cannot bill past fifty cents, whatever the input says.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/substack-newsletter-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Substack Newsletter Scraper on Apify Store](https://apify.com/DevilScrapes/substack-newsletter-scraper)
- [Python version of this example](https://github.com/devilscrapes/substack-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [DEV.to Scraper — Articles by Tag & Author](https://apify.com/DevilScrapes/dev-to-articles-scraper) — developer articles from DEV by tag or author, for a second blogging platform.
- [Google News Scraper — Search & Topic RSS Feeds](https://apify.com/DevilScrapes/google-news-scraper) — news headlines by query or topic, to see what mainstream outlets cover beside a newsletter.
- [Patreon Creator Scraper](https://apify.com/DevilScrapes/patreon-creator-scraper) — creator pages on Patreon, the other big paid-membership platform.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/substack-newsletter-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Substack Inc..
