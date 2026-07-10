# Changelog

All notable changes to the Production Audit (p-audit) skill will be documented in this file.

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
| 2.2.0 | 2026-07-10 | End-to-end flow-tracing methodology, evidence/confidence on findings, release recommendation, 3 new categories (client race conditions, visual consistency, responsive edge cases) |
| 2.1.0 | 2026-07-09 | Four go-live categories: PCI, concurrency/multi-instance, load & capacity, pen-test readiness |
| 2.0.0 | 2026-04-24 | Major report format overhaul, automated verification suite (lint/test/cov/build) |
| 1.3.0 | 2026-01-22 | Added project selection prompt |
| 1.2.0 | 2026-01-20 | Initial tracked version with 12-category audit |
