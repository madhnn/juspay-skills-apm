---
name: connect
description: Re-authenticate the Juspay MCP servers. Run this if juspay-mcp or juspay-docs-mcp stops working due to an expired session.
allowed-tools:
  - Bash
  - mcp__juspay-docs-mcp__authenticate
  - mcp__juspay-docs-mcp__complete_authentication
  - mcp__juspay-mcp__authenticate
  - mcp__juspay-mcp__complete_authentication
---

Re-authenticate the Juspay MCP servers.

## juspay-docs-mcp

1. Call `juspay-docs-mcp.authenticate()` to get the authorization URL.
2. Open it in the browser: `open "<url>"` (macOS) or `xdg-open "<url>"` (Linux).
3. If the OAuth flow redirects to localhost and a code is captured automatically, call `complete_authentication` with it.
4. If the browser shows a code the user must copy, ask them to paste it here and call `complete_authentication` with it.
5. Confirm success.

## juspay-mcp

Attempt `juspay_get_merchant_details` first.
- If it succeeds, juspay-mcp is already authenticated — skip.
- If it fails with an auth error, repeat the same flow using juspay-mcp's `authenticate` and `complete_authentication` tools.

When both are done, tell the user they can run `/integrate` to start the payment integration wizard.
