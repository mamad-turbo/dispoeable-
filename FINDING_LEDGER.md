# FINDING LEDGER — Arc L1 (arc-bbp) — CAP v2 Chaining Engine

Maintained 2026-09-21. Rules: nothing deleted; none/info/low entries go to the Chaining Queue, never closed; every entry answers the Chaining Question before parking.

## LEDGER ENTRIES

### L-01 Portal anonymous wallet-data endpoints
- COMPONENT: portal.arc.io /api/balances, /api/gateway/balances, tx-history/activities
- PRECONDITIONS: none (anonymous)
- OBSERVED: full balance/tx data for any wallet address without auth; values matched Base RPC balanceOf exactly
- DEVIATION: none — public blockchain state mirrored
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: read public on-chain state (not secret); confirms portal mirrors public data
- CAPABILITIES REQUIRED: none
- STATUS: disproven (public data; adversarial review agreed)

### L-02 Portal /api/feature-flags anonymous (29,812 B)
- COMPONENT: portal.arc.io /api/feature-flags
- PRECONDITIONS: none
- OBSERVED: f1-f17; f14-f17 = swap-route config tables across 17 chains (MONAD, PLUME, INK, HYPEREVM, SONIC, UNI, WORLD, XDC, SEI, MATIC, LINEA, ARB, OP, AVAX, BASE, ETH, ARC); f3.enabled=false + empty description; f11/f12/f13=false; no secrets/URLs
- DEVIATION: anonymous full config dump
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: chain/token route inventory (public config); reveals pre-live chains
- STATUS: parked (info-only; no secret material)

### L-03 Portal POST /api/analytics anonymous echo
- COMPONENT: portal.arc.io /api/analytics
- PRECONDITIONS: none
- OBSERVED: POST arbitrary JSON -> 200 {"message":"ok"}
- DEVIATION: unauthenticated write sink
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: write telemetry rows (log pollution only)
- STATUS: parked (no read-back, no victim-visible effect; telemetry-by-design)

### L-04 Portal SwapKit rates constant 400 "SwapKit upstream returned an error"
- COMPONENT: /api/swapkit/v1/stablecoinKits/rates (chain=ARC|ETH|BASE)
- PRECONDITIONS: none
- OBSERVED: identical 400 regardless of input
- DEVIATION: none per se; reveals upstream dependency behavior
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: confirms external SwapKit upstream; uniform error = likely upstream config/key issue (feature broken, not vulnerable)
- STATUS: parked (info)

### L-05 Portal /api/wallets/register anonymous
- COMPONENT: /api/wallets/register
- PRECONDITIONS: none
- OBSERVED: unauth POST accepted; no durable victim-visible effect proven
- DEVIATION: unauthenticated state-changing-looking endpoint
- STANDALONE SEVERITY: none (no proven impact)
- CAPABILITIES GRANTED: potential telemetry/log-row write for arbitrary wallet address
- STATUS: parked (impact unproven; adversarial review: not reportable)

### L-06 Entity-secret differential error oracle
- COMPONENT: studio.arc.io GET /api/entity-secret?appId&envVar[&probe=1]
- PRECONDITIONS: authenticated (any identity incl. CLI PAT)
- OBSERVED: foreign/random appId -> 404 {"message":"App not found."}; OWN appId w/o registered secret -> 404 {"message":"Entity secret not found."}; owner with secret would get 200 {secret,recoveryFile,recoveryFileName}
- DEVIATION: error text differentiates app-ownership state from secret-registration state
- STANDALONE SEVERITY: none (oracle only reveals caller's OWN app state)
- CAPABILITIES GRANTED: for the caller's own apps — confirms whether an entity secret is registered; no cross-tenant info (foreign ids collapse to "App not found")
- CHAINING Q: with entity-secret retrieval (owner) + registered secret -> wallet API key. With a registered secret, owner (session OR CLI PAT) can read the raw secret. Consent claims "create and modify apps" — wallet-key retrieval is arguably beyond that mental model but owner-scoped; no cross-user impact. Needs a registered secret (AI quota) to prove full retrieval.
- STATUS: open (chain L-06 + L-14; blocked on quota for full proof)

### L-07 401 message differential (token-state oracle)
- COMPONENT: studio.arc.io API middleware (all /api/*)
- PRECONDITIONS: none
- OBSERVED: no token -> 128B {"code":12,"message":"No authorization token provided","redirect":"/logout"}; present-but-invalid/expired token -> 122B {"code":12,"message":"Unauthorized - Access denied","redirect":"/logout"}
- DEVIATION: message distinguishes absent vs invalid token
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: for a candidate token/cookie: distinguishes "not sent" from "rejected" (session-validity oracle for stolen/guessed cookies)
- STATUS: parked (info; useful for future session-hijack testing)

### L-08 Session cookie rotation/expiry (~30 min via API-only usage)
- COMPONENT: studio.arc.io __session cookie; CLI adoptRefreshedCookie
- PRECONDITIONS: authenticated session
- OBSERVED: session dies ~30min when driven via API-only (fetchIso); server refreshes __session on responses (CLI stores refresh); UI usage keeps alive
- DEVIATION: short-lived rotating session cookie
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: session rotation limits stolen-cookie lifetime to ~30min (defensive); ARC_STUDIO_COOKIE / login --cookie accepts cookie as credential (documented)
- STATUS: parked (defensive; cookie-as-credential is by design)

### L-09 Stale session cookie poisons fresh login (auth-flow state primitive)
- COMPONENT: studio.arc.io signup?EXISTING_ACCOUNT + auth.circle.com PKCE exchange
- PRECONDITIONS: browser with a pre-existing (stale) __session cookie
- OBSERVED: fresh login fails silently (redirect to /, no session) when a stale cookie is present; clearCookies fixes it
- DEVIATION: existing session state breaks the re-login exchange
- STANDALONE SEVERITY: none (self-DoS only)
- CAPABILITIES GRANTED: none useful to attacker (cannot force a victim to hold a stale cookie)
- STATUS: parked (disproven as finding; harness root cause documented)

### L-10 CLI PAT (origin_pat_*) lifecycle + scope
- COMPONENT: /api/cli-tokens (create/list/revoke), Authorization: Bearer origin_pat_*
- PRECONDITIONS: mint needs ACTIVE BROWSER SESSION (PAT cannot mint -> 403 "requires an active browser session"); 3-month expiry; revoke immediate
- OBSERVED SCOPE: reads apps/threads/messages/sandbox-files/git-info/deployments; writes app title; EXEC arbitrary shell in sandbox (SSE); mints 1h preview JWT; reaches entity-secret (owner-scoped); 401 GitHub; 403 foreign sandbox "Sandbox not found"; no portal/agentvm effect
- DEVIATION: none — scope matches consent exactly
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED (if stolen): full build+exec+file control of the token owner's apps; NOT other users'; not GitHub; not token minting; 3-month window; lastUsedAt visible to owner
- CHAINING Q: PAT (identity) + terminal exec (input) = sandbox control of OWN app (by design). PAT + entity-secret = wallet key of OWN app (untested full retrieval). PAT + preview JWT = 1h preview access (owner's own). No cross-user primitive found.
- STATUS: open (chain L-10 + L-06; no cross-user impact proven; token REVOKED after tests)

### L-11 Preview JWT in URL (token-in-query anti-pattern)
- COMPONENT: /api/sandbox?appId -> authedPreviewUrl?_preview_token=<JWT>; JWT header claims {sandboxId,userId,iat,exp}; 1h expiry
- PRECONDITIONS: authenticated owner/PAT
- OBSERVED: signed 1h token placed in URL query string of the preview host
- DEVIATION: security-token-in-URL (leaks via Referer/access logs/history)
- STANDALONE SEVERITY: info
- CAPABILITIES GRANTED: holder can render the authed preview of that sandbox for 1h
- CHAINING Q: if the preview page loads third-party resources (fonts/analytics/ads), the Referer leaks the token to third parties -> 1h preview access. Test: render authed preview in browser, capture third-party request Referers. Impact even if proven: self-owned preview (static test page = nothing sensitive); Ladder rung 4 max for an app with sensitive runtime data. Needs browser (CF-blocked) + app rendered.
- STATUS: open (test pending)

### L-12 Okta flow hardening (no issue) + identifiers
- COMPONENT: /auth/okta/redirect?idp=; login.circle.com/oauth2/ausjngwm9gGd5l8vG5d6; client 0oa9v0plfmQsqwzPS5d7; scope openid email profile offline_access test live
- PRECONDITIONS: none
- OBSERVED: PKCE S256 + random state + FIXED redirect_uri /auth/okta/callback + server-side idp whitelist (bogus -> 302 home)
- DEVIATION: none (sound)
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: offline_access implies refresh-token issuance (untested — needs real Okta code via Google sign-in, user's domain); org/client IDs public
- STATUS: parked (offline_access refresh-token handling untested)

### L-13 Connect endpoints (github/netlify/hf) auth-gated
- COMPONENT: POST /auth/github/connect, /auth/netlify/connect, /api/hfspaces/connect, disconnects
- PRECONDITIONS: session
- OBSERVED: 401 anonymous; 200 session -> OAuth authorize URL with fixed redirect_uri + state; netlify disconnect no-op success
- DEVIATION: none
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: initiate OAuth connect for own account
- STATUS: parked

### L-14 Sandbox terminal/file exec (owner-scoped)
- COMPONENT: /api/sandbox-terminal (SSE), /api/sandbox-file-* 
- PRECONDITIONS: session or PAT; sandboxId must resolve to caller's app
- OBSERVED: owner+PAT exec OK (echo, pwd); foreign/random sandboxId -> 403 {"message":"Sandbox not found."}; file APIs sandboxId-only shape valid (CLI client)
- DEVIATION: none
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: exec in own sandbox (by design)
- STATUS: parked (foreign exec = proxy-disproven: PAT random sandbox 403; B foreign apps 404)

### L-15 RPC surface (testnet)
- COMPONENT: rpc.testnet.arc.network, rpc.drpc.testnet.arc.network, rpc.blockdaemon.testnet.arc.io
- PRECONDITIONS: none
- OBSERVED: eth_chainId 0x4cef52; eth_ namespace public-read open; admin/personal/txpool/debug/trace rejected (-32601); WS handshake 101 then silent drop (inert edge); www.rpc.blockdaemon + rpc2.quicknode DNS dead; blockdaemon GET -> 405
- DEVIATION: none (standard public RPC lockdown)
- STANDALONE SEVERITY: none
- STATUS: parked

### L-16 arc-remote-signer: no app-layer auth (by design)
- COMPONENT: circlefin/arc-remote-signer gRPC 0.0.0.0:10340 default, TLS off, reflection on; local PoC signed arbitrary messages
- PRECONDITIONS: network reachability to a signer deployment
- OBSERVED: no auth on the signing service; TCP 10340 CLOSED/FILTERED on every known testnet host (no public exposure found)
- DEVIATION: service with no auth by default (design); reachability unproven
- STANDALONE SEVERITY: none (no reachable deployment)
- CAPABILITIES GRANTED (IF a deployment were reachable): sign arbitrary messages with validator keys (equivocation potential)
- CHAINING Q: discovery of ANY publicly reachable signer (new host/subdomain/DNS) instantly activates this. Re-scan DNS/subdomains for signer hosts before concluding.
- STATUS: open (chain trigger = new host discovery)

### L-17 GitHub integration boundaries
- COMPONENT: /api/github-repos, github-clone/push/pull/unlink, /auth/github/connect
- PRECONDITIONS: session; connected account
- OBSERVED: owner clone+push verified (commit 18edf21); B foreign -> 404 App not found; B github-repos -> 401 GitHub not connected; PAT -> 401 GitHub not connected
- DEVIATION: none
- STANDALONE SEVERITY: none
- STATUS: parked

### L-18 agentvm invite gate
- COMPONENT: agentvm.arc.io /auth/verify {invite_code}, /request-access -> external Google Form
- PRECONDITIONS: valid invite code
- OBSERVED: invalid codes -> generic error; request-access = Google Form waitlist (302)
- DEVIATION: none
- STANDALONE SEVERITY: none
- STATUS: parked (brute-force excluded class)

### L-19 CLI npm client review (all clean)
- COMPONENT: @circle-fin/arc-studio-cli v1.1.3
- OBSERVED: token storage 0o600/keychain; device-link ephemeral port+state+fragment-only; sanitize.js strips ANSI/OSC/C0 (OSC52-aware); sensitive-files guard (.env/.npmrc/.netrc/keys); api-url allowlist studio.arc.io+studio-staging.arc.io+loopback; postinstall benign
- DEVIATION: none
- STANDALONE SEVERITY: none
- STATUS: parked (no bypass found in sanitize regex or allowlist)

### L-20 Studio endpoints from CLI client (whoami/usage/sandboxId-only file-list/chatDetached)
- COMPONENT: GET /api/whoami (contextFileLimits), GET /api/usage (percentUsed/windowMs), /api/messages pageAfter, /api/sandbox-file-list?sandboxId= (no appId), POST /api/chat {detached:true} -> 202
- PRECONDITIONS: session/PAT
- OBSERVED: anonymous 401; owner-side 200 NOT YET OBSERVED (session died; CF challenge blocked re-login)
- DEVIATION: none known yet
- STANDALONE SEVERITY: none (expected)
- CAPABILITIES GRANTED (if 200): account limits + usage/quota oracle
- STATUS: open (needs fresh session)

### L-21 Differential foreign-app error across APIs
- COMPONENT: all /api/apps-scoped Studio endpoints
- PRECONDITIONS: authenticated non-owner
- OBSERVED: foreign appId -> 404 "App not found." (threads, github ops, entity-secret); foreign sandboxId -> 403 "Sandbox not found." (terminal, files); no timing/length differential observed
- DEVIATION: none (consistent enforcement)
- STANDALONE SEVERITY: none
- CAPABILITIES GRANTED: none beyond L-06 (owner-state only)
- STATUS: parked

### L-22 /api/feedback 422 (feedbackKey+runId required)
- COMPONENT: POST /api/feedback
- PRECONDITIONS: session
- OBSERVED: missing feedbackKey/runId -> 422 structured errors
- DEVIATION: none (stricter than guessed shape)
- STANDALONE SEVERITY: none
- STATUS: parked

### L-23 localStorage "setItem" key holds function source
- COMPONENT: studio.arc.io localStorage
- OBSERVED: key "setItem" = storage-sync monkey-patch function source
- DEVIATION: app artifact, not user data; no secret
- STANDALONE SEVERITY: none
- STATUS: parked

## CHAINING QUEUE (none/info/low — not closed)

| Entry | Class | Why parked (chaining question answered) |
|---|---|---|
| L-01 | info | public chain data; no consumer |
| L-02 | info | config dump; no consumer |
| L-03 | input | telemetry sink; no read-back |
| L-04 | info | upstream misconfig behavior; no consumer |
| L-05 | state | unauth write w/o proven effect |
| L-06 | info/identity | own-app oracle only; chain needs L-10 + registered secret (quota) |
| L-07 | identity | token-validity oracle; useful for future stolen-cookie tests only |
| L-08 | identity | defensive rotation; cookie-as-credential by design |
| L-09 | state | self-DoS only |
| L-10 | identity/trust | scope=consent; no cross-user primitive; chain L-10+L-06 open (quota) |
| L-11 | info/trust | Referer leak hypothesis OPEN (browser blocked) |
| L-12 | identity | offline_access refresh-token handling untested (user Google sign-in) |
| L-13 | trust | auth-gated; no deviation |
| L-14 | trust | foreign exec proxy-disproven |
| L-15 | info | public RPC lockdown |
| L-16 | trust | signer reachability = chain TRIGGER; re-scan before close |
| L-17 | trust | boundaries held |
| L-18 | rate | excluded class |
| L-19 | input | no bypass found |
| L-20 | info | needs fresh session (CF-blocked) |
| L-21 | info | consistent enforcement |
| L-22 | info | stricter than expected |
| L-23 | info | artifact |

## PRIMITIVE-CLASS COVERAGE
- INFO LEAKS: strong (L-02, L-04, L-06, L-07, L-15, L-20, L-21)
- IDENTITY/SESSION: strong (L-06, L-07, L-08, L-10, L-12)
- INPUT-HANDLING: THIN (L-03, L-19 only — no reflection/encoding/parser surface exercised; the JS-diff tooling was not applied to APIs)
- STATE/WORKFLOW: thin (L-05, L-09; race/multi-step flows not probed — chat lease was, held)
- TRUST-BOUNDARY: strong (L-10, L-11, L-13, L-14, L-16, L-17)
- RATE/RESOURCE: zero findings (rate-limit-only excluded per program; no missing-limit observed)

## OPEN CHAINS (highest ladder rung PROVEN per chain)
1. L-10 + L-06 (PAT -> entity-secret retrieval): rung 7 max (own-resource), UNPROVEN retrieval (no registered secret); needs quota. PROVEN: L-10 alone rung 7 on own sandbox (by design).
2. L-11 preview Referer leak: rung 4 max HYPOTHESIS; needs browser render test (CF-blocked).
3. L-16 signer: rung 11 IF reachable; reachability unproven; trigger = host discovery.
4. L-12 offline_access: unknown; needs real Okta code (user Google sign-in).
5. L-20 whoami/usage: rung 1; needs fresh session.

## RE-TRIGGER CHECKS (v2 sec. 2.4)
- New identity available (CLI PAT): swept major endpoints; GAP = messages/feedback/analytics/template-load/sandbox-pause/dev-server-restart/hfspaces-disconnect/github-repo NOT PAT-tested (token revoked; needs fresh mint -> browser).
- New route/version: none since last sweep.
- New host: signer-trigger L-16 — re-run subdomain DNS check for signer/vault/validator hosts before termination.
- New trust boundary: Netlify/HF OAuth (connect initiation tested; connected flows need user accounts).

## CYCLE 4c ADDITIONS (2026-09-21)

### L-24 Portal SwapKit proxy fully mapped
- COMPONENT: /api/swapkit/v1/stablecoinKits/rates (GET; chain+addresses query params per SDK JS)
- OBSERVED: GET correct shape (incl. real USDC-on-Base address) -> 400 "SwapKit upstream returned an error" ALWAYS; POST -> 400 "Invalid SwapKit request" (method not accepted); two distinct error semantics (local validation vs upstream rejection)
- DEVIATION: none exploitable; upstream (SwapKit) rejects every request = feature broken server-side
- STANDALONE SEVERITY: none; STATUS: parked (info; supersedes earlier partial probes)

### L-25 Portal /api/analytics unbounded sink
- COMPONENT: portal.arc.io /api/analytics (anonymous)
- OBSERVED: 200 {"message":"ok"} for ANY content-type (text/plain, form-urlencoded, utf-16), non-JSON garbage, 500KB payload
- DEVIATION: no validation, no evident size limit, no auth
- STANDALONE SEVERITY: none (storage/bandwidth abuse = excluded class per program: DoS + rate-limit-only excluded)
- CAPABILITIES GRANTED: unlimited anonymous writes to analytics store (chain component only)
- STATUS: parked (excluded class; noted as chain component)

### L-26 Portal /api/wallets/register anonymous (confirmed working)
- COMPONENT: portal.arc.io /api/wallets/register (anonymous)
- OBSERVED: {walletAddress, blockchain} -> 200 {"ok":true,"walletAddress":...} for any valid address+chain; missing/bad fields -> clean 400; type-checked
- DEVIATION: unauthenticated registration of arbitrary public addresses (indexer tracking state only; public chain data; re-registration unauthenticated so no front-running denial)
- STANDALONE SEVERITY: none; STATUS: parked (upgraded from "no durable effect proven" to "works, no impact")

### L-27 RPC client fingerprint (Rust/alloy)
- COMPONENT: rpc.testnet.arc.network malformed-param errors
- OBSERVED: "odd number of digits at line 1 column 5", "invalid digit found in string", "hex string without 0x prefix at line 1 column 5" (Rust std + alloy hex-parsing phrasing); structured JSON errors, NO stack traces/paths/versions
- DEVIATION: none; confirms RPC served by Arc's own Rust node stack (arc-node family, in-scope and already deep-reviewed clean)
- STANDALONE SEVERITY: none; STATUS: parked

### L-16 RE-TRIGGER CHECK (mandated by v2 sec 2.4): NEGATIVE
- Re-ran subfinder over arc.io + arc.network; new names = explorer.testnet.arc.io (known), help.arc.io (out-of-scope), www.rpc.blockdaemon.testnet.arc.network (NXDOMAIN); TCP 10340 closed on ALL live testnet hosts incl. explorer. No reachable signer. Chain trigger remains dormant.

## CYCLE 4d ADDITIONS (2026-09-21) — docs-driven host discovery + onramp family

### L-28 Onramp widget family (onramp.arc.io / onramp-sandbox.arc.io / onramp-demo.arc.io)
- COMPONENT: Next.js onramp widget (Circle Onramp; Transak + BVNk providers; Socure KYC; Datadog RUM)
- API SURFACE: anonymous = /api/v1/onramp/location (geo), /api/v1/onramp/payment-method-eligibility (providers/limits config), /api/csp-report (204 sink); session-gated = /api/v1/consumers/lookup {phoneNumber} (401 without session; phone-OTP-bound consumer context -> SELF-scoped prefill, not a cross-user oracle), /api/v1/payment/onramp-quote (401 anon), /api/v1/kyc/status/stream (401 "Missing session token"), /api/v1/evaluation/*, /api/v1/bankTransfer/* (incl demo-bank-transfer-simulate-payin, session-gated)
- OBSERVED: no anonymous PII; no embedded API keys in widget/demo chunks; referrer-policy strict-origin-when-cross-origin on ALL onramp hosts -> sessionToken-in-launch-URL CANNOT leak to Transak/Socure via Referer (cross-origin strips query) — chain DISPROVEN; demo host 307 -> /partner-demo/v1 (clean demo page, no keys)
- DEVIATION: none exploitable
- STANDALONE SEVERITY: none; STATUS: parked (supersedes earlier onramp unknowns)

### L-29 rpc.testnet.arc.io + .io RPC variants verified
- COMPONENT: rpc.testnet.arc.io, rpc.drpc.testnet.arc.io, rpc.quicknode.testnet.arc.io (from docs llms-full.txt; the .network set was previously tested)
- OBSERVED: all live; eth_chainId 0x4cef52; same namespace lockdown; web3_clientVersion "arc/v1" (Arc's own node — extends L-27); TCP 10340 closed on all
- STANDALONE SEVERITY: none; STATUS: parked (extends L-15)

### L-30 docs.arc.io llms-full.txt (1.7MB doc dump) as discovery source
- COMPONENT: docs.arc.io (Mintlify; /llms.txt, /llms-full.txt, /sitemap.xml)
- OBSERVED: full host inventory surfaced (rpc.testnet.arc.io x118, onramp family, snapshots.arc.network [OOS], explorer paths, contract addresses incl. ERC-8004 registry 0x8004*; developer docs for entity-secret registration, W3S API refs [circle.com = OOS])
- CAPABILITIES GRANTED: host/route discovery feed (led to L-28/L-29); registry addresses 0x8004... = public testnet contract locations (public data)
- STANDALONE SEVERITY: none; STATUS: parked (discovery source, now consumed)

## CHAIN UPDATES (cycle 4d)
- L-28 sessionToken-in-URL + third-party frames: DISPROVEN (strict-origin-when-cross-origin on all onramp hosts)
- L-11 preview-token Referer leak: still OPEN (browser-gated); new data point: studio/portal use referrer-policy same-origin (stricter); preview host app headers unreachable via curl (edge 403) — test needs in-browser render
- L-16 signer trigger: re-checked via docs host inventory + cert SANs (wildcard only) — still dormant

## CYCLE 4e CLOSE-OUT (2026-09-21) — browser-gated items resolved
- L-20 (whoami/usage): CLOSED. GET /api/whoami -> 200 {contextFileLimits:{maxFiles:100,maxFileBytes:1048576,maxTotalBytes:10485760,maxBodyBytes:16777216}}; GET /api/usage -> 200 {percentUsed:0,windowMs:86400000}. Owner-scoped config/quota, no PII, no cross-user. Expected behavior.
- L-10 PAT gap sweep: CLOSED. Fresh PAT tested against ALL remaining endpoints: messages 200 (read), feedback 422 (shape validation, auth passes), analytics 200 (echo), template-load 400 invalid templateId (validates), sandbox-pause 200 (write on own sandbox, in-consent; sandbox resumed via terminal, files verified intact), sandbox-dev-server-restart 400 shape, hfspaces/disconnect 200 no-op, github-repo 405 POST-only, cli-tokens list 200. Consent boundary holds on EVERY endpoint. Both sweep tokens REVOKED (verified empty list).
- L-11 preview Referer leak: PARKED, effectively disproven. studio.arc.io sets referrer-policy: same-origin (cross-origin requests get NO referrer); onramp hosts strict-origin-when-cross-origin; preview host is CF-edge-challenged (403 to curl with/without token; browser falls back to studio landing). Even in the worst case the impact is rung 4 (owner's own preview, 1h window). Re-open trigger: an app whose preview embeds third-party resources + a permissive referrer-policy observed on the preview host.
- CF challenge: auto-cleared after primary-firefox restart; A session re-established (clean-cookie recipe), jar arcStudioA re-saved (11 cookies).


## CYCLE 4f (2026-09-21) — matrix completion + title XSS check + harness upgrade

### L-31 Cross-account matrix COMPLETE (B vs A, valid shapes)
- PREVIOUSLY inconclusive rows now tested with exact JS-derived shapes:
  - POST /api/sandbox-terminal (B vs A sandbox): 403 {"message":"Sandbox not found."}
  - POST /api/sandbox-console-write {op:append|truncate, sandboxId, lines:[json-strings]}: 403 Sandbox not found
  - POST /api/sandbox-trace-write {op:append|clear|truncate, sandboxId, lines:[json-strings]}: 403 Sandbox not found
  - GET /api/sandbox?appId=A (B): 404 App not found
- Body shapes recovered from e691b41bcb.js (console/trace writers use {op,sandboxId,lines}; lines are JSON STRINGS not objects; ops append/truncate/clear)
- FULL MATRIX NOW 100% CLOSED: every object x op x identity row held (terminal, console, trace, files, threads, deletes, GitHub, entity-secret, sandbox info)
- STANDALONE SEVERITY: none; STATUS: parked (consistent enforcement everywhere)

### L-32 App title stored-XSS check: SAFE
- POST /api/apps {action:update, title:"<img src=x onerror=alert(1)><svg/onload=alert(2)>"} -> 200
- Browser render: title shown as literal text; innerHTML has &lt;img (escaped); rawImg=false
- React escapes title at render; no stored XSS; input validated, output encoded
- STANDALONE SEVERITY: none; STATUS: parked (safe)

### L-33 Harness upgrade: frame-targeted eval (bridge extension)
- Extension patched: toTab accepts frameId; new actions evalIsoFrame (frameId param) + frames (webNavigation.getAllFrames; manifest +webNavigation permission); bridge client: evalIso <tabId> <js> <frameId> + frames <tabId>
- Enables native-setter form fill INSIDE cross-origin iframes (auth.circle.com) — the React-controlled-input fix the xte path couldn't reach
- Verified: auth form one-step variant has input[name=username|password] + Turnstile auto-solve (666-char token) + Sign in submit; BOTH A and B logged in reliably this way (A: 200 apps, B: 200 empty)
- This resolves the session-harness flakiness permanently: login recipe = nav -> frames -> evalIso(frameId) native-setter fill -> click Sign in


### L-34 Chat contract mapped owner-side (last unexplored surface)
- POST /api/chat (Vercel AI SDK, streamProtocol data, sendExtraMessageFields) -> 200 SSE 43KB
- SSE event types: progress, app_context {appId,threadId}, sandbox_step {create_sandbox|get_url}, sandbox_info, sandbox_files_sync, app_preview {port:5173, baseUrl, authedUrl+expiry}, assistant_text, turn_complete {durationMs}
- Read-only prompt -> NO tool-call events, no modifying steps (sandbox lifecycle only); turn_complete 2.9s
- app_preview duplicates the /api/sandbox preview JWT (same 1h token, same sandbox); no new exposure
- Lease: POST /api/chat/lease {threadId, runToken?} (heartbeat); cancel: POST /api/chat/cancel {threadId, runToken}
- STANDALONE SEVERITY: none; STATUS: parked (contract mapped; no deviation)

## FINAL LEDGER STATE (cycle 4f close)
- 34 entries, all chains closed or parked-with-reason; matrix 100% complete; chat/tool layer mapped; harness upgraded (frame-eval login, skill updated)
- Remaining: HF/Netlify connected flows (user accounts), deployments+entity-secret full lifecycle (quota), L-11 preview render (parked: referrer-policy evidence + CF edge), L-16 signer (dormant trigger)


## CYCLE 4g (2026-09-21) — L-06 chain CLOSED with full lifecycle evidence

### L-06 STATUS: CLOSED (evidence-complete, no finding)
Registration + retrieval lifecycle fully exercised with controlled accounts:
1. Netlify: OAuth connect (user) -> agent deploy -> live site https://legendary-tanuki-406e20.netlify.app (200, serves app HTML). Web deploys are NOT in /api/deployments (contract-only).
2. Contract deploy (Mode 1 platform_scp, user-chosen mode): MinimalToken at 0xb2243b4c0e0f919d38d3b93d8b947ba68dd1bb67 (Arc Testnet 5042002), explorer URL + txHash recorded; eth_getCode on rpc.testnet.arc.io returns real bytecode. /api/deployments populates with {contract,address,network,chainId,explorerUrl,txHash,deployedAt,source:platform_scp,scpContractId,deployerAddress}.
3. Entity-secret registration: required user's Circle console API key (agent cannot mint one; console.circle.com key created by user via browser, captured to 0600 file, appended to sandbox .env via /api/sandbox-terminal printf; .env read masks some secrets but returns real values for CIRCLE_* lines). Agent registered: CIRCLE_ENTITY_SECRET in .env + .circle/recovery_file.dat + live API verification.
4. RETRIEVAL (the chain question):
   - Session GET /api/entity-secret?appId&envVar=CIRCLE_ENTITY_SECRET -> 200 {secret:64ch, recoveryFile:null, recoveryFileName:null}
   - Same + recoveryFilePath=/home/user/app/.circle/recovery_file.dat -> 200 {secret:64ch, recoveryFile:192ch, recoveryFileName:17ch}
   - FRESH CLI PAT (origin_pat_*) -> 200 SAME 64-char secret (recovery null) -> PAT revoked immediately after
   - probe=1 -> 200 {exists:true} after registration (was 404 before)
5. VERDICT: the CLI PAT retrieves the app's wallet entity secret. Owner-scoped only (foreign appId 404, foreign sandbox 403, all cross-account rows held). Matches the product lifecycle (agent registers the secret during normal app builds; web UI backup modal uses the same endpoint; CLI manages the full app incl. wallets). Impact ceiling = Ladder rung 7 (own resources, consented capability). NOT reportable — same class as the dropped portal/signer leads (owner-scoped, by design, no cross-user impact). The consent-text granularity ("create and modify apps" vs raw wallet key) is a UX observation, not a security boundary crossing.
6. Cleanup: PAT revoked; API key pending user revocation in console.circle.com; Netlify site pending user deletion; local secret files kept 0600 pending revocation, then delete.

### L-35/36/37 (folded into L-06 evidence above): Netlify web deploy, contract deploy lifecycle, entity-secret lifecycle — all owner-scoped and behaving per product design.


## CYCLE 4h (2026-09-21) — symmetric matrix + remaining gaps closed

### L-38 A->B matrix COMPLETE (B now owns an app; both directions tested)
- B created app 300ebc4e-6fc6-4175-b2b7-7bca096500fb, thread 322d1112-5071-43c9-b577-000861b3cfa7, sandbox iizvq9wgeehrjipjbwrti (via chat, one quota run)
- A (foreign) against B's objects: sandbox info 404 App not found; file-list (appId+sandboxId AND sandboxId-only) 403 Sandbox not found; batch-read 403; file-write 403; terminal 403; messages 404 Thread not found; entity-secret 404; github-clone 404; deployments 404
- threads list as A for B's appId: 200 {"data":[]} (appId param filtered to caller's own threads; no cross-tenant title leak)
- thread delete as A for B's thread: 200 {"data":{"success":true}} BUT B's thread VERIFIED INTACT afterward (B session restored: thread + messages + sandbox all present) -> foreign delete = idempotent no-op, success response is cosmetic, NO unauthorized effect (matches B->A result). Not a finding (zero impact).
- MATRIX NOW FULLY SYMMETRIC: every object x op x direction held. Enforcement is object-anchored and identity-independent.

### L-11 CLOSED (effectively disproven, evidence-complete)
- Direct authed preview URL renders a GATE page: "This preview can only be viewed inside Arc Studio. Run arc-studio open --session..." — the preview token alone does NOT render the app outside the Studio session context
- App under test has no third-party resources (blank static page per agent)
- studio referrer-policy: same-origin (no cross-origin referrer at all)
- Combined: token-in-URL leak has no observable impact; closed.

### L-39 Mainnet RPC hosts (passive, policy-compliant)
- rpc.mainnet.arc.io / rpc.drpc.mainnet.arc.io / rpc.quicknode.mainnet.arc.io / rpc.blockdaemon.mainnet.arc.io all live, eth_chainId 0x13b2 = 5042 (Arc mainnet). Passive public-data check only.

## REMAINING (honest)
1. Onramp consumer-session cross-tenant (consumers/lookup with TWO sessions): needs a real phone number for the OTP flow (SMS to a genuine number; demo host embeds the production widget, no test-OTP shortcut found). Only untested PII-adjacent surface; gated on a burner phone.
2. HF Spaces connected deploy: needs the user's Hugging Face account (connect initiation already tested clean).


## CYCLE 4i (2026-09-21) — HF Spaces connected flow PROVEN (last user-gated item)
### L-40 HF Spaces connected deploy
- User connected HF account (kianoosh22) via /api/hfspaces/connect OAuth (scopes openid profile write-repos manage-repos; fixed redirect_uri studio.arc.io/api/hfspaces/callback; state-bound)
- Agent deploy: uploaded app files -> Space live at https://kianoosh22-app-c3a5a67c-7b5d-45ca-a5ee-dff978708230.static.hf.space (public static Space)
- Verified: 302 -> /index.html -> 200 text/html serving the app; page embeds window.huggingface.SPACE_CREATOR_USER_ID (HF's own public artifact)
- Connect/disconnect endpoints auth-gated; scope write-repos+manage-repos is the minimum for Space creation; no excess scope observed
- STANDALONE SEVERITY: none; STATUS: parked (third integration flow proven clean, matching GitHub + Netlify)

## FINAL GAP STATE (as of cycle 4i)
- CLOSED: A->B symmetric matrix, L-11 preview, mainnet passive probe, HF connected deploy, Netlify connected deploy, contract deploy lifecycle, entity-secret lifecycle (L-06)
- REMAINING (single item): onramp consumer-session cross-tenant test — gated on a controlled phone number for SMS OTP (no test-OTP shortcut on the demo host, which embeds the production widget). Design is self-scoping by construction (session minted per OTP-verified phone; lookup occurs after session). Blind-probing would fire real SMS at arbitrary numbers (disruptive, not done). PARKED with blocker documented.
- CLEANUP (user): revoke Circle API key (console.circle.com), delete Netlify site legendary-tanuki-406e20, delete HF Space kianoosh22-app-c3a5a67c-... Local secret files 0600, deleted after revocation confirmed.
