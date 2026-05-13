---
name: private-firecrawl
description: Use the private self-hosted Firecrawl service attached to Hermes for controlled scrape, crawl, and extraction workflows when web_extract is not enough.
version: 0.1.0
platforms: [linux]
metadata:
  hermes:
    tags: [firecrawl, web, scraping, extraction]
    category: web
    requires_toolsets: [terminal]
---

# Private Firecrawl

## When to Use
Use this skill when the user explicitly wants the private self-hosted Firecrawl stack, batch scraping, crawl-job handling, PDF/page extraction through Firecrawl, or Firecrawl troubleshooting.

Prefer `web_extract` for ordinary single-page URL or PDF extraction. Use private Firecrawl only when the private stack is required or when `web_extract` is insufficient.

## Endpoints
- From the Hermes Docker network: `http://firecrawl:3002`
- From the VPS host: `http://127.0.0.1:3002`

Do not read or print `/etc/firecrawl/firecrawl.env`. Do not expose the private Firecrawl API publicly.

## Response Contract
Terminal output and progress events are intermediate state, not the user-facing answer.
After using Firecrawl, always send a final response that includes:
- how many URLs were discovered, attempted, succeeded, and failed
- the requested crawl results, or a concise sample plus the saved artifact path for large batches
- per-URL failures with enough detail to retry

For requests like "crawl all stories on this page", do not stop after scraping the index page. First discover the story links on the page, de-duplicate them, then scrape those linked stories. If the user explicitly says "all", treat that as approval for a larger batch; otherwise ask before crawling more than 25 URLs.

## Single URL Scrape
Use Python's standard library so the workflow works in the Hermes container even when `jq` is unavailable. Keep user input as data, not interpolated shell syntax:

```bash
python3 <<'PY'
import json
import urllib.request

url = "https://example.com"
payload = json.dumps({"url": url, "formats": ["markdown"]}).encode("utf-8")
request = urllib.request.Request(
    "http://firecrawl:3002/v1/scrape",
    data=payload,
    headers={"Content-Type": "application/json"},
    method="POST",
)
with urllib.request.urlopen(request, timeout=90) as response:
    data = json.loads(response.read())

metadata = data.get("data", {}).get("metadata", {}) or data.get("metadata", {}) or {}
markdown = data.get("data", {}).get("markdown") or data.get("markdown") or ""
print(json.dumps({
    "success": data.get("success"),
    "source": metadata.get("sourceURL") or metadata.get("url") or url,
    "title": metadata.get("title"),
    "markdown": markdown,
}, ensure_ascii=False))
PY
```

If running from the VPS host instead of inside the Hermes Docker network, replace `http://firecrawl:3002` with `http://127.0.0.1:3002`.

## Index Page or Story Gallery
For an index page that links to many stories, scrape the index with links enabled, filter out navigation/static links, then batch scrape the discovered story URLs:

```bash
python3 <<'PY'
import json
import re
import urllib.parse
import urllib.request

index_url = "https://example.com/stories"
endpoint = "http://firecrawl:3002/v1/scrape"

payload = json.dumps({
    "url": index_url,
    "formats": ["markdown", "links"],
    "onlyMainContent": False,
}).encode("utf-8")
request = urllib.request.Request(endpoint, data=payload, headers={"Content-Type": "application/json"}, method="POST")
with urllib.request.urlopen(request, timeout=90) as response:
    data = json.loads(response.read())

raw_links = data.get("data", {}).get("links") or data.get("links") or []
if isinstance(raw_links, dict):
    raw_links = list(raw_links.values())
if not raw_links:
    markdown = data.get("data", {}).get("markdown") or data.get("markdown") or ""
    raw_links = re.findall(r"\]\(([^)]+)\)", markdown)

seen = set()
story_urls = []
index_host = urllib.parse.urlparse(index_url).netloc
for item in raw_links:
    href = item.get("url") if isinstance(item, dict) else str(item)
    url = urllib.parse.urljoin(index_url, href)
    parsed = urllib.parse.urlparse(url)
    if parsed.scheme not in {"http", "https"} or url in seen:
        continue
    if parsed.netloc == index_host and (parsed.path.startswith("/docs") or parsed.path.startswith("/assets")):
        continue
    seen.add(url)
    story_urls.append(url)

print(json.dumps({"index": index_url, "discovered": len(story_urls), "urls": story_urls}, ensure_ascii=False))
PY
```

## Batch Scrape
For a short user-approved list of URLs, put one URL per line in a temporary file and emit one JSON result per line:

```bash
python3 <<'PY'
import json
import urllib.request

endpoint = "http://firecrawl:3002/v1/scrape"
with open("urls.txt", encoding="utf-8") as handle:
    urls = [line.strip() for line in handle if line.strip()]

for url in urls:
    payload = json.dumps({"url": url, "formats": ["markdown"]}).encode("utf-8")
    request = urllib.request.Request(
        endpoint,
        data=payload,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(request, timeout=90) as response:
        data = json.loads(response.read())
    metadata = data.get("data", {}).get("metadata", {}) or data.get("metadata", {}) or {}
    markdown = data.get("data", {}).get("markdown") or data.get("markdown") or ""
    print(json.dumps({
        "url": metadata.get("sourceURL") or metadata.get("url") or url,
        "success": data.get("success"),
        "markdown": markdown,
    }, ensure_ascii=False))
PY
```

For larger batches, write raw results to `/opt/data/artifacts/firecrawl/<task>.ndjson` or `.md` and mention that path in the final response instead of pasting every full page into chat.

Keep batches small unless the user explicitly approves a larger crawl. Avoid scraping login-gated, sensitive, or private third-party content unless the user confirms they have permission.

## Troubleshooting
- Check API reachability with `curl -fsS http://firecrawl:3002` from Hermes, or `curl -fsS http://127.0.0.1:3002` from the host.
- If `firecrawl` does not resolve, the API container may not be attached to the Hermes Docker network with alias `firecrawl`.
- On the VPS, `sudo hermes-vps status` should include `firecrawl_api_on_hermes_network=hermes_default` and `firecrawl_api_alias=firecrawl`.
- If scrape requests fail after a fresh Firecrawl init, check the VPS operator notes before recreating Firecrawl volumes; do not delete volumes casually.

## Verification
After a scrape, confirm:
- the response is valid JSON
- `success` is true, or the error explains the failure
- extracted markdown is relevant to the requested URL
- no secrets, tokens, cookies, or private env values appear in the output
