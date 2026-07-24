# Commit Plan for Documentation Updates

Date: 2026-07-24

Purpose: Create clean, auditable documentation commits per repository.

## 1) Ops repository

Files:

- DEPLOY-CHEATSHEET.md
- README.md
- SESSION-HANDOFF.md
- RELEASE-NOTE-2026-07-24.md
- MANAGEMENT-REPORT-DIGITAL-PORTAL-STRATEGIC-VALUE-2026-07-24.md

Commands:

cd /media/lusekero/Projects93/Aera/Website/ops

git add DEPLOY-CHEATSHEET.md README.md SESSION-HANDOFF.md RELEASE-NOTE-2026-07-24.md MANAGEMENT-REPORT-DIGITAL-PORTAL-STRATEGIC-VALUE-2026-07-24.md

git commit -m "docs(ops): add production hardening release notes and management report" -m "Update deploy cheat sheet and session handoff with SSR trusted channel, certbot fallback, limiter recovery, and multi-repo production runbook. Add one-page release note and executive strategic report."

git push origin main

## 2) Backend repository

Files:

- docs/deployment/ENVIRONMENT_REFERENCE.md
- docs/security/PUBLIC_API_CLIENT_AUTH.md

Commands:

cd /media/lusekero/Projects93/Aera/Website/backend

git add docs/deployment/ENVIRONMENT_REFERENCE.md docs/security/PUBLIC_API_CLIENT_AUTH.md

git commit -m "docs(backend): document SSR internal secret and redis production requirements" -m "Add INTERNAL_REQUEST_SECRET guidance, Redis host/url deployment notes, and trusted SSR internal request flow in public API auth docs."

git push origin main

## 3) Client repository

Files:

- docs/getting-started/environment.md
- docs/guides/security-and-auth.md

Commands:

cd /media/lusekero/Projects93/Aera/Website/client

git add docs/getting-started/environment.md docs/guides/security-and-auth.md

git commit -m "docs(client): add SSR internal request secret configuration" -m "Document NUXT_INTERNAL_REQUEST_SECRET usage for server-side trusted requests and production sync requirement with backend secret."

git push origin main

## 4) Proxy repository

Files:

- EDGE-README.md

Commands:

cd /media/lusekero/Projects93/Aera/Website/proxy

git add EDGE-README.md

git commit -m "docs(proxy): add explicit certbot entrypoint fallback guidance" -m "Update first certificate issuance procedure and include stale placeholder lineage cleanup note."

git push origin main

## 5) Recommended push order

1. Ops
2. Backend
3. Client
4. Proxy

Reason: Ops publishes authoritative runbooks first, then technical repo docs align with implementation.

## 6) Final verification

Run from workspace root:

git -C ops status --short

git -C backend status --short

git -C client status --short

git -C proxy status --short

Expected: no output (clean trees) after successful commits.
