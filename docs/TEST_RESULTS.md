# Test results

Only results that were actually executed are listed. Anything not run is listed under "Not verified".

## Phase 2: application migration (executed 25 Oct 2026, Node 22.22.2, sandbox)

| Suite | Command | Result |
|---|---|---|
| Backend (API, auth, DynamoDB repo, server boot, UI end-to-end) | `cd backend && npm test` | **37 passed, 0 failed** |
| Frontend (sign-in logic, service worker, CryptoJS vs OpenSSL) | `node --test "frontend/test/**/*.test.js"` | **22 passed, 0 failed** |
| Dependency audit (production) | `cd backend && npm audit --omit=dev` | 0 vulnerabilities |
| Syntax | `node --check` on every shipped JS file | OK |

### What the tests prove

- **Token verification** uses the real `aws-jwt-verify` code with a locally generated RSA key. Rejected: expired token, wrong app client, wrong issuer, ID token used as access token, wrong signing key, `alg=none`, tampered payload, garbage.
- **Ownership**: user B cannot list, update or delete user A's notes; the owner is the token's `sub` even if the body or query claims another user.
- **DynamoDB code** runs through the real AWS SDK against `dynalite` (a DynamoDB emulator): CRUD, conditional update/delete, ordering, per-user isolation, pagination beyond one 1 MB query page, a 300 KB note round trip.
- **Server process** (`src/server.js`) boots, serves `/health` and `/config.js`, keeps `/metrics` off the public port, exits 0 on SIGTERM, and refuses to start with missing configuration.
- **Logs and metrics** never contain tokens, query strings or ciphertext.
- **Service worker**: never handles `/api/*`, `/config.js`, `/health`, cross-origin or non-GET requests; every precache file exists.
- **CryptoJS 4.2.0 (vendored)** is byte-identical to the npm package, and ciphertext is decryptable by OpenSSL and vice versa, so the existing note format is unchanged.
- **End-to-end UI journey (jsdom)**: sign-in callback, unlock, create, debounced autosave, reload and decrypt, wrong vault password, delete, two users isolated, expired token. The server-side record is ciphertext only and decrypts with the vault password.

### Defects found by testing (and fixed)

1. **Original service worker caches API responses.** Proven by running the original `service-worker.js` against `/api/notes`: it cached the URL even when the response was `no-store`. Fixed in `frontend/service-worker.js`.
2. **Stale unlock screen after session expiry.** The end-to-end test showed that after a 401 the app signed the user out but then returned to the unlock screen. Fixed with a guard in `frontend/app.js`.

### Sanity check on the tests themselves

Two protections were deliberately broken to confirm the tests notice: trusting a `userId` from the request body (test "owner comes from the token" failed) and allowing API responses to be cached (test "API responses are never cacheable" failed). Both were reverted.

## Not verified

- Real Cognito: token verification against live Cognito keys, Google sign-in, the managed login redirect. Only a local key and a stand-in token endpoint were used.
- Real DynamoDB (an emulator was used: no throttling, IAM or latency behaviour).
- A real browser: jsdom does not enforce the Content-Security-Policy, run the service worker, or show the layout. A static scan found nothing the CSP should block (no inline handlers/styles, no external URLs, system fonts only), but this is unconfirmed in a browser.
- Docker image, CI, AWS deployment, ALB, Auto Scaling, Prometheus/Grafana, load testing: later phases.
