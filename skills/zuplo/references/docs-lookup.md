# Zuplo Documentation Lookup

How to find and use Zuplo documentation when configuring the API gateway.

## How to look up documentation

1. **Fetch the doc index** to discover available pages:

   ```
   https://zuplo.com/docs/llms.txt
   ```

   This returns a listing of all doc pages with titles and URLs.

2. **Find the relevant page** in the index for the feature you need to configure.

3. **Fetch that specific page** to get the full documentation with correct configuration format, required fields, and examples.

4. **Configure the feature** using the exact format from the docs. Never guess at configuration — always base it on what the docs say.

## MCP tools (if available)

If the Zuplo Docs MCP server is connected, you can also use:

- **search-zuplo-docs** — Search across all docs by keyword
- **ask-question-about-zuplo** — Ask a natural language question about Zuplo

These are convenient alternatives to fetching the raw docs, but the doc pages at `zuplo.com/docs/` remain the authoritative source.
