# Arc HackerOne engagement resume

## Status
- Program: `arc-bbp` (Circle Arc L1), assessed 2026-09-20.
- No HackerOne reports submitted.
- No currently reportable finding.
- Two earlier leads are retained only as non-submission records:
  - `reports/01_portal_unauth_wallet_data_DO_NOT_SUBMIT.md`
  - `reports/02_remote_signer_UNPROVEN_DO_NOT_SUBMIT.md`

## Controlled identities and fixtures
- Portal A: `ciphershadow_9@wearehackerone.com`.
- Portal B: `ciphershadow_9+arc2@wearehackerone.com`.
- Studio A: metadata/credential file `.studio_account` (0600).
- Studio B: metadata/credential file `.studio_account2` (0600).
- Studio cookie jars `arcStudioA` and `arcStudioB` exist only in the running primary-Firefox extension memory and disappear on Firefox restart.
- Studio A controlled app: `c3a5a67c-7b5d-45ca-a5ee-dff978708230`.
- Studio A controlled sandbox: `irhdpwlq1o8mh9gk4wx0z`.
- Retained owner canary: `/home/user/app/public/bbp-authz-canary.txt`.

## Do not redo
- Portal anonymous wallet-data lead: demonstrated data was public chain data; no confidential linkage or harmful write.
- Remote signer: local unauthenticated arbitrary-message signing is proven, but production reachability and canonical accepted equivocation are not.
- Portal OTP replay, session gates, wallet list scoping, signing-material binding, policy binding and activity-detail cross-account checks.
- Studio anonymous API matrix: sandbox and system routes require authentication.
- Studio two-account authorization matrix:
  - foreign sandbox list/read/write/delete/terminal all denied with 403;
  - foreign app preview denied with 404;
  - foreign thread listing returns an empty caller-filtered list;
  - foreign thread update/delete return generic 200 but are verified owner-side no-ops.
- `malachite` deep review at `72143f6`: no strong exploitable candidate. See `review_malachite.md`.
- `arc-node` deep review at `2a3e8ab`: no strong exploitable candidate. See `review_arc_node.md`.
- Live-host sweep is in `httpx_arc_io_full.txt`; other enumerated names were DNS-dead or non-responsive. `staging.arc.io` is Cloudflare-blocked from both curl tooling and the primary browser.

## Remaining gated gaps
1. Studio A has a working GitHub OAuth connection and repository grant (`mamad-turbo/dispoeable-`); the full owner clone/push and foreign-denial matrix is complete. Nothing further is gated on the GitHub integration.
2. Portal agent spending-cap enforcement. Requires an in-scope testnet-specific funded surface; do not transact through the mainnet-facing portal.
3. EarnKit position confidentiality. Requires a controlled wallet with an actual vault position.
4. A remote-signer lead can be revived only with an untrusted production-equivalent network path, validator-key equality, and accepted canonical conflicting Arc votes.

## Harness
- Primary Firefox: `grjn1l1b.default-esr` on `DISPLAY=:10`, bridge TCP `127.0.0.1:9336`.
- Current browser identity: Studio A, with GitHub OAuth connected.
- The bridge cookie-clearing implementation was fixed to call `browser.cookies.remove` with a URL; sequential A/B switching now removes and restores the expected cookie counts.
- Studio request helper: `studio_api_request.py` (`STUDIO_TAB_ID=1` for the current tab).
- Restoring a cookie jar does not always restore the Studio Bearer session — re-login through the auth iframe may be required; verify identity with `/api/apps` before any matrix row.

## Additional tested-and-clean (cycles 3-4)
- agentvm.arc.io invite gate: uniform rejection for invalid codes, 429 lockout, `/configure.html` session-gated. No bypass.
- Studio foreign-thread surface: `/api/chat` 404 app_not_accessible, `/api/messages` 404, `/api/chat/cancel` runToken-bound, `/api/chat/lease` foreign success is a no-op with zero readable/cancellable effect (negative control proven).
- Studio sandbox control family: dev-server-restart 403 foreign; sandbox-pause foreign 200 is a no-op (owner state verified unchanged).
- Preview URLs (`*.preview.studio.arc.io`) return 403 without the owner-scoped `_preview_token`; token lives only in owner API responses.
- Studio public files (`/llms.txt`, `/SKILL.md`, `/.well-known/agent-skills/index.json`, `/api/health`) contain no sensitive data.

## Cleanup state
- Disposable Studio thread was owner-deleted and verified gone.
- Disposable delete-canary file was owner-deleted and verified gone.
- Foreign Studio write canary never existed according to owner readback.
- Retained intentionally: both controlled accounts, Studio app/sandbox, original authz canary, local evidence, and non-submission technical records.
- No reports were submitted and no production/mainnet transaction was sent.

## Cycle 4 additions (2026-09-21)
- NEW tested: entity-secret (owner-scoped), CLI PAT lifecycle (consent-exact), sandbox-terminal exec (owner+PAT), deployments (empty), preview JWT (1h {sandboxId,userId}), messages/feedback/analytics positive controls, Okta redirect flow (PKCE+state+fixed redirect), netlify/hf connect endpoints (auth-gated), agentvm /request-access (external form), feature-flags (route tables), arc-studio-cli npm package (secure storage + sound device-link), RPC debug/trace + 2 dead hosts.
- Session harness fixed: stale jar poisons login; recipe = clearCookies arc.io+circle.com (no restore) -> signup?EXISTING_ACCOUNT -> fill(960,460)+Tab+Return; jars arcStudioA/B valid.
- All new surfaces CLEAN (no reportable finding). CLI test token REVOKED. App title restored. Controlled artifacts intact.
- GATED (need user/quota): hfspaces+netlify connected flows (user accounts), deployments+entity-secret full lifecycle (AI quota), B foreign terminal direct (login flaky; proxy evidence 403), chat burn (quota).
- RESUME: A session = arcStudioA jar (valid); B = arcStudioB jar (valid but B login itself flaky).
