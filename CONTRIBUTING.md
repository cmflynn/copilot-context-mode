# Contributing to copilot-context-mode

Contributions welcome! Every issue, every PR, every idea matters.

---

## Prerequisites

- Node.js 20+ or [Bun](https://bun.sh/) (recommended for speed)
- GitHub Copilot with MCP support

## Local Development Setup

### 1. Clone and install

```bash
git clone https://github.com/DriveWealth/copilot-context-mode.git
cd copilot-context-mode
npm install
npm run build
```

### 2. Configure MCP to use your local clone

Add to your MCP configuration:

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

### 3. Restart your Copilot session

The MCP server reloads on session start.

## Development Workflow

### Build and test your changes

```bash
# TypeScript compilation
npm run build

# Run all tests
npm run test:all

# Type checking only
npm run typecheck
```

### Available test suites

```bash
npm run test:all            # Run everything
npm run test:store          # FTS5 knowledge base tests
npm run test:fuzzy          # Fuzzy search tests
npm run test:search-wiring  # Search pipeline tests
npm run test:search-fallback # Fallback integration tests
npm run test:stream-cap     # Stream capacity tests
npm run test:turndown       # HTML-to-markdown tests
npm run test:use-cases      # End-to-end use case tests
npm run test:compare        # Context savings comparison
npm run benchmark           # Performance benchmarks
```

## TDD Workflow

We follow test-driven development. Every PR should include tests.

### Red-Green-Refactor

1. **Red** — Write a failing test for the behavior you want
2. **Green** — Write the minimum code to make it pass
3. **Refactor** — Clean up while keeping tests green

### Output quality matters

When your change affects tool output (execute, search, fetch_and_index, etc.), always compare before and after:

1. Run the same prompt **before** your change (on `main`)
2. Run it **again** with your change
3. Include both outputs in your PR

## Submitting a Pull Request

1. Fork the repository
2. Create a feature branch from `main`
3. Follow the local development setup above
4. Write tests first (TDD)
5. Run `npm run test:all` and `npm run typecheck`
6. Test in a live Copilot session
7. Compare output quality before/after
8. Open a PR

## Quick Reference

| Task | Command |
|---|---|
| Rebuild after changes | `npm run build` |
| Run all tests | `npm run test:all` |
| Type check | `npm run typecheck` |
| Dev server | `npm run dev` |
