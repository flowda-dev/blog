# How to integrate a web search API into an LLM agent

## The problem

An LLM agent that searches the web usually has to produce three things at once:

1. a query that fits the current task;
2. enough source content to reason from;
3. an answer that can cite the pages it used.

The hard part is not the HTTP request. It is treating search results as evidence instead of treating every URL and snippet as equally useful.

## Separate search from extraction

Search and extraction answer different questions:

- **Search** tells you which pages exist and which ones look relevant.
- **Extraction** tells you what a specific page actually says.

A search API can combine both, but your application should still keep the two concerns separate in memory. Store the URL, snippet, and page content separately so later steps can reason about why a page was selected and what it actually contains.

## Design an evidence record

Before adding an LLM, define the record that the agent will reason from:

```ts
type Evidence = {
  url: string;
  title: string;
  snippet?: string;
  content?: string;
  contentSource: "extracted" | "snippet";
  selectedBecause: string;
};
```

This is more useful than passing the raw API response to the model. It also makes retrieval, citation, and debugging easier.

## A practical Python example

The example below uses a search endpoint with inline extraction. It intentionally avoids assuming that every result is usable: a failed extraction is skipped rather than treated as missing context.

```python
import os
import requests

API_KEY = os.environ["FLOWDA_API_KEY"]

def search_for_evidence(query: str, max_results: int = 3) -> list[dict]:
    response = requests.post(
        "https://openapi.flowda.ai/v1/search",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={
            "query": query,
            "max_results": max_results,
            "extract_content": True,
            "extract_options": {
                "formats": ["markdown"],
                "max_age_ms": 0,
            },
        },
        timeout=130,
    )
    response.raise_for_status()
    body = response.json()

    if body.get("code") != 0:
        raise RuntimeError(f"search failed: {body.get('code')} {body.get('message')}")

    data = body.get("data") or {}
    if not data.get("success"):
        raise RuntimeError("search did not return usable data")

    evidence = []

    for result in data.get("outputs") or []:
        extract = result.get("extract") or {}
        content = (extract.get("outputs") or {}).get("markdown")

        evidence.append({
            "url": result["url"],
            "title": result["title"],
            "snippet": result.get("snippet"),
            "content": content,
            "contentSource": "extracted" if content else "snippet",
            "selectedBecause": f"search result for: {query}",
        })

    return evidence
```

Run it only after you have checked the response shape in the API reference. Endpoint scope, field names, and rate limits should come from your provider's documentation, not from a code sample alone.

## Validate before the model sees the data

Treat these as separate validation steps:

- the request reached the API;
- the API call succeeded;
- the search returned results;
- each page extraction succeeded;
- the extracted content is nonempty and relevant;
- the result can support the answer you are trying to produce.

A page that loads but contains only navigation text is not usable evidence. Neither is a result whose snippet matches the query but whose extracted page does not.

## Control freshness and cost

Web search becomes expensive when every agent loop performs a full search and extraction. Use a small policy:

- search once per task unless new facts are needed;
- reuse recent snapshots for stable pages;
- force a fresh fetch for prices, availability, or fast-changing docs;
- cap `max_results` to what the task actually needs;
- stop the loop when the evidence is sufficient.

## Prefer testable failure paths

A search API can return:

- no results;
- a failed extraction;
- a valid HTTP response with unusable content;
- a rate limit or auth error;
- a page that redirects to an unexpected host.

Handle these separately. Do not collapse them into one generic "search failed" error, or your agent will lose the information it needs to recover.

## Minimum integration checklist

1. Define the evidence record.
2. Search with a bounded `max_results`.
3. Extract only the pages that matter.
4. Validate extraction success before use.
5. Store the URL, title, snippet, content, and reason for selection.
6. Return the URL as the citation source.
7. Log failures by error kind.
8. Test with a task-specific query, not a demo keyword.

## Related Flowda resources

- [Search the web](https://docs.flowda.ai/docs/api/search)
- [Fetch a page and return its content](https://docs.flowda.ai/docs/api/extract)
- [Flowda developer docs](https://docs.flowda.ai/)
