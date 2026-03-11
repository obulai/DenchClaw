---
name: lead-sourcing
description: Daily automated pipeline to discover AI solopreneurs and small AI companies across GitHub, X, Reddit, HN, Farcaster, Apollo, and web search.
metadata: { "openclaw": { "inject": true, "always": true, "emoji": "🎯" } }
---

# Daily AI Lead Sourcing Pipeline

Automated daily discovery of two types of leads building with AI:

1. **Solopreneurs / indie hackers** — individuals building AI-powered products → `people` table
2. **Small AI companies** (< 10 employees, recently founded) → `company` table

**Obul relevance filter:** Does this person/company use services that Obul offers? LLMs (OpenRouter), web scraping (Firecrawl, Zyte), enrichment (Apollo, Hunter), search (Tavily, Exa, Jina), social data (X, Reddit), media generation. If they consume paid APIs to build their product, they're a potential Obul user.

## Auth & Base URL

- **API Key**: set as env var `OBUL_API_KEY`
- **Proxy pattern**: `https://proxy.obul.ai/proxy/https/{upstream-host}/{path}`
- All requests go through the Obul proxy — never call upstream APIs directly
- The proxy handles x402 payment negotiation automatically

## Pipeline Steps

```
SEARCH → CLASSIFY → SCORE → DEDUPE → LIGHT ENRICH → STORE → SUMMARY
```

**Estimated daily cost: ~$0.22/day (~$6.60/month)**

---

## 1. Search Query Rotation

Rotate queries by day-of-week to spread coverage and avoid rate limits.

### GitHub (FREE — highest priority)

GitHub searches find both people and companies. Use `gh` CLI.

**Monday — Direct OpenRouter usage:**
```bash
gh search repos "openrouter" --sort=updated --json owner,name,description,url,stargazersCount -L 30
gh search code "OPENROUTER_API_KEY" --filename=.env.example -L 20
```

**Tuesday — Multi-provider routing patterns:**
```bash
gh search code "openrouter" --filename=package.json -L 20
gh search repos "llm router" OR "model router" --created=>YESTERDAY -L 20
```

**Wednesday — AI API users (broad):**
```bash
gh search code "anthropic" "openai" --filename=requirements.txt -L 20
gh search repos "ai agent framework" --stars=1..100 --created=>LAST_WEEK -L 20
```

**Thursday — AI wrapper / tool builders:**
```bash
gh search repos "ai wrapper" OR "ai tool" --stars=1..50 --sort=updated -L 20
gh search code "firecrawl" OR "tavily" OR "exa" --filename=package.json -L 20
```

**Friday — SDK users and API cost signals:**
```bash
gh search repos "openrouter" --language=python --sort=stars -L 20
gh search issues "openrouter" OR "api costs" OR "token costs" -L 20
```

**Saturday/Sunday — Broader AI builder discovery:**
```bash
gh search repos topic:openrouter --sort=updated -L 30
gh search repos topic:llm-routing OR topic:ai-gateway OR topic:ai-agent -L 30
```

### X/Twitter ($0.02/day — 2 searches via obul-twit)

```
"openrouter" OR "ai api costs" OR "model routing" -is:retweet
```
```
"indie hacker" ("ai tool" OR "building with ai" OR "shipped") OR ("ai startup" "launched") -is:retweet
```

### Reddit ($0.04/day — 2 searches via obul-reddit)

Subreddits: r/SideProject, r/indiehackers, r/LocalLLaMA, r/startups

```
openrouter OR "ai api" OR "model routing"
```
```
"ai startup" OR "launched" OR "building with ai"
```

### Hacker News (FREE — Algolia API)

```bash
curl -s "https://hn.algolia.com/api/v1/search_by_date?query=openrouter&tags=story&numericFilters=created_at_i>YESTERDAY_UNIX"
curl -s "https://hn.algolia.com/api/v1/search_by_date?query=ai+startup+launched&tags=show_hn&numericFilters=created_at_i>YESTERDAY_UNIX"
```

Replace `YESTERDAY_UNIX` with `$(date -d 'yesterday' +%s)` or equivalent.

### Farcaster ($0.01/day — 1 search via obul-neynar)

```
"openrouter" OR "ai api" OR "building ai" OR "ai startup"
```

### Firecrawl Web Search ($0.004/day — 2 searches)

```
solopreneur indie hacker openrouter ai tool 2026
```
```
"ai startup" "just launched" OR "building" small team 2026
```

### Apollo ($0.02/day — 2 searches)

**People search:**
```json
{
  "person_titles": ["Founder", "Solo Founder", "Indie Hacker", "Creator"],
  "q_keywords": "AI API LLM openrouter",
  "organization_num_employees_ranges": ["1,10"],
  "per_page": 25
}
```

**Company/org search:**
```json
{
  "q_organization_keyword_tags": ["artificial intelligence", "machine learning", "AI"],
  "organization_num_employees_ranges": ["1,10"],
  "per_page": 25
}
```

---

## 2. Classification Logic

Determine whether each result is a **person** (→ `people` table) or **company** (→ `company` table).

### GitHub signal extraction:

| Signal | Classification |
|--------|---------------|
| Solo repo owner (personal account, no org) | **people** — `Is Solopreneur = Yes` |
| Org repo with < 10 public members | **company** — check with `gh api orgs/{name}` |
| Repo README/description mentions "startup", "company", "team" | **company** |
| Individual contributor with AI repos | **people** |

### Social media (X, Reddit, HN, Farcaster):

| Signal | Classification |
|--------|---------------|
| Individual sharing personal project | **people** — `Is Solopreneur = Yes` |
| Mentions company name, "we", "our team" | **company** |
| Show HN / launched my product (solo) | **people** |

### Apollo:

- People search results → **people** table
- Org search results → **company** table
- If person is founder of org with < 10 employees, create both entries

---

## 3. Scoring Rubric — People

**Threshold: >= 30 to qualify**

| Signal | Points |
|--------|--------|
| Explicitly uses OpenRouter | +30 |
| Building an AI product/tool | +20 |
| Uses multiple AI providers (routing need) | +15 |
| Solo/indie (1 person, no org) | +15 |
| Discusses API costs or routing | +10 |
| Uses services Obul proxies (Firecrawl, Tavily, etc.) | +10 |
| Has public contact info | +5 |
| Active in last 7 days | +5 |

---

## 4. Scoring Rubric — Companies

**Threshold: >= 35 to qualify**

| Signal | Points |
|--------|--------|
| Uses OpenRouter or multiple LLM providers | +30 |
| < 10 employees | +20 |
| Founded in last 2 years | +15 |
| AI-first or AI-enabled product | +15 |
| Uses APIs/services that Obul proxies | +10 |
| Has public website with product info | +5 |
| Active development (recent commits, launches) | +5 |

---

## 5. Deduplication

Before inserting, check existing CRM entries to avoid duplicates.

### People dedup query:
```sql
SELECT entry_id FROM v_people
WHERE "Email Address" = ?
   OR "GitHub Handle" = ?
   OR "Twitter Handle" = ?
   OR ("Full Name" ILIKE ? AND "Company" ILIKE ?);
```

### Company dedup query:
```sql
SELECT entry_id FROM v_company
WHERE "Website" = ?
   OR "GitHub Org" = ?
   OR "Company Name" ILIKE ?;
```

If a match is found, skip insertion. Optionally update the existing entry's `Lead Score` if the new score is higher.

---

## 6. Light Enrichment

Enrich the **top 10 people** and **top 5 companies** per day (~$0.12/day).

### People enrichment chain:

1. **GitHub user** → `gh api users/{username}` for name, email, bio, company, website (FREE)
2. **Twitter user** → profile lookup via obul-twit ($0.005)
3. **If email found** → Apollo person match ($0.01)
4. **If domain found** → Hunter email finder ($0.01)

### Company enrichment chain:

1. **GitHub org** → `gh api orgs/{name}` for description, website, members count (FREE)
2. **If domain** → Apollo org enrich ($0.01) for industry, size, funding, tech stack
3. **If domain** → Minifetch for metadata ($0.002)

Full enrichment happens separately via the existing enrichment skill on user request.

---

## 7. Storage

### People — insert with:

```sql
-- Create entry
INSERT INTO entries (id, object_id) VALUES (nanoid32(), 'seed_obj_people_00000000000000');

-- Set fields via entry_fields
-- Status = "Lead"
-- Lead Source = "{source}" (GitHub, Twitter, Reddit, etc.)
-- Lead Score = "{score}"
-- Source URL = "{url}"
-- Source Signal = "{signal_description}"
-- GitHub Handle = "{handle}"
-- Twitter Handle = "{handle}"
-- Discovered At = "{date}"
-- Enriched = "No" (or "Partial" if light enrichment succeeded)
-- Is Solopreneur = "Yes" / "Unknown"
```

### Companies — insert with:

```sql
-- Create entry
INSERT INTO entries (id, object_id) VALUES (nanoid32(), 'seed_obj_company_0000000000000');

-- Set fields via entry_fields
-- Type = "AI Startup" (or "Prospect" if unclear)
-- Lead Source = "{source}"
-- Lead Score = "{score}"
-- Source URL = "{url}"
-- Source Signal = "{signal_description}"
-- Employee Count = "{count_or_range}"
-- Founded Year = "{year}"
-- Tech Stack = "{technologies}"
-- AI Use Case = "{description}"
-- Obul Relevance = "{which Obul services they could use}"
-- GitHub Org = "{org_name}"
-- Discovered At = "{date}"
-- Enriched = "No" (or "Partial")
```

---

## 8. Summary Report

After each run, output a summary:

```
## Lead Sourcing Summary — {date}

### Sources queried:
- GitHub: {count} results
- X/Twitter: {count} results
- Reddit: {count} results
- HN: {count} results
- Farcaster: {count} results
- Firecrawl: {count} results
- Apollo: {count} results

### Results:
- People scored: {count} | Qualified (>=30): {count} | New (after dedup): {count}
- Companies scored: {count} | Qualified (>=35): {count} | New (after dedup): {count}

### Inserted:
- People: {count} new leads (enriched: {count})
- Companies: {count} new leads (enriched: {count})

### Notable finds:
{top 3-5 highest scoring leads with brief description}

### Cost: ~${total}
```

---

## Budget Cap

**Hard cap: $0.50 per daily run.** If costs approach the cap, skip lower-priority sources (Firecrawl, Farcaster) first.

Priority order (cut from bottom when over budget):
1. GitHub (FREE — always run)
2. HN (FREE — always run)
3. Apollo ($0.02)
4. X/Twitter ($0.02)
5. Reddit ($0.04)
6. Farcaster ($0.01)
7. Firecrawl ($0.004)
