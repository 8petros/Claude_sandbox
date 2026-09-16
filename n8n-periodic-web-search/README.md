# Periodic Web Search (n8n workflow)

An n8n workflow that runs on a schedule, searches multiple engines for
several keyword groups (with AND / OR / NOT logic), and appends new,
de-duplicated hits to a timestamped Markdown log.

Import `workflow.json` into n8n (Workflows -> Import from File).

## What it does

1. **Periodic Trigger** (Schedule Trigger) fires on an interval (default:
   every hour).
2. **Search Configuration** (Set node) holds the knobs you edit:
   - `keywordGroups` - a JSON array of keyword groups, each with:
     - `all` - terms that must ALL appear (AND)
     - `any` - terms where AT LEAST ONE must appear (OR)
     - `none` - terms that must NOT appear (NOT)
   - `engines` - comma-separated list from `google,bing,brave,duckduckgo`
   - `outputFile` - absolute path to the Markdown log on disk
   - `resultsPerQuery` - how many results to request per engine/query
3. **Build Query Matrix** (Code) expands every keyword group x every
   enabled engine into a query string, e.g.
   `"artificial intelligence" regulation (EU OR "European Union" OR Brussels) -cryptocurrency`.
4. **Route By Engine** (Switch) sends each query to the matching HTTP
   Request node:
   - Google Custom Search JSON API
   - Bing Web Search API (Azure Cognitive Services)
   - Brave Search API
   - DuckDuckGo HTML results page (no API key required, best-effort - see
     caveat below)
5. Each engine's response is normalized (Code nodes / an HTML Extract
   node for DuckDuckGo) into a common shape:
   `{ engine, keywordGroup, query, title, link, snippet }`.
6. **Merge All Results** concatenates the four branches.
7. **Timestamp Results** stamps every hit with the current UTC date and
   time.
8. **Dedupe & Append To File** reads the existing Markdown log (if any),
   extracts every URL already logged (normalized - trailing slash and
   `utm_*` query params stripped, so re-crawled tracking-tagged links
   still count as duplicates), drops anything already seen, and appends
   only the new entries.
9. **Has New Results?** branches into two `NoOp` nodes purely so a run
   with nothing new is visually distinct from one that logged hits (handy
   if you attach a notification later).

## Output format

Each new hit is appended as one Markdown block:

```markdown
### Article title
- **Date:** 2026-09-16
- **Time:** 14:32:10 UTC
- **Matched keywords:** AI Regulation EU
- **Search engine:** google
- **Link:** [Article title](https://example.com/article)
- **Lead:** One or two lines of summary/snippet from the search result.

---
```

## Setup

### 1. Environment variables (n8n process)

| Variable | Used by | Required? |
|---|---|---|
| `GOOGLE_CSE_API_KEY` | Google Custom Search | only if `google` is in `engines` |
| `GOOGLE_CSE_CX` | Google Custom Search (search engine ID) | only if `google` is in `engines` |
| `BING_SEARCH_API_KEY` | Bing Web Search | only if `bing` is in `engines` |
| `BRAVE_SEARCH_API_KEY` | Brave Search | only if `brave` is in `engines` |

DuckDuckGo needs no key. Drop any engine you don't have credentials for
from the `engines` field in Search Configuration.

### 2. Allow `fs`/`path` in the Code node

The "Dedupe & Append To File" node reads and writes the log file directly
with Node's `fs` module, which n8n's Code node sandbox blocks by default.
Set this on the n8n process:

```
NODE_FUNCTION_ALLOW_BUILTIN=fs,path
```

(In Docker: add it to the `environment:` section or `.env` file used by
the n8n container.)

### 3. Output path

Point `outputFile` at a path the n8n process can write to and that
persists across restarts (a mounted volume in Docker, not the ephemeral
container filesystem).

### 4. Keyword groups

Edit the `keywordGroups` JSON in **Search Configuration**. Example:

```json
[
  {
    "name": "AI Regulation EU",
    "all": ["artificial intelligence", "regulation"],
    "any": ["EU", "European Union", "Brussels"],
    "none": ["cryptocurrency"]
  }
]
```

This becomes the query: `"artificial intelligence" regulation (EU OR "European Union" OR Brussels) -cryptocurrency`.
`all` = AND, `any` = OR, `none` = NOT. Multi-word terms are auto-quoted.

### 5. Schedule

Edit the interval on the **Periodic Trigger** node (default: every hour).

## Caveats

- **DuckDuckGo** has no official search API; this workflow scrapes its
  HTML results page (`html.duckduckgo.com/html/`), which is unofficial,
  rate-limited, and will need selector updates (`.result__a`,
  `.result__snippet`) if DuckDuckGo changes its markup. Useful for
  testing without any API keys; for production reliability, prefer the
  keyed engines.
- Each HTTP Request node has `onError: continueRegularOutput`, so one
  engine failing (bad key, quota, timeout) doesn't stop the others or
  break the run.
- Node parameter names (Switch/HTTP Request/HTML Extract options) were
  authored against n8n's late-1.x node schema. If import shows a node
  needing reconfiguration, re-check that node's operation/response-format
  dropdowns against your installed n8n version.
