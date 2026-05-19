# Juspay Integration Tools — Setup

Follow these steps in order. Do not skip ahead.

---

## Step 1 — Check apm CLI

Run:

```bash
apm --version
```

If that fails, try

```bash
uvx --from apm-cli apm --version
```

If that fails, try:

```bash
pip install apm-cli && apm --version
```

If neither works, tell the user: "Please install `pip` or `uv` — then re-run this setup."

---

## Step 2 — Detect harnesses and configure apm.yml

Run:

```bash
apm targets
```

Look at which targets have status `active`. Then check if an `apm.yml` already exists in the project.

**If `apm.yml` exists:** add or update its `targets` list to include all active targets, and add the package to `dependencies.apm`:

```yaml
targets:
  - claude # include only if active
  - opencode # include only if active

dependencies:
  apm:
    - https://github.com/madhnn/juspay-skills-apm.git
  mcp: []
```

**If `apm.yml` does not exist:** create it with the active targets and the dependency:

```yaml
name: project
version: 1.0.0
includes: auto
targets:
  - claude # include only active ones from Step 2
  - opencode # include only active ones from Step 2

dependencies:
  apm:
    - ssh://git@ssh.bitbucket.juspay.net/~madhan.k_juspay.in/juspay-skills-apm.git
  mcp: []
```

---

## Step 3 — Install

Run:

```bash
apm install
```

This fetches the package and installs:

- The `/integrate` skill into `.claude/skills/` (and/or `.opencode/skills/`)
- `juspay-mcp` server (live merchant dashboard tools)
- `juspay-docs-mcp` server (Juspay documentation)

Wait for install to complete before continuing. This step may take longer so wait with a 5-minute timeout.

---

## Step 4 — Authenticate juspay-mcp

`juspay-docs-mcp` does not require authentication. Only `juspay-mcp` needs auth.

Attempt to call `juspay_get_merchant_details` on `juspay-mcp`.

- If it succeeds: already authenticated. Skip to Step 5.
- If it returns an auth error, proceed below.

**OpenCode / Codex** support native MCP auth — run:

```bash
opencode mcp auth juspay-mcp   # OpenCode
codex mcp auth juspay-mcp      # Codex
```

This opens the browser directly and handles the callback automatically.

**Note** : If the cli output shows `juspay-mcp` is already authenticated, do you wanna re-authenticate, Skip this process.

---

**Claude does not support `mcp auth` commands.** Use the tool-based flow instead:

1. Call the `authenticate` tool on `juspay-mcp`. It will return an authorization URL.
2. Open that URL in the browser:

```bash
open "<auth_url>"       # macOS
xdg-open "<auth_url>"  # Linux
```

3. **If the browser redirects back automatically (OAuth callback to localhost):**
   The code is captured. Call `complete_authentication` with the code. Confirm success.

   **If the browser shows a code the user must copy:**
   Ask the user to paste the code here, then call `complete_authentication` with it.

---

## Step 5 — Confirm both MCP servers are enabled

Verify that both servers are active and reachable:

- Call any tool on `juspay-mcp` (e.g. `juspay_get_merchant_details`) — it should return data, not an error.
- Call any tool on `juspay-docs-mcp` — it should respond successfully.

If either server is not responding, check that it was installed correctly in Step 3 and restart the agent/IDE if needed.

---

## Step 6 — Confirm setup

Tell the user:

> "Setup complete. Both MCP servers are authenticated.
>
> Type `/integrate` to start integrating a Juspay payment product into your app."

---

## Re-authentication

If tokens expire later, the user can type `/connect` in Claude Code to re-authenticate without repeating the full setup.
