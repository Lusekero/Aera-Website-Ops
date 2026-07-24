# AERA Platform Release Note

Date: 2026-07-24
Release Type: Production hardening and operational stability release
Audience: Operations, engineering, service owners, leadership

## Release summary

This release strengthens the AERA public digital platform for production reliability, security, and maintainability. It resolves a critical public access disruption, improves secure server-to-server behavior for SSR, hardens abuse controls, and formalizes production operational workflows across backend, client, and proxy repositories.

## What was delivered

1. Resolved critical production 429 issue

- Fixed Redis rate limiter integration that caused fail-closed behavior for public routes.
- Eliminated runtime error: this.client.rlflxIncr is not a function.
- Restored reliable access to public content endpoints.

2. Hardened SSR-to-API trust model

- Added trusted internal request channel using x-internal-request header.
- Introduced and validated INTERNAL_REQUEST_SECRET on backend.
- Added NUXT_INTERNAL_REQUEST_SECRET on client SSR path.
- Enforced startup fail-fast if production shared secret is missing or weak.

3. Reduced false-positive blocking and operational noise

- Excluded health checks from global security limiter.
- Excluded public upload read traffic (GET/HEAD) from security limiter.
- Suppressed health request log noise by default.

4. Improved certificate issuance reliability

- Added explicit certbot entrypoint fallback for first-time certificate issuance.
- Documented stale placeholder lineage cleanup scenario.

5. Strengthened operations and repeatability

- Added internal request secret generation/synchronization command in backend scripts.
- Exposed command via root orchestration flow for production profile execution.
- Updated handoff and deployment runbooks with validated deployment/recovery sequences.

## Repository activity included in this release window

Backend (Aera-Website-API)

- 83ec296 Fix rlflxIncr Redis limiter integration
- c09c630 Fix production limiter fallback to use Redis backend
- 68e4393 Add internal request secret generator command
- 7f685bf Harden SSR internal requests with shared secret
- b2da40d Exclude health and upload reads from security limiter
- 6b983c6 Skip security limiter for upload GET and HEAD reads

Client (Aera-Website-Client)

- d410042 Add SSR internal request secret header

Proxy (Aera-Website-Proxy)

- e79cf27 Fix cert init to run certbot entrypoint explicitly
- 4433c77 Update production edge compose configuration

## Operational impact

- Platform stability improved for public users by eliminating backend limiter backend miswiring.
- Security posture improved by moving SSR trust from implicit heuristics to explicit secret-based validation.
- Incident response speed improved through clearer runbooks and known-good deployment order.
- Production observability improved by reducing non-actionable health log noise.

## Risk and compatibility notes

- Backend and client shared secret values must remain synchronized in production.
- Redis endpoint configuration must resolve to the Docker Redis service on production network.
- Existing domain/TLS behavior remains compatible, with added fallback command for certificate bootstrap.

## Recommended post-release checks

1. Verify backend logs show no rlflxIncr or RATE_LIMIT_BACKEND_UNAVAILABLE errors.
2. Verify public pages load through SSR without 429 failures.
3. Verify uploads and health endpoints are reachable and not rate-limited.
4. Verify certificate status for active domains and edge health.

## Documentation updated

- ops/SESSION-HANDOFF.md
- ops/DEPLOY-CHEATSHEET.md
- ops/README.md
- backend/docs/deployment/ENVIRONMENT_REFERENCE.md
- backend/docs/security/PUBLIC_API_CLIENT_AUTH.md
- client/docs/getting-started/environment.md
- client/docs/guides/security-and-auth.md
- proxy/EDGE-README.md
