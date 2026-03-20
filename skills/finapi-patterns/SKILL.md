---
name: finapi-patterns
description: Apply when working with finAPI open banking integration code — bank connections, transactions, payments, web forms, OAuth tokens, SCA flows, or any finAPI/open banking related implementation. Automatically enforces known gotchas and production patterns.
---

You are working with finAPI open banking integration. Before writing or reviewing any code, apply these rules automatically — do not wait to be asked.

## Amounts & Currency

- finAPI returns amounts as **floats in euros**. Always convert to cents on import: `Math.round(transaction.amount * 100)`
- Never pass cent values back to finAPI — it expects floats (euros)
- `endToEndId` for payments: must be unique per order, max 35 chars. Use UUIDv7 stripped of hyphens

## String Sanitization (mandatory before any banking API call)

All free-text fields (counterpartName, purpose, endToEndId) must be sanitized:
1. Replace German umlauts: ä→ae, ö→oe, ü→ue, Ä→AE, Ö→OE, Ü→UE, ß→ss
2. Strip any character not in: `a-zA-Z0-9 / - ? : ( ) . , ' +`
3. Enforce length limits: counterpartName 70 chars, purpose 2000 chars, endToEndId 35 chars

Skipping sanitization causes silent rejections from the bank's backend.

## OAuth Tokens

- Use `client_credentials` grant for admin operations (create/delete finAPI users)
- Use `password` grant for all user-scoped operations (accounts, transactions, payments)
- Cache tokens; refresh proactively 60s before expiry — never let a token expire mid-request
- One finAPI user per company/tenant (UUID-based), not one per end user

## Credential Storage

- Store finAPI username/password encrypted at rest (AES-256-GCM + scrypt key derivation)
- Format: `salt + iv + authTag + ciphertext`, base64-encoded
- Never log or expose finAPI credentials

## Web Form 2.0 Embedding

- The web form token = the web form ID returned by the init endpoint
- Extract `targetUrl` by splitting on `/wf/`: `url.split("/wf/")[0]`
- Web Form renders in **Shadow DOM** — your CSS cannot style it, viewport-based layout detection breaks in embedded contexts (modal, sidebar, panel). Always set `layoutConfig` explicitly: `"xs" | "sm" | "md" | "lg"`
- Call `form.unload()` after `onComplete` / `onFail` / `onAbort` to avoid memory leaks
- Each web form instantiation requires a **fresh token** — never reuse a previous web form ID

## Async Operations & Polling

All finAPI operations are async. Never assume immediate completion:
- Web forms: poll until `COMPLETED | ERROR | ABORTED | EXPIRED`
- Background bank update tasks: poll `/tasks/{id}` until `FINISHED`
- Payments: poll `/payments?ids=` for `statusV2` after web form completes
- Default: 2s interval, 120s timeout for tasks; web forms need longer (user interaction, 10+ min)

## SCA Re-trigger

Background bank connection refresh (`/tasks/backgroundUpdate`) may return `WEB_FORM_REQUIRED` mid-flow. Always handle this case — return the new web form to the frontend instead of treating it as an error.

## Transaction Deduplication

Always use a composite deduplication key — never rely on finAPI transaction IDs alone:
```
finapiId = `${bankId}:${connectionId}:${accountId}:${transactionId}`
```
Insert with `ON CONFLICT (..., finapiId) DO IGNORE` to prevent double-import across refreshes.

## Account Resolution Priority

When resolving which account to use from a bank connection:
1. Previously stored `finapiAccountId` (user's explicit choice)
2. Account matching the stored IBAN
3. First CHECKING account
4. First account in the list

## Error Parsing

finAPI returns errors in two formats — always handle both:
```typescript
// Single error: { message: "..." }
// Array: { errors: [{ message: "..." }, ...] }
const msg = body.message ?? body.errors?.map(e => e.message).join("; ");
```

## Environment Separation

- Sandbox and production use **different profile IDs** (whitelabel templates) — never hardcode, always config
- `localhost` cannot be whitelisted for production CORS — use a real domain even in staging
- Sandbox banks are prefixed with "finAPI Test" and behave differently from real banks

## Security Checklist (review before any PR)

- [ ] finAPI credentials encrypted at rest, never in logs
- [ ] Callback URLs are HTTPS
- [ ] Domains registered with finAPI support for CORS
- [ ] Sanitization applied to all payment string fields
- [ ] `endToEndId` is unique (UUID-based, not sequential)
- [ ] Token cache implemented (no per-request token fetch)
