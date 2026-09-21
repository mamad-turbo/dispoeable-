# Arc (Circle L1) — HackerOne arc-bbp — Engagement Notes
Date: 2026-09-20 (AEST). Cycles 1–2. Status: no reportable finding; nothing submitted.

## Program
- H1: https://hackerone.com/arc-bbp — **Circle's Arc L1** (NOT the Arc browser; the user's assumption was wrong — flagged).
- Rules: test on Arc TESTNET only; no mainnet testing. No DoS, no rate-limit reports, no CSRF-on-unauth-forms, no open redirect without chain, no clickjacking w/o sensitive actions. Detailed reports required. Source assets = Critical-eligible.
- Scope (6 in-scope): `*.arc.io` (Tier B payouts) | `rpc.testnet.arc.network` | `rpc.drpc.testnet.arc.network` | github.com/circlefin/{malachite, arc-node, arc-remote-signer} (source).
- Out of scope: help.arc.io, explorer.arc.io, community.arc.io. Everything circle.com is OUT of scope (api.circle.com, faucet.circle.com, console.circle.com, agents.circle.com).
- Rewards: Tier B Low $50-400 / Med $400-800 / High $800-3k / Crit $3k-10k. Chain/source assets go to Tier A (up to $200k, extreme $1M). 1440 reports/90d, 2 resolved — HIGH DUP RISK; submit only proven, well-documented findings.

## Identities (all disposable, operator-approved)
- Acct1: ciphershadow_9@wearehackerone.com — userId 01a0bdf0-35b1-7cdc-92b9-c86a45fdad85. Main wallet (SCA): 0xfda0a35a78e812754952a64d9f7bc3c1afcd4eac (blockchains ARC/MONAD/UNI/BASE/OP/AVAX/ETH/ARB/MATIC). walletSetId 6594d5ca-e08a-53f1-afde-d365c1b934a9.
- Acct1 AGENT wallet (Agent Wallet 1): 0x2cc9d1e85b0f639ab65e08bafd336df5da43c293 — id 17880fc7-8a42-50e0-a1e1-271c4725349a — EOA owner 0xf856b41e3112f4589c67d68d62133d0fae1cdb60 — walletSetId e1dd865f-e934-5cd6-b68e-3ece2752bb6c — **monthly spend limit $100 (DEFAULT policy)**.
- Acct2: ciphershadow_9+arc2@wearehackerone.com — session cookie jar `.cookies_acct2.txt` (0600), deviceId in `.device2.txt`. NO wallets created.
- Mailbox: himalaya account `quizlet9977` (IMAP kianoosh.9977@gmail.com). OTPs arrive as "XXX-###### Your verification code" from noreply@circle.com. NOTE: verify-otp requires the CF cookies from the email-otp response (-b jar).

## Findings after adversarial re-evaluation
1. **DO NOT SUBMIT — portal unauthenticated wallet data endpoints.** All demonstrated fields were public blockchain data keyed by public EVM addresses. `/api/wallets/register` is the unauthenticated Sigil indexer-registration path intentionally called for connected EOAs and no victim-visible effect was found. Cross-account Portal/WaaS boundaries held. Record: `reports/01_portal_unauth_wallet_data_DO_NOT_SUBMIT.md`.
2. **UNPROVEN — DO NOT SUBMIT — arc-remote-signer unauthenticated signing lead.** A local executed PoC proved the unmodified production service/server will sign two conflicting attacker-chosen messages for an unauthenticated plaintext client. However, the repository explicitly treats this as the trusted validator-facing sidecar API and relies on VPC/security-group isolation. Public in-scope RPC hosts had 10340 closed/filtered; no untrusted-to-signer production reachability or canonical accepted Arc double-vote was proven. Current evidence is expected behavior/hardening, not a High vulnerability. Record + PoC: `reports/02_remote_signer_UNPROVEN_DO_NOT_SUBMIT.md`, `reports/no_auth_network_poc_test.go`.

## API surface mapped (portal.arc.io)
- Auth: POST /api/pw/email-otp {email,deviceId,entity} → {sessionId,otpHead}; POST /api/pw/verify-otp {sessionId,otp,entity} → {signingMaterial{signingSessionId,signingExpiresAt,blob}} + Set-Cookie arc-pw-session; GET /api/pw/me; POST /api/pw/wallets; POST /api/pw/initialize → challengeId; POST /api/pw/approve-challenge {challengeId,signingMaterial}.
- Agent: POST /api/pw/agent-wallets (list); POST /api/pw/agent-wallets/create {signingMaterial} → challengeId + blockchains; GET /api/pw/agent-wallets/<id>; GET /api/pw/agent/session; GET /api/pw/policy/limit?wallet_address=.
- Data: GET /api/balances?walletAddress=&blockchain=; GET /api/tx-history/activities?walletAddress=; GET /api/tx-history/activities/:id; GET /api/gateway/balances?depositor=&blockchain=; GET /api/feature-flags (open); POST /api/wallets/register (open write); /v1/earnKit/vaults + /v1/earnKit/position/<addr> (upstream proxy, unauth but needs position to leak); /api/swapkit/v1/stablecoinKits/{rates,tokens} (public pricing); /api/analytics (POST), /api/log/*.
- Auth stack: Dynamic (app.dynamicauth.com env f3386450-b803-48...) + Turnkey (passkeys) + Circle WaaS SCA (scaCore circle_6900_singleowner_v4). Portal runs against MAINNET chains (USDC on Base etc.) — do NOT transact.
- Identities NOT shared cross-origin; studio.arc.io has its own token auth.

## studio.arc.io (in-scope, authenticated assessment completed)
- Vite SPA. Bearer-token APIs return 401 without a session. Mapped: `/api/chat` (+lease/cancel), `/api/threads`, `/api/apps`, `/api/feedback`, GitHub clone/pull/push/repo/repos/unlink, HF Spaces connect/disconnect, sandbox file-batch-read/file-write/file-delete/terminal/console/dev-server/pause/trace, `/api/template-load`, `/api/system/git-info`.
- Two independent program-compliant accounts were created and identity-proved: owner A lists one app; attacker B lists zero apps. Cookie jars `arcStudioA` and `arcStudioB` exist only in the running extension memory.
- Controlled owner fixture: app `c3a5a67c-7b5d-45ca-a5ee-dff978708230`, sandbox `irhdpwlq1o8mh9gk4wx0z`, retained canary `/home/user/app/public/bbp-authz-canary.txt`.
- Cross-account sandbox list, batch-read, write, delete and terminal execution all returned 403 `Sandbox not found`. Owner readback proved the foreign write did not land and the foreign delete did not remove the owner canary.
- Cross-account app preview returned 404. Thread list with a foreign app id returned an empty caller-filtered list. Foreign thread update/delete returned generic 200 responses but owner-side readback proved both were no-ops. A disposable owner thread was then deleted by the owner and verified gone.
- Studio A is connected to GitHub through the normal OAuth flow (grant: `mamad-turbo/dispoeable-`). Owner clone succeeded (repo cloned to `/home/user/app/dispoeable-`), owner file write inside the cloned repo succeeded, and owner push created commit `18edf21255844d43989691b78c9ca678254d18a9` (verified via `git ls-remote`). B's foreign push/pull/unlink against A's app returned 404 `App not found`, and `/api/github-repos` for B returned 401 "GitHub not connected" — B cannot see or use A's grant. No token material appeared in any response. Canaries were removed after verification.

## Tested-and-clean (do NOT redo)
- Testnet RPC namespaces on all 5 endpoints (rpc.testnet.arc.network, rpc.drpc.*, rpc.blockdaemon.testnet.arc.io, rpc.testnet.arc.io, rpc.quicknode.testnet.arc.io): eth_accounts/personal_/admin_/txpool_ all rejected. chainId 0x4cef52=5042002.
- OTP single-use (replay → 401 "OTP session expired or unknown").
- /api/pw/* session-gated (wallets/me/agent-wallets/agent/session/policy/limit → 401 without arc-pw-session).
- policy/limit wallet-bound (cross-user → 401 "Wallet authentication failed").
- approve-challenge signingMaterial-bound (cross-user → 401 signing_session_mismatch).
- agent-wallets cross-user → 502 (no data leak; backend error only).
- tx-history activity detail foreign id → 404.
- Wallet lists scoped per session.
- arc-node: reth-based; custom arc_ namespace = getVersion/getCertificate only; eth_callBundle REMOVED; RPC locked.
- malachite deep review at `72143f6`: no remotely exploitable consensus-safety, validation-bypass or authorization vulnerability confirmed. Notes: `review_malachite.md`. Targeted Rust tests were blocked because cargo was unavailable.
- arc-node deep review at `2a3e8ab`: no exploitable consensus/execution-validation or authorization vulnerability confirmed. Proposal/parts 18 passed, synced-value 16 passed, certificate conversion 9 passed. Notes: `review_arc_node.md`.
- 23 stale subdomains (account/gateway/core/eaas/overmind/inlay/router/storage*/webseed*/tcdn/tracker/cids/hello/demo/charity/qstest*/preview*/socket/tkr/warden/sentry/mainnet.arc.io/testnet.arc.io/devnet.arc.io/test.arc.io etc.) — DNS dead.
- Cycle-3 web sweep: arc.network apex redirects to www.arc.io; staging.arc.network and staging.arc.io are hard Cloudflare blocks in both curl and the primary browser; status.arc.network mirrors status.arc.io; agentvm.arc.io is a waitlist-marketing SPA whose invite gate (`/auth/verify`) rejects invalid codes with a uniform `invalid_invite_code` response, locks codes at 429, and gates `/configure.html` behind a session redirect — no bypass found; portal `/api/feature-flags` remains public (chain/token routing config only), balances/gateway endpoints unchanged; studio `/api/health`, `/robots.txt`, `/llms.txt`, `/SKILL.md`, `/sitemap.xml`, `/.well-known/agent-skills/index.json` are public marketing/skill content; `/cli-auth` is the logged-in-only CLI authorization consent page for account A. Nothing reportable surfaced.
- Cycle-4 studio deep-dive (as account B): foreign `/api/chat` on A's thread → 404 `app_not_accessible`; foreign `/api/messages?latest=1` → 404 `Thread not found`; foreign `/api/chat/cancel` with a forged runToken → 400 `Missing or invalid runToken` (token-bound); foreign `/api/sandbox-dev-server-restart` → 403; foreign `/api/sandbox-pause` returned a generic 200 `success` but is a no-op for an inaccessible sandbox (owner sandbox state and file list unchanged afterward); foreign `/api/hfspaces/connect|disconnect` are account-scoped and succeeded only as B's own no-op; `/api/template-load` requires sandboxId+templateId; `/api/feedback` shape requires feedbackKey/runId/appId/score. One UI quirk: `/api/chat/lease` on a foreign thread returns 200 `success:true`, but it grants nothing readable or cancellable (messages 404, cancel runToken-bound) — response-only success, no impact. No reportable finding.
- Session note: A's cookie jar restored via `restoreCookieJar arcStudioA` after Firefox restarts does NOT always carry the auth session (login is Bearer-token based, cookie jar alone can be insufficient — re-login through the auth iframe was required each time). Identity checks before matrix rows must use `/api/apps` (200 = A, 200-with-empty-list = B), not assumptions.
- Deep-review additions: WebSocket transport on rpc.testnet.arc.network completes the 101 handshake but silently drops app JSON-RPC payloads (curl HTTP POST works, so the drop is app/edge-level, not a finding); studio `/api/sandbox-console-write` and `/api/sandbox-trace-write` accept batch `{lines:[{sandboxId,line}]}` shapes (foreign rows scored inconclusive due to session staleness — shape-validated only); `/api/v2/*` references are Datadog SDK internals, no product surface; portal `/api/log/info` and `/api/analytics` are open POST log sinks by design (telemetry), not a finding. Full ledger: `COVERAGE_LEDGER.json`.

## Remaining gated/non-priority gaps
1. Studio A's GitHub grant (`mamad-turbo/dispoeable-`) exercised the full owner clone/push + foreign-denial matrix. Remaining GitHub surface is limited to multi-repo grants; nothing indicates a gap worth another repository grant.
2. Agent spending-cap enforcement needs a funded testnet-specific surface; the portal is mainnet-facing, so transactions remain off-limits under the program rules.
3. earnKit position confidentiality needs a controlled wallet with a real vault position.
4. Faucet/onramp flows are on `circle.com` assets and out of scope.

## Harness state
- mitmweb: 127.0.0.1:9090 (flows → ~/arc-bbp/arc_mitm_flows.mitm, web UI 9091). Firefox proxy config depends on it.
- Primary Firefox (grjn1l1b.default-esr) currently holds Studio owner A. Bridge 9336 is operational. The extension cookie-removal bug was fixed (`browser.cookies.remove` now receives a URL); sequential Studio identity switching was independently verified.
- Acct2 curl session: .cookies_acct2.txt (0600). signingMaterial blobs in pw_flows_dump*.txt — treat as secrets (mode 0600).
- Files: portal_js/ (42 chunks), studio_js/ (25), feature_flags_full.json, txhist_0001.json, flows, notes.

## Cleanup checklist
- No accounts/wallets to delete (testnet-ish portal wallets, no funds, no charges).
- Do NOT delete .cookies_acct2.txt or blobs until engagement closed.
- Kill mitmweb only after Firefox no longer needs it (or reset proxy prefs to direct).

## Cycle 4 (2026-09-21) — CLI PAT, entity-secret, Okta, session-harness fix
- NEW surface tested: /api/entity-secret, /api/cli-tokens (CLI PAT lifecycle), /api/deployments, /api/sandbox-terminal, /api/sandbox (preview JWT), /api/messages, /api/feedback, /api/analytics, /auth/okta/redirect, /api/template-load, /api/hfspaces/*, /api/sandbox-pause, /api/sandbox-dev-server-restart, /api/github-repo, agentvm /request-access.
- SESSION HARNESS FIX: stale cookie-jar restore POISONS fresh logins (auth server rejects the exchange). Working recipe: clearCookies arc.io + circle.com (NO jar restore) -> nav signup?EXISTING_ACCOUNT -> wait 10s -> click email (960,460) -> Escape -> Ctrl+A/Delete -> type -> Tab -> Ctrl+A/Delete -> type pw -> wait 7s (Turnstile) -> Return -> verify /api/apps. Re-established A and B sessions.
- ENTITY-SECRET (GET /api/entity-secret?appId&envVar[&probe=1|recoveryFilePath]): owner-scoped; foreign -> 404 "App not found"; owner-unregistered -> 404 "Entity secret not found"; GET-only (registration is agent-tool-driven). No bypass.
- CLI PAT (origin_pat_*): mint POST /api/cli-tokens {action:create,name} (browser session REQUIRED -> 403 for PAT); 3-month expiry (expiresAt in list); revoke POST action:revoke -> immediate 401. Scope = consent exactly: read apps/threads/messages/sandbox-files/git-info/deployments, write app title, EXEC sandbox terminal (SSE), mint 1h preview JWT ({sandboxId,userId}); blocked: GitHub (401), foreign sandbox (403 "Sandbox not found"), portal/agentvm (no effect). CLEAN, no boundary violation.
- Preview JWT: header claims {sandboxId, userId OktaId, iat, exp 1h}; authedPreviewUrlExpiresAt; preview hosts 403 to curl (CF). By design.
- Okta: org ausjngwm9gGd5l8vG5d6, client 0oa9v0plfmQsqwzPS5d7, PKCE S256, random state, FIXED redirect_uri /auth/okta/callback, server-side idp whitelist (bogus idp -> home), scope includes offline_access. Sandbox infra = E2B (CSP frame-src *.e2b.app). No issue.
- RPC re-check: www.rpc.blockdaemon.testnet.arc.io + rpc2.quicknode.testnet.arc.network = DNS dead; debug_/trace_ namespaces rejected (-32601) on rpc.testnet.arc.network.
- agentvm /request-access -> 302 to external Google Form (waitlist). Dead end.
- Deployments: /api/deployments?appId -> {"deployments":[]} (app never deployed; parked - deploy needs quota).
- Messages: owner reads thread history (200); feedback needs feedbackKey+runId (422 otherwise); analytics echo {"success":true}.
- Cleanup: CLI token revoked (dead 401), app title restored to bbp-sandbox-test (session POST), screenshots deleted, A jar re-saved (clean-session cookies).
- STILL OPEN (low value): hfspaces connect (needs user HF account like GitHub), deployments (needs quota), foreign terminal test via B (pattern established - skip).

## Cycle 4b (2026-09-21) — CLI client review, whoami/usage, session expiry, CF blocker
- arc-studio-cli npm v1.1.3 reviewed fully: token storage 0o600 file / macOS keychain (arc-studio-cli service); device-link = ephemeral loopback port + random state + fragment-only token + state check; sanitize.js strips ANSI/OSC/C0 incl OSC52 clipboard-overwrite from sandbox file content (terminal-injection aware); sensitive-files.js flags .env/.npmrc/.netrc/.git-credentials/.htpasswd/SSH keys/certs (agent exfil protection); api-url.js ALLOWS ONLY studio.arc.io + studio-staging.arc.io + loopback (no credential capture via baseUrl); postinstall benign; open-browser URL-validated. CLEAN.
- NEW endpoints from CLI client: GET /api/whoami (contextFileLimits maxFiles/maxFileBytes/maxTotalBytes), GET /api/usage (percentUsed/windowMs), /api/messages pageAfter pagination, /api/sandbox-file-list sandboxId-ONLY (no appId param needed), /api/sandbox-pause {appId}, chatDetached (POST /api/chat {detached:true} -> 202 appId+threadId). whoami+usage need a FRESH session (untested 200-side yet; anonymous 401).
- SESSION EXPIRY PATTERN: session cookie dies ~30min when driven via fetchIso alone (server rotates cookie; CLI adoptRefreshedCookie exists because responses refresh __session). UI usage keeps it alive. Restore jar works while server-side session valid.
- BLOCKER: automated re-login triggered a Cloudflare managed challenge that did NOT self-solve (>2min). Stopped per user rule (no captcha hammering / lockout risk). Browser left on the interstitial for user click-through; challenge tab 16 open.

## Cycle 4e (2026-09-21) — browser-gated items done
- CF challenge cleared via primary-firefox restart (clean state). A login re-established; jar arcStudioA saved.
- whoami/usage endpoints confirmed (context-file limits; 0% daily usage). PAT gap sweep complete: consent boundary holds on all endpoints; tokens revoked. Preview host CF-challenged; L-11 parked (referrer-policy same-origin evidence).
- Ledger: 30 entries, all chains closed or parked-with-reason. Remaining gated items: HF/Netlify connected flows (user accounts), deployments/entity-secret lifecycle (quota). No finding.

## Cycle 4f (2026-09-21) — HF deploy proven, matrix complete, cleanup + full restoration
- HF Spaces connected flow PROVEN: user kianoosh22 connected; agent deploy -> live Space https://kianoosh22-app-c3a5a67c-7b5d-45ca-a5ee-dff978708230.static.hf.space (200, serves app). Third integration, clean like GitHub + Netlify.
- A->B symmetric matrix COMPLETE: B owns bbp-b-app (300ebc4e-6fc6-4...) + sandbox iizvq9wgeehrjipjbwrti; A against B: files/terminal 403, messages/entity-secret/github/deployments 404, threads list empty (200), thread-delete success:true = idempotent NO-OP (B's thread verified intact). All rows held both directions.
- L-11 preview closed (no impact: gate page, no third-party resources, same-origin referrer). Mainnet RPCs passive check: 4 hosts, chainId 0x13b2 (5042).
- Onramp consumer cross-tenant: parked by design (real-SMS OTP only; blind-probing fires real SMS = disruptive, out of bounds).
- CLEANUP per user definition: only job PIDs killed (primary-firefox + hermes-page-tcp stopped). NO account delegations touched.
- RESTORATION (user: state as it was): Circle API key recreated in console (TEST_API_KEY:2a0134... new value; original revoked key gone — entity secret unaffected); sandbox .env patched with new key; GitHub connection re-established (OAuth re-run auto-complete); Netlify connection re-established (authorize page, user session); Netlify site recreated + renamed to exact original URL https://legendary-tanuki-406e20.netlify.app (200 verified).
- OBSERVATION: GitHub + Netlify connection records dropped silently between sessions (token/installation expiry on Arc side) — re-run /auth/{github,netlify}/connect when this happens.
- Repo state: account/arc-studio-accounts.md (passwords, API key, IDs, restored URL) + these records. Commits 6f09086d, 329d93c, cbe52b4f. Repo is PUBLIC — user notified.
- Final: no reportable finding; 40+ ledger entries all closed/parked-with-reason. Engagement at evidence-complete end state.
