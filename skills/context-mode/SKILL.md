---
name: context-mode
description: |
  Use context-mode tools (execute, execute_file) instead of bash/cat when processing
  large outputs. Trigger phrases: "analyze logs", "summarize output", "process data",
  "parse JSON", "filter results", "extract errors", "check build output",
  "analyze dependencies", "process API response", "large file analysis",
  "extract elements", "run tests", "test output", "coverage report",
  "git log", "recent commits", "diff between branches", "list containers",
  "pod status", "disk usage", "fetch docs", "API reference",
  "index documentation", "hit endpoint", "call API", "check response",
  "query results", "show tables", "find TODOs", "count lines",
  "codebase statistics", "security audit", "outdated packages",
  "dependency tree", "cloud resources", "CI/CD output".
  Also triggers on ANY MCP tool output that may exceed 20 lines,
  and any operation where output size is uncertain.
---

# Context Mode: Default for All Large Output

## MANDATORY RULE

**Default to context-mode for ALL commands that produce large output. Only use bash for guaranteed-small-output
operations.**

Bash whitelist (safe to run directly):

- **File mutations**: `mkdir`, `mv`, `cp`, `rm`, `touch`, `chmod`
- **Git writes**: `git add`, `git commit`, `git push`, `git checkout`, `git branch`, `git merge`
- **Navigation**: `cd`, `pwd`, `which`
- **Process control**: `kill`
- **Package management**: `npm install`, `npm publish`, `pip install`
- **Simple output**: `echo`, `printf`

**Everything else → `context-mode-execute` or `context-mode-execute_file`.** Any command that reads, queries, fetches,
lists, logs, tests, builds, diffs, inspects, or calls an external service. This includes ALL CLIs (gh, aws, kubectl,
docker, terraform, wrangler, fly, heroku, gcloud, etc.) — there are thousands and we cannot list them all.

**When uncertain, use context-mode.** Every KB of unnecessary context reduces the quality and speed of the entire
session.

## Decision Tree

```
About to run a command / read a file / call an API?
│
├── Command is on the bash whitelist (file mutations, git writes, navigation, echo)?
│   └── Use bash
│
├── Output MIGHT be large or you're UNSURE?
│   └── Use context-mode-execute or context-mode-execute_file
│
├── Fetching web documentation or HTML page?
│   └── Use context-mode-fetch_and_index → context-mode-search
│
├── Processing output from another MCP tool (GitHub API, etc.)?
│   ├── Output already in context from a previous tool call?
│   │   └── Use it directly. Do NOT re-index with index(content: ...).
│   ├── Need to search the output multiple times?
│   │   └── Save to file via execute, then context-mode-index(path) → context-mode-search
│   └── One-shot extraction?
│       └── Save to file via execute, then context-mode-execute_file(path)
│
└── Reading a file to analyze/summarize (not edit)?
    └── Use context-mode-execute_file (file loads into FILE_CONTENT, not context)
```

## When to Use Each Tool

| Situation                       | Tool                                                                                | Example                                            |
|---------------------------------|-------------------------------------------------------------------------------------|----------------------------------------------------|
| Hit an API endpoint             | `context-mode-execute`                                                              | `fetch('http://localhost:3000/api/orders')`        |
| Run CLI that returns data       | `context-mode-execute`                                                              | `gh pr list`, `aws s3 ls`, `kubectl get pods`      |
| Run tests                       | `context-mode-execute`                                                              | `npm test`, `pytest`, `go test ./...`              |
| Git operations                  | `context-mode-execute`                                                              | `git log --oneline -50`, `git diff HEAD~5`         |
| Docker/K8s inspection           | `context-mode-execute`                                                              | `docker stats --no-stream`, `kubectl describe pod` |
| Read a log file                 | `context-mode-execute_file`                                                         | Parse access.log, error.log, build output          |
| Read a data file                | `context-mode-execute_file`                                                         | Analyze CSV, JSON, YAML, XML                       |
| Read source code to analyze     | `context-mode-execute_file`                                                         | Count functions, find patterns, extract metrics    |
| Fetch web docs                  | `context-mode-fetch_and_index`                                                      | Index React/Next.js/Zod docs, then search          |
| MCP output (already in context) | Use directly                                                                        | Don't re-index — it's already loaded               |
| MCP output (need multi-query)   | `context-mode-execute` to save → `context-mode-index(path)` → `context-mode-search` | Save to file first, index server-side              |

## Automatic Triggers

Use context-mode for ANY of these, without being asked:

- **API debugging**: "hit this endpoint", "call the API", "check the response", "find the bug in the response"
- **Log analysis**: "check the logs", "what errors", "read access.log", "debug the 500s"
- **Test runs**: "run the tests", "check if tests pass", "test suite output"
- **Git history**: "show recent commits", "git log", "what changed", "diff between branches"
- **Data inspection**: "look at the CSV", "parse the JSON", "analyze the config"
- **Infrastructure**: "list containers", "check pods", "S3 buckets", "show running services"
- **Dependency audit**: "check dependencies", "outdated packages", "security audit"
- **Build output**: "build the project", "check for warnings", "compile errors"
- **Code metrics**: "count lines", "find TODOs", "function count", "analyze codebase"
- **Web docs lookup**: "look up the docs", "check the API reference", "find examples"

## Language Selection

| Situation                 | Language     | Why                                   |
|---------------------------|--------------|---------------------------------------|
| HTTP/API calls, JSON      | `javascript` | Native fetch, JSON.parse, async/await |
| Data analysis, CSV, stats | `python`     | csv, statistics, collections, re      |
| Shell commands with pipes | `shell`      | grep, awk, jq, native tools           |
| File pattern matching     | `shell`      | find, wc, sort, uniq                  |

## Search Query Strategy

- BM25 uses **OR semantics** — results matching more terms rank higher automatically
- Use 2-4 specific technical terms per query
- **Always use `source` parameter** when multiple docs are indexed to avoid cross-source contamination
    - Partial match works: `source: "Node"` matches `"Node.js v22 CHANGELOG"`
- **Always use `queries` array** — batch ALL search questions in ONE call:
    - `context-mode-search(queries: ["transform pipe", "refine superRefine", "coerce codec"], source: "Zod")`
    - NEVER make multiple separate search() calls — put all queries in one array

## External Documentation

- **Always use `context-mode-fetch_and_index`** for external docs — NEVER `cat` or `context-mode-execute` with local
  paths for packages you don't own
- For GitHub-hosted projects, use the raw URL: `https://raw.githubusercontent.com/org/repo/main/CHANGELOG.md`
- After indexing, use the `source` parameter in search to scope results to that specific document

## Critical Rules

1. **Always console.log/print your findings.** stdout is all that enters context. No output = wasted call.
2. **Write analysis code, not just data dumps.** Don't `console.log(JSON.stringify(data))` — analyze first, print
   findings.
3. **Be specific in output.** Print bug details with IDs, line numbers, exact values — not just counts.
4. **For files you need to EDIT**: Use the normal `view` tool. context-mode is for analysis, not editing.
5. **For bash whitelist commands only**: Use bash for file mutations, git writes, navigation, process control, package
   install, and echo. Everything else goes through context-mode.
6. **Never use `context-mode-index(content: large_data)`.** Use `context-mode-index(path: ...)` to read files
   server-side. The `content` parameter sends data through context as a tool parameter — use it only for small inline
   text.
7. **Don't re-index data already in context.** If an MCP tool returned data in a previous response, it's already
   loaded — use it directly or save to file first.

## Examples

### Debug an API endpoint

```javascript
const resp = await fetch('http://localhost:3000/api/orders');
const {orders} = await resp.json();

const bugs = [];
const negQty = orders.filter(o => o.quantity < 0);
if (negQty.length) bugs.push(`Negative qty: ${negQty.map(o => o.id).join(', ')}`);

const nullFields = orders.filter(o => !o.product || !o.customer);
if (nullFields.length) bugs.push(`Null fields: ${nullFields.map(o => o.id).join(', ')}`);

console.log(`${orders.length} orders, ${bugs.length} bugs found:`);
bugs.forEach(b => console.log(`- ${b}`));
```

### Analyze test output

```shell
npm test 2>&1
echo "EXIT=$?"
```

### Check GitHub PRs

```shell
gh pr list --json number,title,state,reviewDecision --jq '.[] | "\(.number) [\(.state)] \(.title) — \(.reviewDecision // "no review")"'
```

### Read and analyze a large file

```python
# FILE_CONTENT is pre-loaded by execute_file
import json

data = json.loads(FILE_CONTENT)
print(f"Records: {len(data)}")
# ... analyze and print findings
```

## Subagent Usage

When using the `task` tool to launch subagents, include context-mode tool names in your prompt so subagents know to use
them. Subagents have access to the same MCP
tools (`context-mode-batch_execute`, `context-mode-execute`, `context-mode-search`, etc.).

## Anti-Patterns

- Using `curl http://api/endpoint` via bash → 50KB floods context. Use `context-mode-execute` with fetch instead.
- Using `cat large-file.json` via bash → entire file in context. Use `context-mode-execute_file` instead.
- Using `gh pr list` via bash → raw JSON in context. Use `context-mode-execute` with `--jq` filter instead.
- Piping bash output through `| head -20` → you lose the rest. Use `context-mode-execute` to analyze ALL data and print
  summary.
- Running `npm test` via bash → full test output in context. Use `context-mode-execute` to capture and summarize.
- Passing ANY large data to `context-mode-index(content: ...)` → data enters context as a parameter. **Always**
  use `context-mode-index(path: ...)` to read server-side. The `content` parameter should only be used for small inline
  text you're composing yourself.
- Calling an MCP tool then passing the response to `context-mode-index(content: response)` → **doubles** context usage.
  The response is already in context — use it directly or save to file first.

## Reference Files

- [JavaScript/TypeScript Patterns](./references/patterns-javascript.md)
- [Python Patterns](./references/patterns-python.md)
- [Shell Patterns](./references/patterns-shell.md)
- [Anti-Patterns & Common Mistakes](./references/anti-patterns.md)
