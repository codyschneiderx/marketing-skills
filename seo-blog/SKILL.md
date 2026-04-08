---
name: seo-blog
description: Write an SEO blog post targeting a keyword, using SERP research from live Google results and competitor content extraction. Use when the user asks to write/draft a blog post for a target keyword or phrase.
---

# SEO Blog Post

End-to-end process for writing SEO blog posts — SERP research via Serper, content extraction via WebFetch/Exa, write post, optionally publish to a CMS.

When the user gives a target keyword phrase, follow this process. Do NOT publish to any CMS unless the user explicitly asks.

## Process

### Step 1: SERP Research
Search the target keyword via Serper API:
```bash
source .env && curl -s -X POST 'https://google.serper.dev/search' \
  -H "X-API-KEY: $SERPER_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"q": "<keyword>", "gl": "us", "hl": "en"}'
```
Show the user the top 10 results (title, URL, domain).

### Step 2: Extract Content from Top Results
Pull full text from each SERP result using two methods:
- **WebFetch** (try first — free, works for most sites)
- **Exa AI** (fallback for blocked sites like Reddit, Quora, paywalled content)

For Exa, use `verbosity: "full"` and `maxAgeHours: 0` for livecrawl:
```bash
source .env && curl -s -X POST 'https://api.exa.ai/contents' \
  -H "x-api-key: $EXA_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "ids": ["<url>"],
    "text": {"verbosity": "full", "maxCharacters": 50000},
    "livecrawlTimeout": 15000,
    "maxAgeHours": 0
  }'
```

### Step 2.5: Analyze the SERP as a Search-Intent Brief (MANDATORY — DO NOT SKIP)
The SERP is not a list of tool names to mention. It is Google telling you what shape, depth, voice, and content type satisfies the query. The voice is whatever the SERP rewards.

**You MUST fetch ALL top 9-10 organic results in Step 2, not a sample.** Sampling 4-5 produces an incomplete brief. Use WebFetch first, Exa with `livecrawlTimeout` as fallback for blocked sites (Reddit, Quora, paywalled, JS-heavy SPAs).

After fetching, build a brief covering ALL of these dimensions:
- **Word count**: measure each top result. Target the median.
- **POV**: 1st person, 2nd person, or 3rd person authoritative. **Match the dominant POV.** If 8/9 results are 3rd person, write 3rd person.
- **Format**: listicle, how-to, deep-dive guide, definitional authority, comparison, personal experience. Match the dominant format.
- **H2 count and structure**: count headings on each top result. Match or exceed.
- **Stats and benchmarks**: count `[0-9]+%` patterns and citation density. **If the SERP winners are stat-heavy (5+ stats each), stats are MANDATORY.**
- **Recurring sections**: comparison tables, key takeaways blocks, FAQs, vendor landscapes, "why now" framing. If 3+ winners share a section, include it.
- **Vendor positioning**: how does the SERP treat the publisher's own product? Almost always as a vendor case example mid-post, NOT as the framing of the entire post.
- **Gap**: what would a reader still want to know after reading the top 3? That gap is your unique angle within the SERP-matched format.

**How to apply:**
1. Fetch ALL top 9-10 organic results. Use Exa fallback for any failures.
2. Build the brief table covering every dimension above.
3. Write to the brief. POV, length, stats density, recurring sections — match the SERP.
4. Verify with grep/wc before publishing: word count, H2 count, stat count, first-person markers.

### Step 3: Write the Blog Post
- ~1,500 words unless user specifies otherwise
- Target keyword in title and naturally throughout
- **Voice is dictated by the SERP, not by a default.** Read the dominant POV from the Step 2.5 brief and match it. Most marketing/SaaS keywords reward 3rd person authoritative.
- If user specifies a product to feature, place it at #1 with links
- Save as markdown file in the project root: `blog-<slug>.md`
- Present to user for review

#### "vs" Comparison Posts — Featured Product Positioning
For any blog post comparing two tools, include a dedicated section **after the main comparison but before the conclusion/FAQ** featuring the user's product (if specified). Keep it 2-3 paragraphs — persuasive but not spammy. Frame it as "if neither tool feels right, there's a new category."

### Step 4: Publish (only if user asks)
Only publish to a CMS when the user explicitly asks. Generate payload as JSON, POST via curl.

**How to apply:** When user says something like "write a blog post for [keyword]" or "target keyword: [phrase]", run this process. Always stop after Step 3 and show the draft — only publish if explicitly told to.
