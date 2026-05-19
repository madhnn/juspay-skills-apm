# juspay-integrate

A Claude Code skill package that provides a fully guided, doc-driven Juspay payment integration wizard.

## What's included

- **`/integrate` skill** — walks you through selecting a Juspay product, fetching live documentation, collecting merchant credentials, and generating production-ready integration code
- **`juspay-mcp`** — live merchant dashboard tools (merchant details, webhook settings, API key provisioning, integration monitoring)
- **`juspay-docs-mcp`** — Juspay documentation MCP (product explorer, doc fetcher)

## Supported products

| Product | Type |
|---------|------|
| Hyper Checkout | Web / Mobile SDK |
| Express Checkout SDK | Web / Mobile SDK |
| Express Checkout API | Backend API |
| Hyper UPI / UPI TPAP SDK | Mobile SDK |
| Payout | Backend API |
| Juspay Billing | Backend API |
| LotUS Pay / JusBiz | Specialized |

## Usage

After installing, type `/integrate` in Claude Code to start the wizard. Optionally pass `--product <id>` to skip product selection.

## Install

```bash
apm install
```

> **Note:** `juspay-docs-mcp` points to a hosted endpoint. Update the URL in `apm.yml` if the endpoint changes.
