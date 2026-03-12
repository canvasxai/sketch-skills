---
name: icp-discovery
description: >
  ICP (Ideal Customer Profile) discovery and market validation skill. Use this whenever
  a user wants to know who to sell to, find their target customer, validate their market,
  build an outbound list, understand their addressable market, or do competitive research
  to understand who competitors sell to. Triggers on: "who should I sell to", "find my
  ICP", "who would buy this", "help me find leads", "validate my market", "build a prospect
  list", "who are my competitors selling to", or any time a user describes a product/service
  and wants to understand their customer. Always use this skill when the user mentions
  Apollo, outbound sales, market sizing, or competitive ICP research.
---

# ICP Discovery & Market Validation

## What this skill does

This skill helps Claude have a natural, structured conversation to uncover who a user's ideal customer is, then validates and quantifies that profile using two tools: **web scraping** and the **Apollo People Search API**. Web scraping serves two distinct purposes here: (1) researching competitors _before_ the Apollo search to build a smarter ICP, and (2) enriching the sample contacts _after_ the Apollo search to understand who these people actually are and what their companies do. Apollo sizes and samples the market. The goal is a well-targeted ICP in the 1,000–10,000 range — and a clear picture of what those people's worlds look like.

---

## The Conversation Guide

Don't think of this as phases. Think of it as a natural back-and-forth where you're a sharp go-to-market advisor learning about someone's business. Start broad, get specific, use what you learn to shape the next question. Never dump a list of questions — ask one or two at a time, listen carefully, and let the conversation flow.

### Start with the product

The very first thing to understand is what they're selling and at what price. Everything else flows from this. Ask them to describe their product or service — what it does, what problem it solves, and crucially, **what it costs**. Pricing is the single most important filter:

- Under $100/mo → mass market, self-serve, high volume
- $100–$1,000/mo → SMB, some sales motion needed
- $1,000–$10,000/mo → mid-market, needs a champion inside the company
- $10,000+/mo → enterprise, long cycle, multiple stakeholders

If they haven't mentioned pricing, ask. Without it you can't calibrate anything else.

### Understand the customer dimensions

Once you know the product and price, explore these dimensions through conversation. You don't need to ask about all of them — use judgment about what's most relevant:

**Who buys it** — What title actually signs the contract? Who champions it internally? Is there a difference between the person who uses it day-to-day and the person who pays for it? What does that buyer care most about in their role?

**Where they live** — What countries or regions? Does geography matter for this product (language, regulation, time zones)? Are they concentrated in certain cities or distributed globally?

**Why they buy** — What's the trigger event that makes someone urgently need this? Is this a "nice to have" or a "must fix right now" pain? What does their world look like right before they go looking for a solution like this?

**What kind of company** — How big (number of employees)? What industry or vertical? Are they a startup, a mid-size company, an enterprise? What tools and tech do they already use?

**Who to exclude** — Just as important as who to target. Who would _seem_ like a good fit but isn't? Any industries, geographies, or company types that are actually a bad fit?

### Use web scraping to fill in the blanks

When the user is unsure about their ICP — or when you need external signal to validate or challenge what they've said — use the web scraping tool to research competitors and similar products. This is especially useful when:

- The user doesn't know what titles or industries to target
- You want to see who a competitor is publicly selling to (their homepage, case studies, pricing page, "customers" page)
- You need to understand how a market is segmented
- The user mentions a competitor by name and you want to understand that competitor's positioning

**How to use the web scraping tool:**

```
direct-execute-web-scrape
  --url "<target URL>"
  --formats markdown
  --only-main-content true
  --output json
```

Good pages to scrape for ICP signals:

- `[competitor.com]/customers` or `/case-studies` → who they're selling to, company logos, titles in testimonials
- `[competitor.com]/pricing` → what segments they price for (Starter/Pro/Enterprise tiers reveal the company sizes they target)
- `[competitor.com]` homepage → the headline copy almost always says who the product is for
- LinkedIn company pages (if accessible) → job titles in posts, what language they use
- G2, Capterra, or Trustpilot reviews for a competitor → reviewers list their title and company size

Extract from scraped content: job titles mentioned, company sizes referenced, industries highlighted, pain points in testimonials, geographic focus in case studies. Use this to sharpen or validate the ICP the user is building.

If a page can't be scraped (blocked, login-walled, etc.), move on — don't keep trying. Use what you got and proceed.

### Synthesize before searching

Once you've gathered enough signal (from the conversation and/or web scraping), summarize the ICP back to the user clearly before running the Apollo search. Make it feel like you're reflecting back what you heard, not reading from a template:

> "Based on what you've told me — and looking at how [Competitor X] positions their product — it sounds like your ideal customer is a **Head of Operations or VP of Finance** at a **mid-sized SaaS company** (roughly 50–500 employees), probably in the **US or UK**, who's frustrated with manual reconciliation processes. Does that feel right? Anything to adjust before I search?"

Always confirm before running the search.

---

## Apollo People Search API

Once the ICP is confirmed, use Apollo to size the market and pull a sample.

**Endpoint:**

```
POST https://api.apollo.io/api/v1/mixed_people/api_search
```

**Headers:**

```json
{
  "Content-Type": "application/json",
  "Cache-Control": "no-cache",
  "X-Api-Key": "hE9DnJQC8xf2gkBASYpu_A"
}
```

The Apollo API key is hardcoded above. Use it directly in all API calls — do not ask the user for a key.

**Building the request body** — map ICP dimensions to Apollo filters. See `references/apollo-filters.md` for the full parameter reference. Always include these quality defaults:

```json
{
  "contact_email_status": ["verified"],
  "per_page": 10,
  "page": 1
}
```

Use `"include_similar_titles": true` for broad role-based searches. Use `false` only when targeting exact titles.

**Reading the response:**

- `total_entries` → total addressable market size
- `people[]` → sample matches with `first_name`, `title`, `organization.name`
- Last names will be obfuscated (e.g., `Do***e`) on the free tier — this is normal

**Presenting results** — show the market size prominently and a sample table:

> Your ICP matches **4,832 people** in Apollo with verified emails.
>
> | Name          | Title             | Company |
> | ------------- | ----------------- | ------- |
> | James D\*\*\* | VP of Engineering | Stripe  |
> | Sarah M\*\*\* | Head of Ops       | Notion  |

**Market size interpretation:**

| Size             | Signal            | What to do                                           |
| ---------------- | ----------------- | ---------------------------------------------------- |
| < 100            | Too narrow        | Broaden titles, expand geography or company size     |
| 100–999          | Niche             | Fine for hyper-targeted outreach; consider loosening |
| **1,000–10,000** | **Sweet spot ✅** | Right size for a focused outbound motion             |
| 10,000–50,000    | Broad             | Tighten — smaller size range, more specific titles   |
| > 50,000         | Very broad        | Needs a sub-segment; what's the highest-value slice? |

**Iterate** — adjust filters with the user until both the quality of the sample (do these people look right?) and the quantity feel right. Suggest specific changes: "The sample looks good but 42,000 is too many — what if we narrowed to companies with 51–200 employees instead of up to 500?" Re-run as many times as needed.

---

## Enriching Contacts with Web Scraping

Once Apollo returns a sample of matching people, web scraping becomes a research tool — not for competitors, but for the contacts themselves. The goal is to build a rich picture of who these people are, what their companies actually do, and what context they operate in. This is what separates a cold list from a genuinely informed outreach strategy.

**When to do this:** Offer to enrich after a good Apollo result is confirmed. Frame it as: "Want me to dig into a few of these companies so you can see what their world actually looks like before you reach out?"

**What to scrape for each contact:**

_Their company website_ — Scrape the homepage and `/about` page to understand what the company does in their own words, who their customers are, what problems they solve, and how they position themselves. Look for language that reveals their priorities and current growth stage.

_Their company's product or services pages_ — Reveals the depth of their operation, whether they're building vs. buying, and where your product might plug in.

_Their company's jobs page_ — Job listings are a goldmine. If they're hiring a Head of Data or three DevOps engineers, that tells you where they're investing and what pain they're feeling right now. A company hiring aggressively is growing; one with a hiring freeze is cost-cutting.

_Their LinkedIn profile_ (if accessible — not always) — Current role, tenure, past companies, how they describe their own work. Tenure is especially useful: someone who's been in the role 6+ months is settled and likely running into the pain your product solves; someone brand new may be re-evaluating everything.

_Recent news or press_ — Search `[company name] news` or scrape their `/blog` or `/press` page. Funding announcements, new product launches, leadership changes, expansions — any of these can be a trigger event that makes your product timely.

**How to use the web scraping tool:**

```
direct-execute-web-scrape
  --url "<target URL>"
  --formats markdown
  --only-main-content true
  --output json
```

**How to present enrichment findings:** Don't just dump raw content. Synthesize what you learned into a short profile for each company:

> **Acme Corp** (Sarah M\*\*\*, Head of Operations)
> Mid-sized logistics SaaS, ~200 employees, Series B. They're building their own internal tooling based on their jobs page — 4 open backend roles. Their homepage leads with "manual operations slow you down", which maps directly to your pain point. Recently raised $18M (TechCrunch, 3 months ago) — likely in a growth/hiring phase. Good timing.

Do this for 3–5 companies from the Apollo sample. It gives the user a real sense of whether the ICP is landing on the right people, and often sparks refinements ("these are too big" or "I didn't realize they were already solving this themselves").

**If pages are blocked or login-walled:** Skip that source and note it. LinkedIn is often inaccessible; in that case, rely on the company website and news. Don't get stuck trying to force access — move to what's available.

---

## Judgment Calls

**When to use web scraping vs. going straight to Apollo:**
If the user has a clear, confident ICP → go straight to Apollo. If they're unsure, or if you sense their self-reported ICP might be off → scrape competitors first to ground the conversation in evidence before building filters.

**When to enrich contacts after Apollo:**
Always offer this when the Apollo sample looks good and the market size is in range. It's especially valuable when: the user is about to start outreach and wants to personalize, they're unsure if the ICP is truly the right fit, or they want to understand what trigger events or buying signals to look for. Even enriching 3–5 companies can reveal patterns that sharpen the whole strategy.

**When to push back on the user's ICP:**
If their stated ICP would produce fewer than 100 results or more than 50,000, challenge the assumptions before running more searches. Ask what they know about their best existing customers (or the customer they most want) — that's almost always more useful than abstract guessing.

**When to stop asking questions:**
When you have enough to build a reasonable Apollo query. You don't need perfect information. If you can fill in `person_titles`, `organization_num_employees_ranges`, and at least one of `q_organization_keyword_tags` or `organization_locations`, you have enough to run a first search and let the results guide the rest of the conversation.
