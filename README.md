# claude-code-finapi-plugin

A [Claude Code](https://claude.ai/code) plugin for **finAPI open banking** integration — built from production experience implementing PSD2-compliant bank connections, transaction import, and SEPA payment initiation in DACH markets.

## What's included

### Agent: `finapi-openbanking`

A specialist agent Claude invokes for complex finAPI tasks. Covers:

- OAuth token management (client_credentials vs password grant, token caching)
- Bank connection lifecycle via Web Form 2.0 (init → embed → finalize → refresh)
- Transaction import with pagination and deduplication
- SEPA payment initiation with string sanitization
- SCA (Strong Customer Authentication) re-trigger handling
- Polling patterns for all async operations
- Error parsing (single error vs array format)
- Account resolution priority logic
- Production readiness checklist
- Full API reference table

### Skill: `finapi-patterns`

An auto-invoked skill that fires whenever Claude detects finAPI-related code. Silently enforces:

- Amount conversion (finAPI floats → cents)
- String sanitization rules (umlauts, character allowlist, length limits)
- Token grant type selection
- Credential encryption requirements
- Web Form embedding gotchas (Shadow DOM, `layoutConfig`, `targetUrl` extraction)
- Async polling requirements
- Transaction deduplication key pattern
- Security checklist before PRs

## Installation

```bash
# Install directly from GitHub
claude plugin install https://github.com/urakozz/claude-code-finapi-plugin
```

Or clone and install locally:

```bash
git clone https://github.com/urakozz/claude-code-finapi-plugin
claude plugin install ./claude-code-finapi-plugin
```

## Usage

Once installed, the skill fires automatically when you work on finAPI code. To explicitly invoke the agent:

```
use the finapi-openbanking agent to implement bank connection refresh with SCA handling
```

Or reference it in conversation — Claude will pick it up from the description.

## Key gotchas this plugin prevents

| Gotcha | Rule enforced |
|--------|--------------|
| Wrong amount unit | finAPI returns euros (float); always `Math.round(amount * 100)` for cent storage |
| Silent payment rejection | Sanitize all strings: umlauts → ASCII, strip unsupported chars, enforce length limits |
| Token expiry mid-request | Cache tokens, refresh 60s before expiry |
| Broken embedded web form | Set `layoutConfig` explicitly — Shadow DOM breaks viewport detection |
| Double transaction import | Use composite dedup key: `bankId:connectionId:accountId:transactionId` |
| SCA treated as error | `WEB_FORM_REQUIRED` during background update is expected — return new web form |
| Wrong grant type | `client_credentials` for admin, `password` for user-scoped operations |

## Resources

- [finAPI Access API](https://docs.finapi.io)
- [Web Form 2.0 docs](https://documentation.finapi.io/webform/embedded-web-form-2-0)
- [finAPI npm package](https://www.npmjs.com/package/@finapi/web-form)

## License

MIT
