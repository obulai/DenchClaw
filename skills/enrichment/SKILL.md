---
name: lead-enrichment
description: Find leads, enrich contacts/companies, discover emails, and scrape data via Obul proxy APIs (Apollo, Hunter, StableEnrich, Firecrawl).
metadata: { "openclaw": { "inject": true, "always": true, "emoji": "🔍" } }
---

# Lead Finding & Enrichment via Obul APIs

You have access to enrichment APIs through the **Obul proxy gateway**. This gives you pay-per-use access to Apollo, Hunter, StableEnrich, and Firecrawl — all through a single API key.

## Auth & Base URL

- **API Key**: `{{OBUL_API_KEY}}` (set as env var `OBUL_API_KEY`)
- **Proxy pattern**: `https://proxy.obul.ai/proxy/https/{upstream-host}/{path}`
- All requests go through the Obul proxy — never call upstream APIs directly
- The proxy handles x402 payment negotiation automatically

## Quick Reference

| Task | API | Cost | Endpoint |
|------|-----|------|----------|
| Search people | Apollo | $0.01/req | `/proxy/https/api.apollo.io/api/v1/mixed_people/search` |
| Enrich person | Apollo | $0.01/req | `/proxy/https/api.apollo.io/api/v1/people/match` |
| Enrich company | Apollo | $0.01/req | `/proxy/https/api.apollo.io/api/v1/organizations/enrich` |
| Bulk enrich | Apollo | $0.05/batch | `/proxy/https/api.apollo.io/api/v1/people/bulk_match` |
| Find email | Hunter | $0.01/req | `/proxy/https/api.hunter.io/v2/email-finder` |
| Verify email | Hunter | $0.01/req | `/proxy/https/api.hunter.io/v2/email-verifier` |
| Domain search | Hunter | $0.01/req | `/proxy/https/api.hunter.io/v2/domain-search` |
| LinkedIn scrape | StableEnrich | $0.02/req | `/proxy/https/stableenrich.com/api/v1/linkedin/profile` |
| Web scrape | Firecrawl | $0.001/req | `/proxy/https/api.firecrawl.dev/v1/scrape` |

---

## API Reference

### 1. Apollo — People Search & Enrichment

Apollo has 270M+ contacts. Use it as the **primary enrichment source**.

#### Search People

Find leads matching filters (title, company, location, industry).

```bash
curl -s -X POST "https://proxy.obul.ai/proxy/https/api.apollo.io/api/v1/mixed_people/search" \
  -H "x-api-key: $OBUL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "person_titles": ["CTO", "VP Engineering"],
    "organization_locations": ["United States"],
    "organization_num_employees_ranges": ["51,200"],
    "per_page": 25,
    "page": 1
  }'
```

Key filters: `person_titles`, `person_locations`, `organization_locations`, `organization_domains`, `organization_num_employees_ranges` (e.g. `"1,10"`, `"11,50"`, `"51,200"`, `"201,500"`, `"501,1000"`, `"1001,5000"`, `"5001,10000"`), `q_keywords`, `person_seniorities` (e.g. `"c_suite"`, `"vp"`, `"director"`, `"manager"`).

Response includes: `people[]` with `first_name`, `last_name`, `title`, `organization.name`, `email`, `linkedin_url`, `city`, `state`, `country`.

#### Enrich Person (by email or LinkedIn)

```bash
curl -s -X POST "https://proxy.obul.ai/proxy/https/api.apollo.io/api/v1/people/match" \
  -H "x-api-key: $OBUL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com"
  }'
```

Can also match by: `linkedin_url`, `first_name` + `last_name` + `organization_name`.

Response: `person` object with full profile — `title`, `headline`, `organization` (name, website, industry, size), `phone_numbers`, `email`, `linkedin_url`, `city`, `state`, `country`, `departments`, `seniority`.

#### Enrich Company (by domain)

```bash
curl -s -X POST "https://proxy.obul.ai/proxy/https/api.apollo.io/api/v1/organizations/enrich" \
  -H "x-api-key: $OBUL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "domain": "example.com"
  }'
```

Response: `organization` with `name`, `website_url`, `linkedin_url`, `founded_year`, `industry`, `estimated_num_employees`, `total_funding`, `latest_funding_round_date`, `technologies`, `city`, `state`, `country`, `short_description`.

#### Bulk Enrich (10+ people, $0.05/batch)

Use for batch operations — up to 100 people per request.

```bash
curl -s -X POST "https://proxy.obul.ai/proxy/https/api.apollo.io/api/v1/people/bulk_match" \
  -H "x-api-key: $OBUL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "details": [
      {"email": "person1@example.com"},
      {"email": "person2@example.com"},
      {"linkedin_url": "https://linkedin.com/in/person3"}
    ]
  }'
```

Response: `matches[]` array with full person objects.

---

### 2. Hunter — Email Finding & Verification

Hunter specializes in email operations. Use it for email discovery and deliverability checks.

#### Find Email (name + domain)

```bash
curl -s "https://proxy.obul.ai/proxy/https/api.hunter.io/v2/email-finder?domain=example.com&first_name=John&last_name=Doe" \
  -H "x-api-key: $OBUL_API_KEY"
```

Response: `data.email`, `data.score` (confidence 0-100), `data.position`, `data.company`.

#### Verify Email

```bash
curl -s "https://proxy.obul.ai/proxy/https/api.hunter.io/v2/email-verifier?email=user@example.com" \
  -H "x-api-key: $OBUL_API_KEY"
```

Response: `data.result` (`deliverable`, `undeliverable`, `risky`, `unknown`), `data.score`, `data.regexp`, `data.mx_records`.

#### Domain Search (all emails at a company)

```bash
curl -s "https://proxy.obul.ai/proxy/https/api.hunter.io/v2/domain-search?domain=example.com&limit=20" \
  -H "x-api-key: $OBUL_API_KEY"
```

Response: `data.emails[]` with `value`, `type` (personal/generic), `confidence`, `first_name`, `last_name`, `position`.

---

### 3. StableEnrich — LinkedIn Scraping

Use when you need LinkedIn profile data that Apollo doesn't have.

#### Scrape LinkedIn Profile

```bash
curl -s -X POST "https://proxy.obul.ai/proxy/https/stableenrich.com/api/v1/linkedin/profile" \
  -H "x-api-key: $OBUL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "linkedin_url": "https://www.linkedin.com/in/username"
  }'
```

Response: Full LinkedIn profile — `headline`, `summary`, `experience[]`, `education[]`, `skills[]`, `connections_count`, `location`.

---

### 4. Firecrawl — Web Scraping

Ultra-cheap fallback for scraping any URL to markdown. Use for company websites, blog posts, team pages, etc.

#### Scrape URL to Markdown

```bash
curl -s -X POST "https://proxy.obul.ai/proxy/https/api.firecrawl.dev/v1/scrape" \
  -H "x-api-key: $OBUL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/about",
    "formats": ["markdown"]
  }'
```

Response: `data.markdown` — full page content as markdown.

---

## Workflows

### Find Leads

When the user asks to find leads matching criteria:

1. **Search** with Apollo `mixed_people/search` using their filters (title, location, company size, etc.)
2. **Store** results in the CRM `people` table (see CRM Integration below)
3. **Report** count found, sample results, and estimated cost

### Enrich Existing Contacts

When the user has contacts in their CRM that need enrichment:

1. **Read** contacts from DuckDB that have missing fields
2. **Enrich** each via Apollo `people/match` (by email or LinkedIn URL)
   - Use **bulk match** if 10+ contacts ($0.05 vs $0.01 each = cheaper at 6+)
3. **Update** CRM records with enriched data
4. **Report** fields filled, success rate, cost

### Email Discovery

When the user needs email addresses:

1. **Try Hunter email-finder** first (name + domain → email, $0.01)
2. **Verify** the found email with Hunter email-verifier ($0.01)
3. **Fallback**: If Hunter misses, try Apollo person match which often includes email
4. **Store** verified emails in CRM with verification status

### Company Research

When the user wants company intelligence:

1. **Apollo org enrich** by domain → structured company data ($0.01)
2. **Firecrawl scrape** company website for additional context ($0.001)
3. **Hunter domain search** to find key contacts ($0.01)
4. **Store** in CRM `companies` table

---

## CRM Integration

Map API responses to DuckDB fields. Always check existing schema first with:
```sql
DESCRIBE people;
DESCRIBE companies;
```

### People Field Mapping

| API Field | CRM Field | Source |
|-----------|-----------|--------|
| `first_name` | `first_name` | Apollo/Hunter |
| `last_name` | `last_name` | Apollo/Hunter |
| `email` | `email` | Apollo/Hunter |
| `title` | `job_title` | Apollo |
| `organization.name` | `company` | Apollo |
| `linkedin_url` | `linkedin_url` | Apollo/StableEnrich |
| `phone_numbers[0].sanitized_number` | `phone` | Apollo |
| `city`, `state`, `country` | `location` | Apollo |
| Hunter `data.result` | `email_verified` | Hunter |
| `headline` | `headline` | Apollo/StableEnrich |
| `seniority` | `seniority` | Apollo |

### Companies Field Mapping

| API Field | CRM Field | Source |
|-----------|-----------|--------|
| `name` | `name` | Apollo |
| `website_url` | `website` | Apollo |
| `industry` | `industry` | Apollo |
| `estimated_num_employees` | `employee_count` | Apollo |
| `total_funding` | `funding` | Apollo |
| `short_description` | `description` | Apollo |
| `linkedin_url` | `linkedin_url` | Apollo |
| `founded_year` | `founded` | Apollo |
| `technologies` | `tech_stack` | Apollo |

If CRM fields don't exist yet, create them with appropriate types before inserting data.

---

## Cost Guidelines

- **Always prefer the cheapest API** that can fulfil the request
- **Log estimated costs** when running batch operations: `echo "Estimated cost: N requests × $X = $Y"`
- **Use bulk endpoints** when processing 6+ records (Apollo bulk match = $0.05 for up to 100)
- **Firecrawl first** for simple web data ($0.001) before reaching for enrichment APIs ($0.01+)
- **Cache results**: Before calling an API, check if the CRM already has the data
- **Confirm with user** before operations estimated to cost > $1.00

### Typical Cost Examples

| Operation | Records | Est. Cost |
|-----------|---------|-----------|
| Find 25 leads | 1 search | $0.01 |
| Enrich 10 people | 1 bulk | $0.05 |
| Enrich 100 people | 1 bulk | $0.05 |
| Find + verify 10 emails | 20 requests | $0.20 |
| Full company research | 3 requests | $0.021 |
| Scrape 50 web pages | 50 requests | $0.05 |
