# QuantRisk MCP

**Portfolio risk analytics as MCP tools — VaR, Monte Carlo, optimization, options Greeks, and stress testing — for AI assistants.**

→ **Project home: [quantrisk.dev](https://quantrisk.dev)**

[![npm version](https://img.shields.io/npm/v/@quantrisk/mcp-server)](https://www.npmjs.com/package/@quantrisk/mcp-server)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![MCP Compatible](https://img.shields.io/badge/MCP-compatible-blue)](https://modelcontextprotocol.io)

---

## What it does

There are thousands of MCP servers; very few do quantitative finance. QuantRisk lets your AI assistant answer questions like *"what's my portfolio's VaR at 95%?"* with a real number instead of a definition. It exposes ten institutional-grade analytics tools over MCP — they run server-side on Cloudflare Workers and return structured JSON the model can reason about.

## Install

```bash
npm install -g @quantrisk/mcp-server
```

Get a key at [quantrisk.dev/upgrade](https://quantrisk.dev/upgrade) (free tier available, no card required), then add to your client config.

### Claude Desktop

`~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "quantrisk": {
      "command": "quantrisk-mcp",
      "env": { "QUANTRISK_API_KEY": "qr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }
    }
  }
}
```

### Cursor

`.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global):

```json
{
  "mcpServers": {
    "quantrisk": {
      "command": "quantrisk-mcp",
      "env": { "QUANTRISK_API_KEY": "qr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }
    }
  }
}
```

### Direct HTTP (Streamable)

If your client speaks Streamable HTTP, point it at the hosted endpoint:

```json
{
  "mcpServers": {
    "quantrisk": {
      "transport": "http",
      "url": "https://quantrisk-mcp.quantrisk.workers.dev/mcp",
      "headers": { "Authorization": "Bearer qr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }
    }
  }
}
```

## Tools

| Tool | Purpose | Tier |
|------|---------|------|
| `analyze_risk` | VaR (historical / parametric / Cornish-Fisher), CVaR, volatility, beta, max drawdown | Free |
| `monte_carlo_simulation` | Distribution of future returns across simulated paths | Free |
| `stress_test` | P&L under GFC 2008, COVID 2020, dot-com bust, etc. | Free |
| `correlation_matrix` | Pairwise correlations + eigenvalue decomposition | Free |
| `performance_attribution` | Sharpe, Sortino, Treynor, Calmar, Information ratio | Free |
| `sector_exposure` | GICS sector + market-cap concentration (HHI) | Free |
| `price_history` | Historical OHLCV for one or more tickers | Free |
| `optimize_portfolio` | Mean-variance optimization (max Sharpe / min variance / target return) | Pro |
| `compare_portfolios` | Head-to-head risk/return comparison of 2–5 allocations | Pro |
| `calculate_greeks` | Delta, gamma, theta, vega, rho for options portfolios | Pro |

## Pricing

| | Free | Pro ($29/mo) |
|---|---|---|
| Positions per call | 20 | 500 |
| Monte Carlo paths | 1,000 | 100,000 |
| Price history | 1 ticker × 1 yr | 20 tickers × 5 yr |
| Calls / day | 100 | 5,000 |
| Tools | 7 | 10 |

[Subscribe at quantrisk.dev/upgrade](https://quantrisk.dev/upgrade) — cancel any time.

## Architecture

Two pieces ship from this repo:

1. **`bin/quantrisk-mcp.js`** — a stdio ⇄ Streamable-HTTP bridge installed via `npm install -g`. It reads JSON-RPC from stdin, forwards each message to the hosted server with the user's API key, and writes responses back. This is what Claude Desktop / Cursor talk to.

2. **`src/`** — the Cloudflare Worker that handles the actual MCP protocol, runs the analytics engine, and gates by tier. State lives in a Durable Object (`UserState`) per API key. Stripe handles billing; webhooks promote/demote tiers.

The math is in `src/engine/` — pure TypeScript, no external dependencies, fully unit-tested.

## Self-hosting

You can run your own instance on Cloudflare Workers:

```bash
git clone https://github.com/QuantRisk/mcp-server.git
cd mcp-server
npm install
cp .env.example .env   # fill in STRIPE_SECRET_KEY + STRIPE_WEBHOOK_SECRET
npx wrangler kv namespace create PRICE_CACHE
# put the returned id into wrangler.toml
echo "<sk_...>"   | npx wrangler secret put STRIPE_SECRET_KEY
echo "<whsec_...>" | npx wrangler secret put STRIPE_WEBHOOK_SECRET
npx wrangler deploy
```

You'll need:
- A Cloudflare account (free tier works for low traffic)
- A Stripe account if you want paid-tier gating; you can rip out the tier middleware for a no-auth fork

## Development

```bash
npm install
npm test            # full vitest suite
npm run test:watch  # watch mode
npm run dev         # `wrangler dev` — local Worker on :8787
npm run typecheck   # tsc --noEmit
npm run build       # compile src/ → dist/ (used by npm publish)
```

The engine layer (`src/engine/`) is the highest-leverage place to contribute — it's pure math, has near-100% test coverage, and every formula carries a comment with the source paper.

**Before opening a PR:**
- Engine functions need tests in `test/engine/`
- New external dependencies need a one-line justification in the PR description
- Tool input/output shapes are frozen — schema changes require a version bump

## License

MIT — see [LICENSE](LICENSE). The hosted service at `quantrisk.dev` is a separate commercial offering; this license covers the source code, not access to the API.
