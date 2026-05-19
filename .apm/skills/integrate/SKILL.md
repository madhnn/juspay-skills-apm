---
name: integrate
description: >
  Execute the /integrate command for LLM agents. Triggers when the user types
  `/integrate`, `/integrate --product`, or asks to "integrate a Juspay product",
  "set up payments", "add payment SDK", or any variation of setting up a Juspay
  product into their app or codebase. This skill drives a fully guided, doc-driven
  wizard: it reads product summaries locally, probes candidates via MCP, then fetches
  actual documentation pages and generates complete integration code.
compatibility:
  tools:
    - juspay-docs-mcp (explore_product, doc_fetch_tool)
    - juspay-mcp (juspay_get_merchant_details, juspay_get_webhook_settings, juspay_update_webhook_settings, juspay_get_general_settings, juspay_update_general_settings, juspay_create_api_key)
  mcp_servers:
    - juspay-docs-mcp
    - juspay-mcp
---

# /integrate — Juspay Integration Orchestrator

> **PRIME DIRECTIVE:** This file is a decision engine. It contains no product knowledge.
> Product knowledge lives in `products/`. Authoritative implementation facts come only from MCP tool calls — never from memory or training.
>
> **MCP PREFERENCE:** Always prefer `juspay-mcp` tools for live merchant data (credentials, settings, gateway config, integration status). Use `juspay-docs-mcp` only for documentation structure and page content.

---

## AGENT SELF-CHECK (run mentally before each phase)


- Did I authenticate with `juspay-mcp` before calling any `juspay-mcp` tools? If not, Please trigger the authentication flow now.
- Did I call `juspay_get_merchant_details` to establish merchant context before asking for credentials?
- Did I read `products/` before calling `explore_product`? Can I conclude from the catalog alone?
- Did I scan the codebase before asking disambiguation questions (language, framework)?
- Did I call `doc_fetch_tool` before writing any code?
- Am I using method names and field names from the fetched docs, not from memory?
- For SDK/web redirect products: did I fetch test resources and run tests for each checklist stage wherever possible

---

## FLAG PARSING

Extract flags before starting:

| Flag              | Effect                                                                                                   |
| ----------------- | -------------------------------------------------------------------------------------------------------- |
| `--product <id>`  | Skip recommendation. Confirm the product in one line, then go to Phase 1. Still run catalog-first check. |
| `--platform <id>` | Hint for platform selection in Phase 2 — still verify against codebase before asking.                    |

---

## UI INTERACTION RULES

Whenever the user must choose between fixed options:

- Use native select / choice UI
- Do NOT ask for free-text replies if options are known
- Wait for a selection before continuing
- Do NOT rephrase the same question again after rendering choices
- **Do NOT ask for information you can derive** — from the codebase, from `juspay-mcp` live data, or from the catalog

Format choices as structured options, not inline prose.

---

## PHASE 0 — Intent Collection and Product Selection

### Step 0A — Load product catalog

Read all files in `products/`. Each file has: product ID, platforms, use cases, and intent signals.

Store the full set as `$PRODUCT_CATALOG`. This is your local knowledge for matching — do not use training-data knowledge about products.

### Step 0B — Auto-resolve integration type from merchant account

Call:

```
juspay-mcp:juspay_get_merchant_details()
```

Extract and store:

- `$MERCHANT_ID` — from the `merchantId` field
- `$CLIENT_ID` — always default to `$MERCHANT_ID`. **Never extract this from the API response.** Inform the user:
  > "Client ID is typically the same as your merchant ID (`$MERCHANT_ID`). If you use a different client ID, please provide it now — otherwise I'll proceed with `$MERCHANT_ID`."
  > Wait for confirmation or a custom value before continuing.
- `$INTEGRATION_TYPE` — from the `integrationType` field, which is an **array**; take the first element (e.g. `["PP"]` → `"PP"`)

Map `$INTEGRATION_TYPE` to a recommended product:

| `$INTEGRATION_TYPE`         | Recommended product ID                       |
| --------------------------- | -------------------------------------------- |
| `PP`                        | `hyper-checkout`                             |
| `ec_sdk`                    | `ec-headless`                                |
| `ec_api`                    | `ec-api`                                     |
| _(anything else or absent)_ | No inference — fall back to Step 0B-Fallback |

### Step 0B-Confirm — Present inferred recommendation

If a mapping was found, present a single confirmation:

> "Based on your account configuration, it looks like you're set up for **[Product Name]**.
>
> Shall I proceed with integrating **[Product Name]**?
>
> 1. Yes, proceed
> 2. No, let me choose a different product type
> 3. No, let me choose a specific product"

- **Option 1** → set `$PRODUCT` to the inferred product ID and skip to Phase 1
- **Option 2** → go to Step 0B-Fallback (product type list)
- **Option 3** → show the full flat product list from `$PRODUCT_CATALOG` and let the user pick directly

### Step 0B-Fallback — Ask the user what they want to build

Only reached if `$INTEGRATION_TYPE` is absent, unrecognized, or the user chose Option 2 above.

> What type of product are you looking to integrate?
>
> 1. **Checkout**
> 2. **UPI**
> 3. **Payouts**
> 4. **Billing**
> 5. **Not sure**

---

### Step 0B1 — Ask the User to Choose a Product

Based on the selected category, ask the relevant follow-up question.

#### If the user selects **Checkout**

> Which Checkout product would you like to integrate?
>
> - Hyper Checkout
> - Express Checkout SDK
> - Express Checkout API

#### If the user selects **UPI**

> Which UPI product would you like to integrate?
>
> - UPI TPAP
> - Hyper UPI

and so on for Payouts and Billing categories...

#### If the user selects **Not sure**

ask:

> Please describe your use case and I'll recommend the right product and integration flow.

Store as `$INTENT`.

### Step 0C — Match intent to candidates

Using `$INTENT` and the `intent signals` field in each `$PRODUCT_CATALOG` entry, select 1–3 products as `$CANDIDATES[]`.

Matching rules:

- "checkout UI", "payment page", "mobile SDK" → prefer products with runtime platforms (android, ios, web, etc.)
- "API only", "server-side", "REST", "backend" → prefer products with no runtime platforms
- "recurring", "subscriptions", "mandates" → billing/mandate products
- "payout", "transfer", "disburse" → payout products
- "UPI", "TPAP", "P2P", "P2M" → UPI products

Aim for 1–3 candidates. Fewer is better.

### Step 0D — Catalog-first product resolution

**Before calling `explore_product`, check if the catalog entry is conclusive:**

A catalog entry is **conclusive** if:

- The `platforms` list either matches `$DETECTED_PLATFORM` exactly, or has only one option
- No further platform disambiguation is required to start code generation

If conclusive → skip `explore_product` for this candidate and proceed.
If **not** conclusive (e.g. multiple overlapping platforms, need page count for complexity signal) → call:

```
juspay-docs-mcp:explore_product({ product: <candidate-id> })
```

Extract only what you need for recommendation:

- Product title
- Platform IDs — runtime IDs signal a client SDK; `docs` only signals a server API; a mix signals both
- Number of numbered base integration pages (complexity signal)
- List of supported platforms if a client SDK is present

Do not fetch individual doc pages here.

### Step 0E — Recommend and confirm

Present your recommendation grounded in what you read from `products/` and `explore_product`:

> "Based on what you described, here's what I recommend:
>
> **[Product Title]** — [one-line reason tied to their intent]
>
> _(Alternative)_ **[Product Title]** — [reason]
>
> Which would you like to integrate? Or pick from the full list below:"

List all products from `$PRODUCT_CATALOG` as a numbered reference so the user can override.

Store the confirmed choice as `$PRODUCT` (the product ID from the products/ file).

---

## PHASE 1 — Full Product Exploration

**Only call `explore_product` if it wasn't already called in Phase 0D for `$PRODUCT`.**

If already called and `$DOC_MAP` is populated → skip directly to Phase 1A.

Otherwise call:

```
juspay-docs-mcp:explore_product({ product: $PRODUCT })
```

Read the full response. This is the authoritative doc structure.

### 1A — Parse into $DOC_MAP

Extract and store:

- Product title and description
- `platforms[]` — every platform entry with its ID and title
- For each platform: `sections[]` → for each section: `sectionTitle`, `pages[]`
- For each page: `pageTitle` and the `md content link` URL

> Pages numbered "1. …", "2. …" are base integration pages in required order. Preserve that order exactly.

### 1B — Classify product type

| Platform IDs observed                                                                                 | Classification             |
| ----------------------------------------------------------------------------------------------------- | -------------------------- |
| Runtime IDs: `android`, `ios`, `web`, `flutter`, `react-native`, `cordova`, `capacitor`, `iframe-web` | `$PRODUCT_TYPE = sdk`      |
| Only `docs`                                                                                           | `$PRODUCT_TYPE = api-only` |
| Mix of `docs` + runtime IDs                                                                           | `$PRODUCT_TYPE = hybrid`   |

Store as `$PRODUCT_TYPE`.

---

## PHASE 2 — Adaptive Flow

Branch on `$PRODUCT_TYPE`:

### If `api-only`

No platform question. Backend language comes from `$DETECTED_LANG` — only ask if not detected.

### If `sdk`

**Step 2-SDK-A — Detect platform from codebase**

Before asking the user, scan the working directory for platform signals:

| File / pattern found                                                              | Detected platform     |
| --------------------------------------------------------------------------------- | --------------------- |
| `pubspec.yaml`                                                                    | `flutter`             |
| `package.json` with `react-native` in dependencies                                | `react-native`        |
| `package.json` with `@capacitor/core` in dependencies                             | `capacitor`           |
| `config.xml` or `package.json` with `cordova` in dependencies                     | `cordova`             |
| `build.gradle` or `AndroidManifest.xml` (no Flutter/RN/Cordova/Capacitor signals) | `android`             |
| `*.xcodeproj` or `Podfile` (no Flutter/RN signals)                                | `ios`                 |
| `package.json` with no mobile framework signals, or `index.html` / `.html` files  | `web` or `iframe-web` |

If a platform is detected with confidence, present it as the pre-selected recommendation:

> "I detected your project is a **[Platform]** app (found `[signal file]`).
>
> Shall I proceed with **[Platform]**?
>
> 1. Yes, use [Platform]
> 2. No, let me choose a different platform"

If the user confirms, skip to disambiguation. If they choose option 2, or if no signal is found, present the full platform list from `$DOC_MAP`.

**Step 2-SDK-B — Disambiguation**

Apply after platform is confirmed:

- Android: ask Java vs Kotlin
- iOS: ask Swift vs Objective-C, CocoaPods vs SPM
- Web: if both `web` and `iframe-web` are in the doc map, ask which variant

Store as `$PLATFORM`. Filter `$DOC_MAP` to the chosen platform's pages.

### If `hybrid`

Ask first:

> "This product has both a backend API and a client SDK. What do you need?
>
> 1. **Backend API only**
> 2. **Client SDK only**
> 3. **Both**"

Then follow the `api-only` path, `sdk` path, or both, as appropriate.

---

## PHASE 3 — Doc Fetch

**Always use `doc_fetch_tool`. Only fall back to WebFetch if MCP returns an explicit error on a valid URL.**

```
juspay-docs-mcp:doc_fetch_tool({ url: "<md content link from $DOC_MAP>" })
```

Fetch order:

1. Pre-Requisites / Overview — always first; defines credentials, auth format, version constraints
2. Numbered base integration pages — in exact numbered order from `explore_product`
3. Webhooks, Order Status API
4. Error Codes (resources section)
5. Advanced sections — only if user asks

While reading each page, extract and store:

- `$PARAMS` — every request field, method param, constructor arg (required vs optional)
- `$CODE_EXAMPLES` — exact method names, class names, key identifiers from the docs
- `$ERROR_CODES` — all status values, error codes, failure reasons
- `$VERSION_CONSTRAINTS` — min SDK version, min language/platform version
- `$WARNINGS` — any "note", "important", "warning" callout blocks

---

## PHASE 4 — Parameter Collection

Tell the user:

> "I've read the documentation. I'll collect what I need."

### Step 4A — Auto-resolve merchant context via MCP

`$MERCHANT_ID`, `$CLIENT_ID`, and `$INTEGRATION_TYPE` were already fetched in Phase 0B — reuse those values. Do not call `juspay_get_merchant_details()` again.

Then call both in parallel:

```
juspay-mcp:juspay_get_webhook_settings()
juspay-mcp:juspay_get_general_settings()
```

From webhook settings, extract:

- `$WEBHOOK_URL` — existing webhook URL if configured (check if non-empty)
- `$WEBHOOK_EVENTS` — currently subscribed events

From general settings, extract:

- `$RETURN_URL` — existing return URL if configured (check if non-empty)

**If `$WEBHOOK_URL` is empty or not configured:**

First, scan the codebase for an existing webhook handler (e.g. `api/juspay/webhook`, `api/webhook`, `webhooks` route). If one exists, note its path as `$WEBHOOK_PATH`.

Then ask the user:

> "No webhook URL is configured. Your app has a webhook handler at `$WEBHOOK_PATH`.
> Please provide your deployed base URL so I can set it to `https://<your-domain>/$WEBHOOK_PATH`.
>
> If you don't have a deployed URL yet, you can:
>
> - Run `ngrok http <port>` locally to get a temporary public URL
> - Leave this for now and configure it on the Juspay dashboard before going live"

Do **not** set a placeholder URL (e.g. `https://www.webhook.com`) — only call `juspay_update_webhook_settings` if the user provides a real, publicly reachable HTTPS URL that routes to the webhook handler.

Once a valid URL is provided, call:

```
juspay-mcp:juspay_update_webhook_settings({
  webHookurl: <base-url> + "/" + <$WEBHOOK_PATH>,
  webhookEvents: <merge existing $WEBHOOK_EVENTS with standard events for this product>
})
```

After updating, confirm what was set:

> "Webhook URL set to `<url>`. The following events are now enabled:
>
> - `EVENT_NAME_1`
> - `EVENT_NAME_2`
> - _(list every event where the value is `true` in the updated config)_"

Store the final URL as `$WEBHOOK_URL`.

**If `$RETURN_URL` is empty or not configured:**

First, scan the codebase for an existing return URL page or handler that can receive the Juspay redirect and handle order status response. If one exists, note its path and use that when asking the user.

Ask the user:

> "No return URL is configured for your account. Please provide the URL customers should be redirected to after payment completes. This must be a real route in your app that can handle the payment return/order status response."

After the user provides a URL, validate that it matches an existing route or handler in the codebase.

- If it matches, call:

```
juspay-mcp:juspay_update_general_settings({ returnUrl: <user-provided URL> })
```

- If it does not match, warn the user:

> "The URL you provided does not appear to exist in the current codebase as a return handler. Are you sure you want to use this?"

Store the final URL as `$RETURN_URL`.

**Environment is always production.** Do not ask the user. Use production host URLs from the docs.

### Step 4B — Auto-provision API key

Do not ask the user for an API key. Instead, call:

```
juspay-mcp:juspay_create_api_key({ description: "integrate-skill-<product>-<date>" })
```

Store the returned plaintext value as `$API_KEY`. Warn the user:

> "A new API key has been created for your account for testing this integration"

Never display the key to the user.

### Step 4C — Collect remaining required params

Ask in order:

1. **Required params** — each required field the user must supply (skip auto-generated fields and anything already resolved: $MERCHANT_ID, $CLIENT_ID, $API_KEY, $WEBHOOK_URL)
2. **Platform version check** (SDK path only) — if docs specify a minimum version
3. **Backend language** — if not already detected from codebase

---

## PHASE 5 — Code Generation

**Rule: use code examples and method names from fetched docs as the base. Substitute collected values. Do not use method or class names you did not see in the docs.**

Generate in order:

1. **Auth / credentials setup** — use environment variables, never hardcode values
2. **Core integration** — API call or SDK install → init → open → response handler
3. **Webhook handler** — if docs have a webhooks section; include signature verification
4. **Status verification utility** — if docs have a status/order API
5. **DB schema** — Read existing codebase to generate a DB schema for storing transaction/order IDs, statuses, and any other relevant info for reconciliation and status checks. Use field names from the docs.
   - Ask the user if they want to see the raw SQL or a Prisma/TypeORM/Mongoose schema based on their detected backend language/framework.
   - If the product has a status API, include fields for storing Juspay order/transaction IDs to correlate with their internal orders.
   - Generate validation rules based on any constraints mentioned in the docs (e.g. max length, required fields).

6. **Error handling** — use error codes from the docs to show how to handle different cases

---

## PHASE 6 — Checklist and Error Reference

### Checklist

Generate a checklist from what you actually fetched — every item must reflect something real in this product's docs.

```
## Integration Checklist — [Product] on [Platform or API]

### Credentials
- [ ] [credential from docs] stored as env var
- [ ] API key generated via dashboard or juspay-mcp

### [Backend / API]
[items derived from API doc pages]

### [Frontend / SDK] (if applicable)
[items derived from SDK doc pages]

### Testing
- [ ] Successful sandbox transaction
- [ ] Error case from $ERROR_CODES tested
- [ ] Webhooks verified end-to-end

### Integration Stages (from Juspay)

Call:

```

juspay-mcp:juspay_integration_monitoring_status({
platform: <$PLATFORM mapped to "Backend" | "Web" | "Android" | "IOS">,
product_integrated: <mapped from $PRODUCT — see mapping below>,
merchant_id: $MERCHANT_ID,
start_time: <30 days ago in YYYY-MM-DDTHH:MM:SSZ>,
end_time: <now in YYYY-MM-DDTHH:MM:SSZ>
})

```

**Product mapping for `product_integrated`:**

| $PRODUCT | product_integrated |
|---|---|
| hyper-checkout | Payment Page Session |
| ec-headless | EC + SDK |
| ec-api | EC Only |
| *(others)* | Payment Page Session |

**Platform mapping for `platform`:**

| $PLATFORM | platform |
|---|---|
| web, iframe-web | Web |
| android | Android |
| ios | IOS |
| flutter, react-native, cordova, capacitor | Android |
| api-only products | Backend |

Render the response as a checklist. For each stage where `visibilityResult` is `true` and `disableStage` is `false`:

```

### [sectionDisplayName or section key]

- [ ] **[stageDisplayName]** ⚠️ Critical ← only if criticalResult is true
      [stageDescription]

```

- Do **not** show `status` — it is not updated in real time
- Mark stages with `criticalResult: true` as ⚠️ Critical
- Group stages by their parent section key

### Go-Live
- [ ] Switched to production environment and keys
- [ ] Production end-to-end test passed
```

### Error Reference

```
## Error Codes

| Code / Status | Meaning | Recommended action |
|---------------|---------|-------------------|
[from $ERROR_CODES]
```

### What's next

Briefly offer to go deeper on sections from `$DOC_MAP` that weren't part of the base integration — but only mention things you actually saw in the doc map.

---

## PHASE 7 — Live Testing

**Always attempt to run the server and test the integration yourself. Do not tell the user to test manually if you can do it.**

### Step 7A — Start the dev server

Scan the codebase for the start command:

| Signal                              | Command                       |
| ----------------------------------- | ----------------------------- |
| `package.json` with `"dev"` script  | `npm run dev` (or `yarn dev`) |
| `pubspec.yaml`                      | Cannot run — skip to Step 7C  |
| Mobile-only project (no web server) | Cannot run — skip to Step 7C  |

Run the server in the background, wait for it to be ready, then proceed.

> **Important:** Shell environment variables override `.env` files in Vite/Node. Before starting the server, check if any required env vars (e.g. `JUSPAY_API_KEY`) are already set in the shell and would conflict with the project's `.env`. Unset them if they don't belong to this project.

### Step 7B — Run backend API tests

For each backend endpoint generated in Phase 5, send a real HTTP request using `curl` and verify both the HTTP response AND any DB/state side-effects:

1. **Session / order creation endpoint** — POST with a real order ID from the DB; expect a payment link back; verify `juspay_order_id` is written to the DB row.
2. **Order status endpoint** — GET with the Juspay order ID from step 1; expect a status response from Juspay.
3. **Webhook endpoint** — POST a synthetic `ORDER_SUCCEEDED` payload using the `juspay_order_id` from step 1; use `curl -w "%{http_code}"` and assert the HTTP status code is **200** (Juspay marks a webhook as "not notified" for any non-200 response and retries with progressive delays); also verify the body contains `{"status":"ok"}` and query the DB to confirm `payment_status` and `juspay_payment_id` were updated.
4. **Webhook endpoint (failure)** — POST a synthetic `ORDER_FAILED` payload; assert HTTP 200 and verify `payment_status = failed` in the DB.

If any test fails:

- Read the server logs for the actual error message
- Diagnose and fix the root cause (wrong env var, bad header, type mismatch, etc.)
- Re-run until the test passes

### Step 7C — Run browser-based integration stage tests (SDK/web redirect products only)

**This step applies when `$PRODUCT_TYPE = sdk` or `$PRODUCT_TYPE = hybrid` and the product uses a Juspay-hosted payment page (web redirect flow).**

The integration checklist stages from Phase 6 (New Card, UPI Collect, UPI Intent, Wallet, etc.) are registered on Juspay's servers only when real transactions flow through Juspay's hosted payment page. Some of these can be tested via curls, call out clearly what you can't test instead of silently skipping.

**Step 7C-1: Fetch test credentials**

fetch the test resources page for the platform:

```
juspay-docs-mcp:doc_fetch_tool({ url: "<test-resources md content link from $DOC_MAP>" })
```

Extract:

- `$TEST_CARDS` — card numbers, expiry, CVV for the Dummy PG / simulator
- `$TEST_UPI_VPA` — VPA values for UPI success/failure (e.g. `success@upi`, `failure@upi`)
- `$DUMMY_PG_FLOWS` — how to trigger success vs failure for each payment method on the simulator

### Step 7D — Report results

After all testing, report a unified pass/fail table covering both backend and browser tests:

```
| Test | Type | Result |
|------|------|--------|
| POST /api/juspay/session → payment link + DB write | Backend | ✅ / ❌ |
| GET /api/juspay/order-status | Backend | ✅ / ❌ |
| POST /api/juspay/webhook ORDER_SUCCEEDED → DB updated | Backend | ✅ / ❌ |
| POST /api/juspay/webhook ORDER_FAILED → DB updated | Backend | ✅ / ❌ |
| Payment Page Opens | Browser | ✅ / ❌ |
| New Card — success | Browser | ✅ / ❌ |
| New Card — failure | Browser | ✅ / ❌ |
| UPI Collect — success | Browser | ✅ / ❌ |
| UPI Collect — failure | Browser | ✅ / ❌ |
| [other visible stages] | Browser | ✅ / ❌ |
```

If a browser test cannot be completed (Juspay payment page blocks headless browsers, CAPTCHA, etc.), say so explicitly — do not silently skip it.

---

## TOOL CALL REFERENCE

| When          | Tool                                        | Purpose                                                                                       |
| ------------- | ------------------------------------------- | ----------x----------------------------------------------------------------------------------- |
| Phase 0A      | Read `products/*.md`                        | Load product summaries for intent matching                                                    |
| Phase 0B      | `juspay_get_merchant_details()`             | Auto-resolve merchant ID, client ID, integration type — infer recommended product             |
| Phase 0D      | `explore_product(candidate-id)`             | Probe type and platforms before recommending                                                  |
| Phase 1       | `explore_product($PRODUCT)`                 | Get full doc structure and page URLs                                                          |
| Phase 3       | `doc_fetch_tool(url)`                       | Fetch individual doc pages for implementation details                                         |
| Phase 4A      | `juspay_get_webhook_settings()`             | Check if webhook URL is already configured                                                    |
| Phase 4A      | `juspay_get_general_settings()`             | Check if return URL is already configured                                                     |
| Phase 4A      | `juspay_update_webhook_settings(...)`       | Configure webhook URL if not already set                                                      |
| Phase 4A      | `juspay_update_general_settings(...)`       | Configure return URL if not already set                                                       |
| Phase 4B      | `juspay_create_api_key(...)`                | Provision a new API key; returned plaintext shown once                                        |
| Phase 6       | `juspay_integration_monitoring_status(...)` | Fetch live integration stages; render as checklist with criticality + description (no status) |
| Fallback only | WebFetch                                    | Only if `doc_fetch_tool` returns an error on a valid URL                                      |

**Never construct doc URLs yourself.** All URLs come from the `md content link` field in `explore_product` responses.

---

## GUARDRAILS

1. **No product knowledge in this file.** Product IDs, platform lists, and credential names come from `products/` or MCP responses — never from this file or training data.

2. **Read `products/` before matching intent.** Do not guess which product fits based on training-data familiarity.

3. **Call `explore_product` before recommending.** Phase 0D is mandatory. You must know the product's type and integration complexity before presenting it as a recommendation.

4. **Never construct doc URLs.** All URLs come from the `md content link` field in `explore_product` responses.

5. **Never fabricate.** If a page didn't load or a section wasn't in the docs, say so. Offer the raw URL for the user to check manually.

6. **Product type comes from `explore_product`.** Do not infer type from the product name or training data.

7. **Platform list comes from `$DOC_MAP`.** Present exactly the platforms that `explore_product` returned — no more, no less.

8. **API-only products never get a platform question.** If `$DOC_MAP` has no runtime platform IDs, skip platform selection entirely.

9. **Code examples come from the docs.** Use the exact method names, class names, and code structure from the fetched documentation pages as your source of truth.

10. **Parameters come from the docs.** The actual required fields are what the fetched pages say.

11. **Code uses doc-sourced names only.** If a method or class name doesn't appear in the fetched pages, do not use it.

12. **Error codes come from the docs.** Collect them from every page you fetch. Do not invent them.
