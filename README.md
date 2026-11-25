# LLM Web Scraping: Agent, Library, or Fetch API? How to Choose, What Each Costs, and Where ScraperAPI Fits

Three routes work for LLM web scraping today: a general GUI agent that drives a real browser, a developer library that turns a page into schema-valid JSON, and a Python pipeline that pairs stealth fetching with schema validation. Use the agent when data only appears after clicks or a login, and use a library or pipeline when your URLs already return usable HTML and you need the same fields every day. Underneath all three sits the part nobody demos: getting the page at all. That layer is what ScraperAPI sells, and it is where most production budgets actually go.

> **The short version.** If you already know your target URLs, split the job. Let a scraping API fetch pages and let an LLM extract fields from the HTML. ScraperAPI's free plan gives you 1,000 API credits a month at 5 concurrent connections, and the first 7 days after signup raise that to 5,000 credits, which is enough to measure what your own targets cost before you pay anything. 👉 [Start on ScraperAPI's free plan and see the current plan pricing](https://www.scraperapi.com/?fp_ref=coupons)

## The split that makes the rest of this readable: fetch layer vs understanding layer

Every LLM scraping stack has two layers, whether or not the person who built it names them. The fetch layer takes a URL and returns something a model can read: rendered HTML, cleaned text, Markdown, or a screenshot. The understanding layer takes that payload plus a schema and returns JSON. Tools that look like competitors usually sit on different layers, which is why comparisons between them tend to talk past each other.

ScraperAPI is a fetch-layer product. It routes requests through rotating proxy pools with JavaScript rendering and anti-bot handling, and it returns HTML or, with Autoparse, parsed JSON. It has no opinion about what a "product" or an "article" is on your site. That definition stays with you, in a schema or a prompt.

### What a request actually costs

Credit cost varies by domain rather than sitting flat at one per URL. These are ScraperAPI's published multipliers:

| Target type | Credits per request | Examples |
| --- | --- | --- |
| Standard page | 1 | Most public websites |
| Amazon | 5 | amazon.com and all Amazon domains |
| Google and Bing, including subdomains | 25 | google.com, bing.com |
| LinkedIn | 30 | linkedin.com |
| Anti-bot bypass surcharge | +10 on top of the base cost | Pages behind Cloudflare, DataDome or PerimeterX, charged when bypass logic activates |

Run the free allowance through those numbers and you get a realistic test budget: 1,000 standard pages, 200 Amazon pages, 40 Google result pages, or roughly 90 requests against a Cloudflare-protected site once the +10 surcharge applies. The Domain Cost Estimator inside the dashboard is the authoritative figure for one specific URL, and the `max_cost` parameter caps what a single scrape is allowed to spend.

### What happens when you run out

This is the part people discover too late, so it belongs next to the pricing question rather than in a footnote.

| Tier | Auto-upgrade when credits run out | Pay-as-you-go past the monthly cap |
| --- | --- | --- |
| Hobby | Yes | No |
| Startup | Yes | No |
| Business | Yes | No |
| Scaling | Yes | Yes |
| Professional | Yes | Yes |
| Advanced | No | Yes |
| Enterprise | No | Yes |

Without one of those two options switched on, scraping pauses until credits renew. Exceeding your concurrency limit returns a 429 and the request is turned away rather than queued, and you are not charged for it. Credits reset at each renewal date and unused ones do not carry forward. Annual billing is exactly 10% below monthly and hands you the full year's credits upfront, so you can spend them on any schedule, and there is a 7-day no-questions-asked refund window if the credit maths turns out wrong for your targets. 👉 [Check ScraperAPI's plans against your own target list](https://www.scraperapi.com/?fp_ref=coupons)

## Route 1: a general agent that drives the browser

Webscraper, an arXiv preprint from Guan-Lun Huang and Yuh-Jzer Joung at National Taiwan University dated 31 March 2026, targets what the authors call index-and-content architecture. An index page lists entries with links; each link leads to a content page. News section fronts, search results, video lists and social feeds all follow this shape, which is why the paper treats it as the general case rather than one layout among many.

The task is defined by a single natural-language prompt that names the site, the scope and the fields, and the deliverable is one JSON file with an object per extracted item. The paper's own example asks for the first two pages of a BBC section, extracting title, link and content for each story. Two stages sit under that prompt: web navigation, where the agent interacts with the interface until content is surfaced, and web information extraction, where the surfaced content is parsed out of semi-structured HTML.

The agent is built on Anthropic's Computer Use and runs an Observe-Reason-Act loop with three native tools. Computer handles screenshots, mouse movement, clicks and scrolling. Bash handles filesystem work, scripts and curl. Str Editor writes, reads and revises files programmatically, which is how the agent generates and iterates on extraction scripts. Two custom tools carry the scraping-specific work:

- **Parse Tool** sends the HTML plus the user's requirements to GPT-o3, which writes a task-specific Python script that runs in a code interpreter and returns structured data. The paper gives three reasons for outsourcing the parsing instead of having the agent write it inline: long code-generation transcripts stay out of the main agent's context window, the hardest page (the index page) goes to a stronger reasoning model, and the system prompt stays short enough that the agent spends its attention on strategy rather than parser syntax.
- **Merge Tool** aggregates the structured lists produced across rounds, deduplicating and merging them, which matters as soon as pagination starts returning overlapping items.

A structured five-stage prompting procedure wraps the orchestration. The available text names the procedure without enumerating the stages, so treat it as a design pattern to imitate rather than a recipe to copy.

On results: evaluation covered six news websites, and the full framework, with the guiding prompt and both custom tools, beat the Anthropic Computer Use baseline on extraction accuracy by a margin the authors describe as significant. The framework was also run against e-commerce sites to check that it generalises beyond news. The specific accuracy figures sit in a section that is not available here, so the defensible claim is "better than the baseline agent," not a percentage. Note also that Computer Use appears in this paper as the thing being improved on, not as a recommended product.

## Route 2: a TypeScript library where the model reads the page

LLM Scraper is a TypeScript library, now at version 2.0 with Vercel AI SDK 6 support, that runs Playwright and hands page content to a model alongside a Zod or JSON Schema definition. It supports GPT, Sonnet, Gemini, Llama and Qwen series models, keeps types end to end, and streams partial objects.

The decision that matters most is which format you feed the model, because it sets both token cost and failure modes:

| `format` | What the model receives |
| --- | --- |
| `html` | Pre-processed HTML |
| `raw_html` | Raw HTML, no processing |
| `markdown` | Markdown |
| `text` | Body text extracted with Mozilla's Readability.js |
| `image` | A screenshot, multimodal models only |
| `custom` | Whatever your own function returns |

`html` versus `raw_html` is the difference between paying for navigation chrome and script noise, and paying for it twice. `text` throws away structure entirely, which is fine for prose articles and bad for anything where a price sits in a table cell next to a product name. `image` only works with multimodal models and adds a vision pass to every page, so it suits layouts that resist parsing rather than bulk runs. `custom` lets you pre-clean, pre-filter or pull content from a cache before the model ever sees it.

The most underused call in the library is `generate()`. Instead of `run()`, which invokes the model on every page, `generate()` returns a reusable Playwright script; you execute it with `page.evaluate(code)` and validate the result with `schema.parse(result)`. One LLM call, then zero token spend per page until the site's markup changes. That is the practical answer for a target you scrape daily.

## Route 3: a Python pipeline with stealth, chunking and schema enforcement

llm-structured-scraper is a selector-free pipeline built for the failure mode that kills CSS-selector scrapers: a redesign that moves one div and silently returns empty rows. It swaps DOM matching for semantic extraction against a Pydantic schema, with Instructor enforcing the schema at the API boundary so every response comes back as valid JSON.

The fetching half is where the anti-detection work lives. A `StealthPlaywrightScraper` class rotates user agents, screen resolutions and timezones, strips webdriver flags, injects Chrome runtime objects, randomises fingerprints, integrates playwright-stealth, and supports proxy configuration for IP rotation and geo-targeting. Semaphore-based concurrency control and exponential-backoff retries handle the operational side.

The processing half is a reasonable default for anyone rolling their own:

- HTML is converted to Markdown with scripts, styles, navigation and ads stripped first, which cuts tokens before the model sees anything.
- `tiktoken` splits the Markdown at line boundaries with **1,500-token chunks for local models and 6,000 for remote ones**; a single overlong line gets its own chunk rather than being cut mid-sentence.
- Temperature is pinned at **0.0** to suppress invented values, and every chunk reports prompt, completion and total token counts so cost tracking is per-run rather than per-month guesswork.
- Wait conditions cover `load`, `domcontentloaded`, `networkidle` (the default) and custom selectors, with a 30,000 ms timeout in the example config.
- Groq or Ollama is selected with a single `is_local` switch, trading cloud speed and per-token billing for free local inference on your own hardware.

## When an agent is the wrong tool

A benchmark covering **35 sites across five security tiers**, including authentication, anti-bot and CAPTCHA controls, compared two workflows: LLM-assisted scripting, where the model writes conventional scraping code and the human runs it, and end-to-end agents that navigate and extract on their own. The headline finding is that agents made complex scraping accessible, needing as little as one prompt and fewer than five rounds of refinement to finish a workflow that would otherwise require specialist skill.

The same benchmark is blunt about the opposite case. On static sites with stable, predictable structure and content delivered in plain HTML, an agent is overkill, and scripted extraction is simpler and faster. Traditional tools also stay cheaper for large batch runs where runtime overhead matters. Headless browsers extend reach into client-rendered pages, but integration complexity and execution time climb with them.

Agent downsides worth budgeting for: execution time is often higher, reasoning is non-deterministic so outputs vary between runs, and success rates drop noticeably against harder or obfuscated CAPTCHAs. LLM-based CAPTCHA solving exists but adds latency. If your target is a static product catalogue, none of that complexity buys you anything.

## The two costs that stack, and how to stop paying one of them

Fetching is billed per request, in credits, with a multiplier per domain. Extraction is billed per token, and the pipeline defaults above show why a page is not one unit of work: a 6,000-token chunk holds a decent listing page, but a long article or a spec-heavy product page can spill into several chunks and several model calls. A page with 40 products might fit one chunk; the same page with reviews expanded might take three.

The paper behind Webscraper puts it plainly: LLM-based extraction reaches high accuracy, and its computational cost is a real obstacle at scale. That is the pressure behind the generate-once approach represented by AutoScraper's XPath generation, which is cheap but aimed at locating individual data points rather than extracting a whole page of structured records.

The configuration that usually wins on a stable target combines both layers: fetch through an API that handles proxies and anti-bot, spend one LLM pass turning the page into a script or a schema-bound parser, then re-run that script on a schedule with the model out of the loop. You keep the LLM for the two moments it earns its cost, first-time schema discovery and re-parsing after a redesign, and pay only credits in between.

## Before you point this at 10,000 URLs

The failure modes are predictable, and so are the fixes. From the pipeline's own troubleshooting guide:

- **403s and bot detection.** Confirm stealth is enabled, add proxies, widen the delay between requests, and run with `headless=False` once to see what the site is actually serving.
- **Empty CSV or missing rows.** Check the intermediate Markdown first. If it has content, tighten the system prompt, increase chunk size, or verify the page genuinely contains the data you expect.
- **TimeoutError on navigation.** Switch from `networkidle` to `load`, raise the timeout, or wait for a specific selector instead of the whole page.
- **Invalid JSON or schema validation errors.** Add examples to the prompt, set `max_retries=3` for production runs, or change model. Smaller models fail schema conformance long before they fail comprehension.
- **Memory climbing.** Fewer concurrent browsers and smaller chunks, not a bigger machine.

On top of those, expect non-deterministic output from any agent route, which is why schema validation belongs at the boundary rather than in a later cleaning step.

## Compliance, briefly

There is now a benchmark specifically evaluating the legal compliance of AI agent web scraping (ScrapeBench, by Joseph Marvin Imperial, Daniel Slate and Noam Kolt). Its published conclusions are not available here, so the only fair statement is that this dimension is being measured now, and the answer for your own targets depends on their terms of service and your jurisdiction.

## FAQ

**Does an LLM scraper return accurate data?**
The Webscraper paper reports accuracy significantly above its Anthropic Computer Use baseline across six news sites, and the pipeline community's standard defence against invented values is temperature 0.0 plus hard schema validation. Neither gives you a guarantee per field; sampling and diffing against a manual read is still how you catch systematic errors.

**Do I need an LLM at all for a static site that returns HTML?**
No. The 35-site benchmark found agents to be overkill on static, predictable pages, where scripted extraction is faster and simpler. Reach for an LLM when the layout varies, the content loads after interaction, or writing selectors has become a maintenance job.

**Can these tools get past CAPTCHAs?**
Partly. Success rates fall sharply as CAPTCHA difficulty and obfuscation increase, and LLM solving introduces latency. On the fetch side, ScraperAPI includes CAPTCHA and anti-bot handling in every plan, with a +10 credit surcharge applied when bypass logic activates against Cloudflare, DataDome or PerimeterX.

**How do I keep costs from running away?**
Two levers. On the fetch side, cap spend per request with `max_cost` and check the Domain Cost Estimator before a large run, because a Google page costs 25 times a standard page. On the extraction side, log tokens per chunk, and once a page's schema is stable, switch to a generated script so the model stops being called.

**What if I buy a plan and it is wrong for my volume?**
ScraperAPI runs a 7-day no-questions-asked refund policy, and the free plan plus the 7-day 5,000-credit trial let you size your real usage before committing. 👉 [Compare the plans and start with the free tier](https://www.scraperapi.com/?fp_ref=coupons)

## What to pick

Static HTML, stable structure, big batches: a traditional script wins, and adding an agent only adds runtime and variance.

Pages that need a login, several clicks, or a fight with anti-bot protection, where you need the data once or occasionally: an end-to-end agent, accepting that output varies between runs and CAPTCHAs get harder.

The same fields on the same sites, week after week: fetch through an API, define a schema, and use one LLM pass to produce a reusable script. Check the credit multiplier for your specific domains first, since that number, not the plan name, decides your monthly bill, and confirm the concurrency limit against how many pages you need in parallel before you commit to a tier.

👉 [Run your own URLs through ScraperAPI's free plan](https://www.scraperapi.com/?fp_ref=coupons)
