# Copilot Context Mode

**The other half of the context problem — adapted for GitHub Copilot.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

https://github.com/user-attachments/assets/2163dfb6-455a-436e-8dbe-7e7ed1a01cf8

Every MCP tool call dumps raw data into your context window. A Playwright snapshot costs 56 KB. Twenty GitHub issues cost 59 KB. One access log — 45 KB. After 30 minutes, 40% of your context is gone.

Inspired by Cloudflare's [Code Mode](https://blog.cloudflare.com/code-mode-mcp/) and [mksglu/claude-context-mode](https://github.com/mksglu/claude-context-mode), this MCP server sits between Copilot and tool outputs. **315 KB becomes 5.4 KB. 98% reduction.**

## Install

### Option 1: Local development

```bash
git clone https://github.com/DriveWealth/copilot-context-mode.git
cd copilot-context-mode && npm install && npm run build
```

Then add to your MCP configuration:

```json
{
  "mcpServers": {
    "context-mode": {
      "command": "node",
      "args": ["/path/to/copilot-context-mode/start.mjs"]
    }
  }
}
```

### Setup

Unlike Claude, GitHub Copilot does not support hooks to automatically activate MCPs. Instead, you must instruct Copilot to use context-mode tools via a global instructions file. Without this, Copilot will ignore the MCP even when it's configured.

Create `~/.copilot/copilot-instructions.md` with the following content:

````markdown
# Copilot Global Instructions

## Context-Mode Tool Preference

When the **copilot-context-mode MCP** is available, prefer its tools over defaults for any operation that reads,
queries, fetches, or may produce large output. This keeps context lean and avoids truncation.

### Tool preference rules

| Instead of                                                  | Use                                                                                                        |
|-------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `bash` (read-heavy commands)                                | `context-mode-execute`                                                                                     |
| `view` / `cat` / `head` / `tail`                            | `context-mode-execute_file`                                                                                |
| multiple sequential `bash`/`execute` calls                  | `context-mode-batch_execute`                                                                               |
| `web_fetch`                                                 | `context-mode-fetch_and_index` + `context-mode-search`                                                     |
| loading docs into context directly                          | `context-mode-index` + `context-mode-search`                                                               |
| large `atlassian-jira_*` / `atlassian-confluence_*` results | call the MCP tool, then **in the same response** immediately pass result to `context-mode-index` + query with `context-mode-search` |

**`bash` is reserved for writes and side effects only:** file mutations (`mkdir`, `mv`, `cp`, `rm`, `touch`, `echo >`,
redirects), git writes (`add`, `commit`, `push`, `checkout`, `branch`, `merge`), package installs, process
control (`kill`), and navigation (`cd`, `pwd`).

### Jira / Confluence → context-mode is MANDATORY

**Every** `atlassian-jira_*` or `atlassian-confluence_*` call MUST be followed — in the **same response** —
by `context-mode-index` to store the result, then `context-mode-search` to retrieve only what's needed. Never leave raw
Jira/Confluence JSON sitting in the context window.

```
# Required pattern — always do this in one response:
1. atlassian-jira_get_issue(issue_key="PROJ-123")        # fetch
2. context-mode-index(content=<result>, source="Jira: PROJ-123")  # index
3. context-mode-search(queries=["relevant term"])         # retrieve
```

### batch_execute is your primary tool

For any task requiring multiple commands or queries, use `batch_execute` in a **single call** with all commands and
search queries together. This is far more efficient than sequential `execute` calls.

```
# Good: one batch_execute with commands + queries
batch_execute(
  commands=[{label: "...", command: "..."}],
  queries=["what I'm looking for", "related term"]
)

# Bad: multiple execute calls
execute("cmd1") → execute("cmd2") → search("query")
```
````

This file is loaded by Copilot at session start and instructs it to route all read-heavy operations through the context-mode MCP tools.

## The Problem

MCP has become the standard way for AI agents to use external tools. But there is a tension at its core: every tool interaction fills the context window from both sides — definitions on the way in, raw output on the way out.

Context Mode processes tool outputs in sandboxes so only summaries reach the model.

## Tools

| Tool | What it does | Context saved |
|---|---|---|
| `batch_execute` | Run multiple commands + search multiple queries in ONE call. | 986 KB → 62 KB |
| `execute` | Run code in 10 languages. Only stdout enters context. | 56 KB → 299 B |
| `execute_file` | Process files in sandbox. Raw content never leaves. | 45 KB → 155 B |
| `index` | Chunk markdown into FTS5 with BM25 ranking. | 60 KB → 40 B |
| `search` | Query indexed content with multiple queries in one call. | On-demand retrieval |
| `fetch_and_index` | Fetch URL, convert to markdown, index. | 60 KB → 40 B |

## How the Sandbox Works

Each `execute` call spawns an isolated subprocess with its own process boundary. Scripts can't access each other's memory or state. The subprocess runs your code, captures stdout, and only that stdout enters the conversation context. The raw data — log files, API responses, snapshots — never leaves the sandbox.

Eleven language runtimes are available: JavaScript, TypeScript, Python, Shell, Ruby, Go, Rust, PHP, Perl, R, and Elixir. Bun is auto-detected for 3-5x faster JS/TS execution.

Authenticated CLIs work through credential passthrough — `gh`, `aws`, `gcloud`, `kubectl`, `docker` inherit environment variables and config paths without exposing them to the conversation.

When output exceeds 5 KB and an `intent` is provided, Context Mode switches to intent-driven filtering: it indexes the full output into the knowledge base, searches for sections matching your intent, and returns only the relevant matches with a vocabulary of searchable terms for follow-up queries.

## How the Knowledge Base Works

The `index` tool chunks markdown content by headings while keeping code blocks intact, then stores them in a **SQLite FTS5** (Full-Text Search 5) virtual table. Search uses **BM25 ranking** — a probabilistic relevance algorithm that scores documents based on term frequency, inverse document frequency, and document length normalization. **Porter stemming** is applied at index time so "running", "runs", and "ran" match the same stem.

When you call `search`, it returns relevant content snippets focused around matching query terms — not full documents, not approximations, the actual indexed content with smart extraction around what you're looking for. `fetch_and_index` extends this to URLs: fetch, convert HTML to markdown, chunk, index. The raw page never enters context.

## Fuzzy Search

Search uses a three-layer fallback to handle typos, partial terms, and substring matches:

- **Layer 1 — Porter stemming**: Standard FTS5 MATCH with porter tokenizer. "caching" matches "cached", "caches", "cach".
- **Layer 2 — Trigram substring**: FTS5 trigram tokenizer matches partial strings. "useEff" finds "useEffect", "authenticat" finds "authentication".
- **Layer 3 — Fuzzy correction**: Levenshtein distance corrects typos before re-searching. "kuberntes" → "kubernetes", "autentication" → "authentication".

## The Numbers

Measured across real-world scenarios:

**Playwright snapshot** — 56.2 KB raw → 299 B context (99% saved)
**GitHub Issues (20)** — 58.9 KB raw → 1.1 KB context (98% saved)
**Access log (500 requests)** — 45.1 KB raw → 155 B context (100% saved)
**React docs** — 5.9 KB raw → 261 B context (96% saved)
**Analytics CSV (500 rows)** — 85.5 KB raw → 222 B context (100% saved)
**Git log (153 commits)** — 11.6 KB raw → 107 B context (99% saved)
**Test output (30 suites)** — 6.0 KB raw → 337 B context (95% saved)
**Repo research (subagent)** — 986 KB raw → 62 KB context (94% saved, 5 calls vs 37)

Over a full session: 315 KB of raw output becomes 5.4 KB.

[Full benchmark data with 21 scenarios →](BENCHMARK.md)

## Requirements

- **Node.js 18+**
- **GitHub Copilot** with MCP support
- Optional: Bun (auto-detected, 3-5x faster JS/TS)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the local development workflow and testing guidelines.

```bash
git clone https://github.com/DriveWealth/copilot-context-mode.git
cd copilot-context-mode && npm install
npm test              # run tests
npm run test:all      # full suite
```

## Credits

Forked from [mksglu/claude-context-mode](https://github.com/mksglu/claude-context-mode) by Mert Koseoğlu. Original concept inspired by Cloudflare's [Code Mode](https://blog.cloudflare.com/code-mode-mcp/).

## License

MIT
