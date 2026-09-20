---
generated: '2026-09-19'
method: generated
name: Open a workspace and read your masked deal flow (tier 2)
description: After an accepted manifest, register a NEW workspace UUID, read the agreement, and read matches, deal flow, valuation and pairings — names and granular metrics stay masked until a human principal signs the Terms.
api: openapi/capepartners-fr-openapi.yml
operations: ["POST /api/workspace/join", "GET /api/tos/text", "POST /api/session/{session_id}", "GET /api/session/{session_id}", "GET /api/matches/{session_id}", "GET /api/deal-flow/{session_id}", "GET /api/valuation/{session_id}", "GET /api/pairings/{session_id}", "GET /api/activity/{session_id}", "POST /api/nda/sign"]
operation_ids: overlay-assigned (overlays/capepartners-fr-openapi-overlay.yaml) — the provider spec declares no operationId; every step below also cites the verbatim METHOD + PATH from openapi/capepartners-fr-openapi.yml.
source: >-
  Grounded in openapi/capepartners-fr-openapi.yml (each METHOD + PATH verified verbatim), a2a/capepartners-fr-agent-card.json,
  https://www.capepartners.fr/agent-exchange.html, https://www.capepartners.fr/api/exchange/spec and https://www.capepartners.fr/llms.txt.
  Cross-cutting rules cite authentication/, conventions/, errors/ and rate-limits/. No value below is invented: limits, states,
  error strings and field names are the provider's own.
---

# Open a workspace and read your masked deal flow (tier 2)

The upgrade from the exchange inbox. A workspace UUID is a **capability token carried in the URL path** — the provider's homepage says to treat it like a password; anyone holding it can read name, email, company and financials. Never log it and never let it into a `Referer`.

## Preconditions
- An **ACCEPTED** manifest (see the tier-1 skill). An agent joining without one gets `403 handshake_required` and keeps tier 1.
- A **verified business email** — consumer webmail (gmail/outlook/hotmail/yahoo) is rejected.
- Send **no `Origin`/`Referer`**; a foreign one is refused 403. UUIDs must be well-formed v4 or the guard answers 400 before anything is touched.

## Steps
1. **Join with a NEW uuid** — `postApiWorkspaceJoin` (`POST /api/workspace/join`) with `{"uuid": "<fresh UUID v4>", "name", "email", "company", "exchange_key": "<your msgid>"}`. A join never takes over an existing workspace (`403 workspace_not_yours`). The workspace is named for the agent, your organisation is carried as an unverified claim, and no supervisor is bound yet. Response: `session_id` / `uuid`.
2. **Read the agreement first** — `getApiTosText` (`GET /api/tos/text`) (canonical) or its back-compat alias `getApiNdaText` (`GET /api/nda/text`). This is the single governing agreement (ToS v1.0, effective 8 September 2026, French law).
3. **Set the profile** — `postApiSessionBySessionId` (`POST /api/session/{session_id}`) with `identity` (`name`, `email`, `company`, `role`) and either `buyer` (`sector`, `check_size_min/max`, `geography`, `growth_target`, `ebitda_target`, `solution_1..3`) or `seller` (`revenue`, `growth`, `ebitda_margin`, `product`, `sector`, `description`). The write is a COALESCE upsert: existing values are kept where you send nothing. Read it back with `getApiSessionBySessionId` (`GET /api/session/{session_id}`) — it returns `data_quality` and `missing_fields`.
4. **Read matches** — `getApiMatchesBySessionId` (`GET /api/matches/{session_id}`) (`limit` optional). Pre-signature you get redacted names and coarse fit bands (strong / moderate / weak / poor). Retrieving the list records a suggestion count for each counterparty (idempotent per pair per day) but **never creates a pairing**.
5. **Read your own deal flow** — `getApiDealFlowBySessionId` (`GET /api/deal-flow/{session_id}`) (`refresh`, `window`): the whole scored universe over the fit threshold 50, masked; `top`, `fresh` (sourced / enriched), `universe` bands, `watchlist`. Mirror: `getApiInterestSignalsBySessionId` (`GET /api/interest-signals/{session_id}`) for inbound interest about your own entity (counts only).
6. **Valuation and pairings** — `getApiValuationBySessionId` (`GET /api/valuation/{session_id}`) (seller only; `status: no_data` when financials are not verified — the provider does not fabricate a number) and `getApiPairingsBySessionId` (`GET /api/pairings/{session_id}`) (`phase`, `match_score`, any `phase_request` with `expires_at`).
7. **Check the To Do** — `getApiActivityBySessionId` (`GET /api/activity/{session_id}`) returns `{events, todo, nda_signed}`; `todo[]` names what needs attention (agreement review, profile gaps, parked suggestions, phase changes awaiting confirmation).
8. **Signing is a human step** — `postApiNdaSign` (`POST /api/nda/sign`) records a signature, but an agent-initiated signature stays *pending* until the declared human supervisor approves the emailed link; only then do `GET /api/matched-names/{session_id}`, `GET /api/search/{session_id}` and `GET /api/infomemo/{session_id}` stop answering `403 nda_required:true`. Do not loop on the gate — it does not move by polling.

## Rate limits and errors
- 60 requests / 60 s per IP per endpoint family on confidential reads → 429 with `retry_after` in the body. 403 is the guard family (cross-origin, Turnstile, handshake, NDA, scope); 404 covers "no buyer or seller for this session" as well as unknown ids. See `errors/capepartners-fr-problem-types.yml`.

## Reversibility
- Watchlist pins are reversible (`postApiWatchlist` (`POST /api/watchlist`), `action: add|remove`). A pairing phase change proposed with your `session_id` is a **request** the counterparty confirms/dismisses; an unanswered request lapses after the consent TTL (default 14 days). There is no un-sign and no undo for a submitted manifest. Details in `conventions/capepartners-fr-conventions.yml`.
