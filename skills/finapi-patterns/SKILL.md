---
name: finapi-patterns
description: Apply when working with finAPI open banking integration code — bank connections, transactions, payments, web forms, OAuth tokens, SCA flows, or any finAPI/open banking related implementation. Automatically enforces known gotchas and production patterns.
---

You are working with finAPI open banking integration. Before writing or reviewing any code, apply these rules automatically — do not wait to be asked.

## Amounts & Currency

- finAPI returns amounts as **floats in the account's currency** (e.g. `12.50` = 12.50 of whatever currency the account holds — EUR, GBP, CZK, PLN, etc.). Be deliberate about how your system stores them — common choices are floats, decimal strings, or integer minor units. Pick one and convert consistently on the boundary.
- When sending amounts back to finAPI (e.g. payments), always send floats — not integers, not strings.
- Document the unit convention clearly wherever money crosses a system boundary.

## String Sanitization (mandatory before any banking API call)

All free-text fields (counterpartName, purpose, endToEndId) must be sanitized before sending to finAPI:
1. Replace characters your bank stack may reject — German umlauts are a common source of silent failures: ä→ae, ö→oe, ü→ue, Ä→AE, Ö→OE, Ü→UE, ß→ss
2. Strip unsupported characters — most banking stacks only allow: `a-zA-Z0-9 / - ? : ( ) . , ' +`
3. Enforce length limits: counterpartName 70 chars, purpose 2000 chars, endToEndId 35 chars

Skipping sanitization causes **silent rejections** from the bank's backend with no useful error message.

## OAuth Tokens

- Use `client_credentials` grant for admin/server-to-server operations (create/delete finAPI users)
- Use `password` grant for user-scoped operations (accounts, transactions, payments)
- Cache tokens and refresh proactively before expiry — never let a token expire mid-request
- One finAPI user per tenant/company (UUID-based), not one per end user

## Credential Storage

- finAPI user credentials must be stored encrypted at rest — never in plaintext
- Choose an encryption scheme appropriate for your platform and security requirements
- Never log or expose finAPI credentials in error messages, traces, or responses

## Web Form 2.0 Embedding

- The web form **token = the web form ID** returned by the init endpoint
- Extract `targetUrl` by splitting on `/wf/`: `url.split("/wf/")[0]`
- Web Form renders in **Shadow DOM** — your CSS cannot style it, and viewport-based layout detection breaks in embedded contexts (modal, sidebar, panel). Always set `layoutConfig` explicitly: `"xs" | "sm" | "md" | "lg"`
- Call `form.unload()` after `onComplete` / `onFail` / `onAbort` to avoid memory leaks
- Each web form instantiation requires a **fresh token** — never reuse a previous web form ID

## Async Operations & Polling

All finAPI operations are async — never assume immediate completion:
- Web forms: poll until `COMPLETED | ERROR | ABORTED | EXPIRED`
- Background bank update tasks: poll `/tasks/{id}` until `FINISHED`
- Payments: poll `/payments?ids=` for `statusV2` after web form completes
- Recommended defaults: 2s interval, 120s timeout for tasks; web forms need longer (user interaction can take 10+ min)

## SCA Re-trigger

Background bank connection refresh (`/tasks/backgroundUpdate`) may return `WEB_FORM_REQUIRED` mid-flow. Always handle this explicitly — surface a new web form to the user instead of treating it as an error.

## Transaction Deduplication

Always use a composite deduplication key — never rely on finAPI transaction IDs alone, as IDs can change across refreshes:
```
finapiId = `${bankConnectionId}:${accountId}:${transactionId}`
```
Use this key to prevent double-import when re-fetching transactions.

## Account Resolution Priority

When a bank connection has multiple accounts, resolve to the intended one via a consistent priority:
1. Previously stored account ID (user's explicit choice)
2. Account matching a stored IBAN
3. Account matching expected type (e.g. CHECKING)
4. First account as fallback

## Error Parsing

finAPI returns errors in two formats — always handle both:
```typescript
// Single error: { message: "..." }
// Array: { errors: [{ message: "..." }, ...] }
```

## Environment Separation

- Sandbox and production use **different profile IDs** (whitelabel templates) — always load from config, never hardcode
- `localhost` cannot be whitelisted for production CORS — use a real domain even in staging
- Sandbox test banks behave differently from real banks — verify flows in production-like environments before go-live

## Security Checklist (review before any PR)

- [ ] finAPI credentials encrypted at rest, never in logs or error responses
- [ ] Callback URLs use HTTPS
- [ ] Domain whitelisted with finAPI support for CORS
- [ ] All payment string fields sanitized
- [ ] `endToEndId` is unique per payment (UUID-based recommended)
- [ ] Token caching implemented — no per-request token fetch
- [ ] SCA re-trigger (`WEB_FORM_REQUIRED`) handled explicitly
