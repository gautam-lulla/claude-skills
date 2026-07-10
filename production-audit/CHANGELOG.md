# Changelog

All notable changes to the Production Audit (p-audit) skill will be documented in this file.

## [2.3.0] - 2026-07-10

### Added
- **Live Verification pass (read-only)** in the Audit Methodology: security findings (IDOR, cross-tenant, authorization) and accessibility (WCAG 2.2 AA) are now verified against the running dev servers — log in as two test accounts and attempt cross-tenant reads, run axe-core/Lighthouse, keyboard-walk key flows. Read actions and clearly-marked test data ONLY; no writes, no mutations, no destructive/money-moving calls. Findings verified live earn `Confirmed`; unverifiable ones stay `High Confidence`/`Needs Verification`
- **Section 22 Codebase-Specific Invariants & Known-Bug Regression Checks** — a LIVING checklist that loads the target project's CLAUDE.md "Invariants" section and treats every past bug as a named regression check, plus a high-severity starter set (cross-tenant authz, money-step zero-retry, booking idempotency, secret non-exposure, analytics DB-split joins, AiProviderService wiring, resolver decorator placement, @Cron double-fire, single-instance limiters, deploy SITE_URL, ingest CORS, MCP boundaries, ws override). Auto-expands as CLAUDE.md grows; highest-signal section because each check maps to a real prior incident
- **Report Section 2B Scope & Coverage Disclosure**: states which flows were traced end-to-end, whether the live-verification pass ran, and which checks were static-only — so confidence ratings are backed by visible coverage rather than self-asserted
- **Severity rubric**: severity is now derived from exploitability × blast radius × data sensitivity, not eyeballed — reproducible across runs and auditors
- **Adversarial second pass**: every High/Critical finding must survive an explicit refutation attempt (look for the upstream guard, the base-repo tenant filter, the framework default) before it ships
- **Machine-readable JSON findings file** emitted alongside the Markdown report, enabling automatic audit-over-audit diffing (resolved/new/still-open) instead of manual comparison

### Changed
- **Resource Constraints rewritten.** The stale "never run parallel agents" rule (written for a 16GB machine) is replaced with the real constraint for 32GB: up to ~4 parallel read-only analysis agents (~6 when dev servers are down), but never more than ONE heavy process (test/build/live-verification browser pass) at a time. The cap is on concurrent heavy work, not on agent count
- **`Confirmed` confidence now has a hard bar**: reproduced live, exhaustively traced with no possible upstream guard, or self-evident in source (e.g. hardcoded secret). Unexecuted "looks wrong" findings cannot be Confirmed
- **Grep scope widened** beyond `src/` to `admin/` and `packages/` for all frontend/rendered-output checks (accessibility, visual consistency, client race conditions, XSS, token storage)
- Health Scorecard template expands to 22 rows; overall score is now /220

## [2.2.0] - 2026-07-10

### Added
- **Audit Methodology section** (applies to every category): trace critical user flows end-to-end instead of reviewing files in isolation; adversarial posture for security, systematic for accessibility, precise for UI; evidence-over-speculation standard; challenge happy-path assumptions; explicit read-only rule — no code modifications until the report is complete
- **Three new audit categories** (21 total):
  - Section 19 **Client-Side Race Conditions & State Integrity**: duplicate submissions and idempotency (double-click, refresh-resubmit, retry duplicates), stale state and lost updates (out-of-order responses, optimistic-update rollback, cache invalidation), cleanup and leaks (effects, timers, listeners, in-flight requests), multi-tab/multi-device/poor-network checklist
  - Section 20 **Visual & Interaction Consistency**: design-token discipline, interaction-state matrix (hover/focus/active/selected/disabled/loading/success/warning/destructive/error), layout robustness (shift, clipping, truncation, empty states), copy consistency (terminology, capitalization, action labels, date/number formats)
  - Section 21 **Responsive & Edge-Case Content**: viewport sweep (320px through ultra-wide, 200% zoom, large text), content stress tests (long values, translated/expanded text, empty values, very large datasets, zero results, malformed content)
- **Security depth** in Section 2: new 2.5 Authorization, Tenancy & Object-Level Access (server-side permission checks, IDOR, cross-tenant scoping, privilege escalation) and 2.6 Injection, Redirects & Client-Side Storage (SQL/command/template/HTML/prompt injection, insecure redirects, token storage, secrets in URLs, generalized webhook signature verification, storage buckets, third-party integrations)
- **Reliability depth** in Section 6 (renamed Error Handling & Reliability): async failure modes (unhandled rejections, swallowed errors, broken retry loops, incomplete rollback), mandatory UI failure/progress-state matrix (loading/empty/error/offline/timeout/partial success), assumption audit at trust boundaries
- **Accessibility depth** in Section 7, framed against WCAG 2.2 AA: 7.6 Focus Management & Dynamic Content (focus trapping/restoration, aria-live for toasts/validation/loading, custom-widget ARIA patterns) and 7.7 Forms, Zoom & Motion (autocomplete attributes, error recovery, 200% zoom, large text, prefers-reduced-motion)
- **Evidence & confidence on every finding**: findings now include Evidence (required — no evidence, no finding) and Repro steps, plus a Confidence rating (Confirmed / High Confidence / Needs Verification); prioritized by real-world impact and exploitability
- **Release Recommendation** in the Executive Summary: Safe to ship / Ship with known risks / Do not ship, with a justification naming the deciding findings
- **Action Plan additions**: Quick Wins list (safe, low regression risk) and Needs Architecture / Deeper Investigation list

### Changed
- Health Scorecard template expands to 21 rows; overall score is now /210

## [2.1.0] - 2026-07-09

### Added
- **Four go-live audit categories** (18 total), added while preparing the first production launch:
  - Section 15 **Payment Security & PCI Compliance**: cardholder-data flow mapping and scope determination (SAQ level cheat sheet), no-PAN-in-logs checks, money-operation integrity (zero-retry charges, idempotency keys, server-derived amounts, webhook signatures), PCI operational go-live gates
  - Section 16 **Concurrency & Multi-Instance Safety**: in-memory state inventory with safe/degraded/broken classification, database race-condition patterns (find+save upserts, check-then-insert, unbacked uniqueness), horizontal-scaling readiness (@Cron double-fire, pool sizing, shared queues)
  - Section 17 **Load & Capacity Readiness**: load-test harness coverage of money paths (not just health checks), backpressure and limits (per-class rate limiting, body size, timeout budgets, expensive-endpoint gating), hot-path efficiency (N+1, indexes, batching)
  - Section 18 **Penetration-Test Readiness**: API attack surface (GraphQL introspection/depth/complexity, public-route inventory as pen-test scope, IDOR), auth hardening (login throttling, lockout, enumeration, cookie flags), input/outbound hardening (SSRF, uploads, raw SQL, CORS), external pen-test scheduling gate

### Changed
- Health Scorecard template expands to 18 rows; overall score is now /180

## [2.0.0] - 2026-04-24

### Changed
- **Major report format overhaul** -- 10-section canonical report replacing the minimal v1 template
- Reports now saved to Desktop as timestamped files for audit-over-audit tracking

### Added
- **Automated Verification Suite**: lint, unit tests, coverage report (test:cov), build run as part of every audit
- Executive Summary with plain-English verdict
- Health Scorecard (0-10 per category, letter grades, trend arrows)
- Risk Heat Map (severity count grid)
- Root Cause Patterns (cluster related findings into systemic issues)
- Comparison to Last Audit (delta tracking across runs)
- Action Plan (tiered: Fix Now / This Sprint / Schedule Later)
- Finding IDs (SEV-NNN) for cross-referencing
- Effort estimates on every finding (Quick / Moderate / Significant)
- Coverage breakdown table with threshold comparison
- Known/accepted exceptions excluded from open finding counts

### Unchanged
- All 14 audit categories remain the same
- E2E tests still opt-in (skipped by default)

## [1.3.0] - 2026-01-22

### Added
- Project selection prompt before audit
- Can now audit any project from anywhere

### Changed
- Updated version format to semver

## [1.2.0] - 2026-01-20

### Added
- Initial version tracking and changelog
- 12-category comprehensive audit
- Severity ratings (Critical, High, Medium, Low)
- Automated fix suggestions

### Features
- Software Architecture Audit
- Security Audit
- Performance Audit
- Code Quality Audit
- Developer Experience & Consistency Audit
- Error Handling Audit
- Accessibility Audit
- SEO Audit
- Environment & Configuration Audit
- Dependencies Audit
- Build & Deployment Audit
- Monitoring & Observability Audit

---

## Version History

| Version | Date | Summary |
|---------|------|---------|
| 2.3.0 | 2026-07-10 | Read-only live-verification pass (security + a11y), codebase-specific invariant regression checks (Section 22, living), severity rubric, adversarial second pass, coverage disclosure, JSON output; parallel-agent rule modernized for 32GB |
| 2.2.0 | 2026-07-10 | End-to-end flow-tracing methodology, evidence/confidence on findings, release recommendation, 3 new categories (client race conditions, visual consistency, responsive edge cases) |
| 2.1.0 | 2026-07-09 | Four go-live categories: PCI, concurrency/multi-instance, load & capacity, pen-test readiness |
| 2.0.0 | 2026-04-24 | Major report format overhaul, automated verification suite (lint/test/cov/build) |
| 1.3.0 | 2026-01-22 | Added project selection prompt |
| 1.2.0 | 2026-01-20 | Initial tracked version with 12-category audit |
