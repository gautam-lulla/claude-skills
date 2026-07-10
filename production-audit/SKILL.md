---
name: production-audit
description: Comprehensive production readiness audit for Next.js/NestJS applications. Covers security, performance, code quality, accessibility, SEO, error handling & reliability, client-side race conditions, visual consistency, responsive edge cases, codebase-specific invariant regression checks, and operational readiness. Includes a read-only live-verification pass against the running app.
version: 2.3.0
author: SphereOS
license: MIT
tags: [Audit, Security, Performance, Code Quality, Production, Next.js, NestJS]
dependencies: []
allowed-tools: Read, Glob, Grep, Bash, Task, WebFetch
github-repo: gautam-lulla/claude-skills
skill-path: production-audit
---

## FIRST: Check for Updates

**Before proceeding, check if a newer version is available:**

1. Read local version from `~/.claude/skills/production-audit/SKILL.md` frontmatter
2. Fetch remote version: `https://raw.githubusercontent.com/gautam-lulla/claude-skills/main/production-audit/SKILL.md`
3. Compare the `version:` field in both

**If remote version > local version**, use AskUserQuestion:
- Question: "A newer version of p-audit is available. Current: [local] → Latest: [remote]. What would you like to do?"
- Options:
  1. "Update and continue (Recommended)"
  2. "Continue with current version"
  3. "View changelog"

**If user selects "Update and continue":**
- Fetch SKILL.md, CHANGELOG.md, LEARNINGS.md from GitHub raw URL
- Save to `~/.claude/skills/production-audit/`
- Confirm: "Updated p-audit to version [X]. Proceeding..."

**If user selects "View changelog":**
- Fetch and display `https://raw.githubusercontent.com/gautam-lulla/claude-skills/main/production-audit/CHANGELOG.md`
- Then ask again

**If versions match**, proceed silently (no prompt).

---

# Production Readiness Audit

This skill performs a comprehensive audit of a Next.js/NestJS application to ensure it is ready for production deployment. The audit covers security, performance, code quality, **developer experience & code consistency** (recommended for early-stage projects), accessibility, SEO, error handling & reliability, client-side race conditions, visual & interaction consistency, responsive/edge-case behavior, and operational readiness.

---

## IMPORTANT: First Step - Select Project

**Before running any audit checks, you MUST ask the user which project to audit using AskUserQuestion:**

```
Use AskUserQuestion with:
- Question: "Which project should I run the production audit on?"
- Options:
  1. Current directory ([show actual pwd path])
  2. Enter a different path
```

If the user selects option 2 ("Enter a different path"), they will type the path. Use that path for all subsequent audit commands.

Store the selected project path and use it as the base path for ALL file searches, reads, and commands throughout the audit.

---

## IMPORTANT: Resource Constraints

The real constraint is **concurrent heavy processes**, not agent count. On a 32GB machine, a
read-only analysis agent (grep/read/trace) costs only ~0.3-0.5GB; a `npm test` or `npm run build`
spikes 1-2GB. So the cap belongs on heavy work, not on parallelism.

- **Parallel analysis agents are OK — up to ~4.** You MAY fan out read-only analysis agents (e.g.
  one per audit category or category-group) to speed up the static pass, then synthesize their
  findings into one report. Cap at ~4 when the dev servers are up (mirrors the project's Jest
  maxWorkers), up to ~6 when they're not. Each such agent must be read-only: Read/Grep/Bash-for-
  inspection only, no tests, no builds, no writes.
- **Never more than ONE heavy process at a time.** Tests, builds, and the live-verification browser
  pass are serialized — one at a time, regardless of how many analysis agents are running. This is
  the guard that actually matters. Don't have two agents each run `npm test`/`npm run build`, and
  don't duplicate dev servers already running.

**Live-verification pass (read-only):** This machine comfortably runs the three dev servers
(backend :3501, admin :3502, site/booking-engine :3580), and they're usually already up. The audit
MAY connect to the running app and drive a browser (Playwright) to VERIFY security and accessibility
findings — see "Live Verification" in the Audit Methodology. Run it as a single sequential pass
(it's one of the "one heavy process at a time" slots). If the servers are down, note it and fall
back to static analysis for those checks rather than cold-starting a heavy stack mid-audit.

---

## How to Use This Skill

Invoke with `/production-audit` and Claude will:

1. **Ask which project to audit** (current directory or custom path)
2. **Run automated verification suite** (runs sequentially, not in parallel):
   - `npm run lint` -- code quality and style violations
   - `npm test` -- unit test suite (uses maxWorkers: 1 per project config)
   - `npm run test:cov` -- coverage report with threshold validation
   - `npm run build` -- confirms the project compiles cleanly
   - Note: E2E tests are **skipped by default** (require running services). Include only if the user explicitly requests them.
   - If any command fails, capture the failure output and continue with the remaining checks. Do NOT abort the audit.
3. Read the project structure and configuration files
4. Trace the application's critical user flows end-to-end (see Audit Methodology), then perform analysis across all 22 audit categories
5. Generate a detailed report using the Audit Report Template below
6. Provide severity ratings (Critical, High, Medium, Low), a Confidence rating per finding, and per-category health scores
7. Suggest fixes for each issue found, grouped into an action plan with a release recommendation

---

## Audit Methodology (applies to every category)

1. **Trace flows end-to-end.** Do not review files in isolation. Identify the important user flows (auth/login, booking/payment, content publish, public API reads, form submission) and follow each across the stack: UI → client state → API → auth/authorization → service → database → external providers → back to the UI. Most high-severity findings live at the seams between layers, not inside single files. Static grep checks find the cheap issues; flow tracing finds the ones that matter.

2. **Adopt the right posture per category.** Be **adversarial** in the security review — think like an attacker probing authorization, injection points, and trust boundaries, not like a linter. Be **systematic** in the accessibility review — walk WCAG 2.2 AA expectations against real flows, don't spot-check. Be **precise** in the UI review — name the exact token, state, or measurement that diverges; "looks inconsistent" is not a finding.

3. **Evidence over speculation.** Every finding must cite the code path, behavior, or reproducible condition that supports it, and carry a Confidence rating (Confirmed / High Confidence / Needs Verification). Distinguish confirmed vulnerabilities from theoretical concerns. If something smells wrong but you cannot support it with evidence, verify it or leave it out — do not pad the report with speculation.

4. **Challenge assumptions.** Actively hunt for problems that only emerge under real conditions: unreliable networks, concurrent actions, repeated clicks, multiple tabs, malicious input, empty or enormous datasets, and non-ideal content. "Works on the happy path at the developer's viewport" is not evidence of production readiness. Do not limit the audit to obvious linting or styling issues.

5. **The audit is read-only.** Do NOT modify code, configuration, or data during the audit. Produce the complete findings report first. Fixes happen afterward, as separately agreed work — never mid-audit. (Read-only means no code/config/data changes — the Live Verification pass below still only performs safe *read* actions.)

6. **Live Verification (security + accessibility).** Static analysis proposes; the running app confirms. After the static pass, run a single sequential verification pass against the already-running dev servers (do not cold-start a heavy stack — if they're down, note it and skip). This pass is **read-only probing**: safe read actions and clearly-marked test data ONLY — never write to real/prod-like data, never fire destructive or money-moving calls, never exercise mutations for their side effects.
   - **Security:** for each suspected authorization/IDOR/cross-tenant finding, verify it. Log in as two separate test accounts (two orgs), and attempt to read account B's object using account A's session — a success is a *Confirmed* Critical; a clean 403/404 kills the finding. Check that guarded mutations reject unauthenticated/wrong-role callers.
   - **Accessibility:** WCAG 2.2 AA is not reliably auditable from source. Run axe-core / Lighthouse against the key rendered pages, keyboard-walk the primary flows (login, booking, a form, a modal), and confirm what a screen reader announces on dynamic changes. Report tool output, not guesses.
   - Anything you could not verify live stays at *High Confidence* / *Needs Verification* and says why (servers down, no test data, etc.). Never upgrade to *Confirmed* without an executed check.

7. **Severity is derived, not guessed.** Rate each finding on three axes and combine — don't eyeball it:
   - **Exploitability:** trivial/remote-unauth (high) → needs auth + specific conditions (low)
   - **Blast radius:** whole tenant base / all deployed sites / money or messages to third parties (high) → one non-critical widget (low)
   - **Data sensitivity:** credentials, PII, cross-tenant data, payment data (high) → cosmetic/public data (low)
   Guide: **Critical** = high on exploitability AND (high blast radius OR high sensitivity) — e.g. unauth cross-tenant data read. **High** = serious but gated by auth/conditions, or high-impact but not directly exploitable. **Medium** = real but contained. **Low** = minor/cosmetic. **Informational** = no direct risk, worth noting. Applying the rubric the same way every run is what makes audit-over-audit trend numbers mean something.

8. **Adversarial second pass on High/Critical.** Before finalizing, re-examine every High and Critical finding with the opposite goal: try to *refute* it. Is there a guard upstream you missed? A tenant filter applied in a base repository? A framework default that already covers it? A finding that survives a genuine refutation attempt ships; one that doesn't gets downgraded or dropped. This is the single biggest defense against confident-but-wrong findings.

9. **Search the whole tree, not just `src/`.** This monorepo splits across `src/` (backend), `admin/` (Next.js admin), and `packages/` (shared renderer + `site-template` booking engine). The grep commands in each category default to `src/`; widen them to `admin/` and `packages/` wherever the check applies to frontend or rendered output (accessibility, visual consistency, client race conditions, XSS, token storage). A check that only greps `src/` silently gives the frontend a free pass.

---

## Audit Categories

1. [Software Architecture Audit](#1-software-architecture-audit)
2. [Security Audit](#2-security-audit)
3. [Performance Audit](#3-performance-audit)
4. [Code Quality Audit](#4-code-quality-audit)
5. [Developer Experience & Code Consistency Audit](#5-developer-experience--code-consistency-audit) ⭐ *Early-stage priority*
6. [Error Handling & Reliability Audit](#6-error-handling--reliability-audit)
7. [Accessibility Audit](#7-accessibility-audit)
8. [SEO Audit](#8-seo-audit)
9. [Environment & Configuration Audit](#9-environment--configuration-audit)
10. [Dependencies Audit](#10-dependencies-audit)
11. [Build & Deployment Audit](#11-build--deployment-audit)
12. [Monitoring & Observability Audit](#12-monitoring--observability-audit)
13. [Test Coverage Audit](#13-test-coverage-audit) ⭐ *Critical for CI/CD*
14. [Technical Debt Audit](#14-technical-debt-audit) ⭐ *Health tracking*
15. [Payment Security & PCI Compliance Audit](#15-payment-security--pci-compliance-audit) ⭐ *Go-live priority*
16. [Concurrency & Multi-Instance Safety Audit](#16-concurrency--multi-instance-safety-audit) ⭐ *Go-live priority*
17. [Load & Capacity Readiness Audit](#17-load--capacity-readiness-audit) ⭐ *Go-live priority*
18. [Penetration-Test Readiness Audit](#18-penetration-test-readiness-audit) ⭐ *Go-live priority*
19. [Client-Side Race Conditions & State Integrity Audit](#19-client-side-race-conditions--state-integrity-audit) ⭐ *Real-user conditions*
20. [Visual & Interaction Consistency Audit](#20-visual--interaction-consistency-audit)
21. [Responsive & Edge-Case Content Audit](#21-responsive--edge-case-content-audit)
22. [Codebase-Specific Invariants & Known-Bug Regression Checks](#22-codebase-specific-invariants--known-bug-regression-checks) ⭐ *Living — highest signal*

---

## 1. Software Architecture Audit

### 1.1 Project Structure & Organization

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Directory structure | High | Inconsistent or flat structure, no clear separation |
| Feature organization | Medium | Related files scattered across directories |
| Barrel exports | Medium | Missing or inconsistent index.ts files |
| Colocation | Medium | Related files not colocated (component + styles + tests) |

**Expected Next.js App Router Structure:**
```
src/
├── app/                      # Routes and pages
│   ├── (marketing)/          # Route groups for layouts
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── (dashboard)/
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── api/                  # API routes
│   ├── layout.tsx            # Root layout
│   ├── error.tsx             # Error boundary
│   └── not-found.tsx         # 404 page
├── components/
│   ├── ui/                   # Primitive/atomic components
│   ├── blocks/               # Composite sections
│   └── layout/               # Layout components
├── lib/                      # Utilities and shared logic
│   ├── utils/                # Pure utility functions
│   ├── hooks/                # Custom React hooks
│   ├── context/              # React context providers
│   └── api/                  # API client/fetching layer
├── types/                    # TypeScript type definitions
├── config/                   # Configuration constants
└── styles/                   # Global styles
```

**Audit Checklist:**
- [ ] Clear separation between app routes and reusable code
- [ ] Components organized by type (ui/blocks/layout) or feature
- [ ] Shared utilities in `lib/` not scattered in components
- [ ] Types centralized or colocated with relevant code
- [ ] No business logic in page components (delegated to lib/)

### 1.2 Component Architecture

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Single Responsibility | High | Components doing too many things |
| Prop drilling | High | Props passed through 3+ levels |
| Component size | Medium | Components > 200-300 lines |
| Presentation vs Container | Medium | Mixed data fetching and UI rendering |
| Reusability | Medium | Duplicate component logic |

**Component Hierarchy Pattern:**
```
Pages (Route Components)
  └── Fetch data, handle routing
  └── Pass data to templates/blocks

Templates (Page-level layouts)
  └── Compose blocks into page structure
  └── No data fetching

Blocks (Section components)
  └── Self-contained sections (Hero, Features, etc.)
  └── Receive content as props

UI Components (Primitives)
  └── Button, Input, Card, etc.
  └── Purely presentational
  └── Design system atoms
```

**Anti-patterns to Flag:**
```typescript
// ❌ BAD: Component doing too much
export function UserDashboard() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  const [notifications, setNotifications] = useState([]);
  // ... 300 lines of mixed concerns
}

// ✅ GOOD: Separated concerns
export function UserDashboard() {
  return (
    <DashboardLayout>
      <UserProfile />
      <UserPosts />
      <UserNotifications />
    </DashboardLayout>
  );
}
```

**Audit Commands:**
```bash
# Find large components (>200 lines)
find src/components -name "*.tsx" -exec wc -l {} \; | awk '$1 > 200 {print}'

# Check for prop drilling patterns
grep -r "props\." --include="*.tsx" src/components | wc -l
```

### 1.3 Data Flow & State Management

| Check | Severity | What to Look For |
|-------|----------|------------------|
| State location | High | Global state for local concerns |
| Server vs Client state | High | Duplicating server data in client state |
| Prop drilling depth | Medium | Data passed through many component layers |
| State updates | Medium | Unnecessary re-renders from state changes |
| Cache strategy | Medium | No caching or over-caching |

**State Categories:**
| Type | Where to Store | Example |
|------|----------------|---------|
| Server state | React Query / Apollo / Server Components | User data, CMS content |
| Global UI state | Context or Zustand | Theme, sidebar open |
| Local UI state | useState | Form inputs, modals |
| URL state | searchParams | Filters, pagination |
| Form state | React Hook Form / useFormState | Form values, validation |

**Data Flow Pattern (Next.js App Router):**
```
Server Component (page.tsx)
  └── Fetch data from CMS/API
  └── Pass as props to children

Client Components (interactive parts only)
  └── Local UI state (useState)
  └── Form state (useFormState)
  └── NO data fetching (receive as props)
```

**Anti-patterns to Flag:**
```typescript
// ❌ BAD: Fetching in client component
'use client';
export function ProductList() {
  const [products, setProducts] = useState([]);
  useEffect(() => {
    fetch('/api/products').then(res => res.json()).then(setProducts);
  }, []);
  // ...
}

// ✅ GOOD: Server component fetches, client receives
// page.tsx (Server Component)
export default async function ProductsPage() {
  const products = await getProducts();
  return <ProductList products={products} />;
}

// ProductList.tsx (Client Component for interactivity)
'use client';
export function ProductList({ products }: { products: Product[] }) {
  // Only UI state here
}
```

### 1.4 Separation of Concerns

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Business logic in components | High | Complex logic mixed with JSX |
| API calls in components | High | fetch/axios directly in components |
| Hardcoded values | Medium | Magic numbers/strings in components |
| Style logic in components | Low | Complex conditional styling inline |

**Layer Separation:**
```
┌─────────────────────────────────────────────┐
│  Presentation Layer (components/)           │
│  - React components                         │
│  - UI rendering only                        │
│  - Receives data as props                   │
└─────────────────────────────────────────────┘
                    ↑ props
┌─────────────────────────────────────────────┐
│  Application Layer (app/, lib/hooks/)       │
│  - Page components                          │
│  - Custom hooks                             │
│  - Orchestrates data flow                   │
└─────────────────────────────────────────────┘
                    ↑ data
┌─────────────────────────────────────────────┐
│  Data Layer (lib/api/, lib/content/)        │
│  - API clients                              │
│  - Data fetching functions                  │
│  - Data transformation                      │
└─────────────────────────────────────────────┘
                    ↑ raw data
┌─────────────────────────────────────────────┐
│  External Services (CMS, APIs, DB)          │
└─────────────────────────────────────────────┘
```

**Expected File Organization:**
```typescript
// lib/content/index.ts - Data fetching abstraction
export async function getPageContent(slug: string) { ... }
export async function getProducts() { ... }

// lib/utils/formatters.ts - Pure utility functions
export function formatPrice(amount: number) { ... }
export function formatDate(date: Date) { ... }

// lib/hooks/useCart.ts - Stateful logic
export function useCart() { ... }

// components/ui/Button.tsx - Presentational only
export function Button({ children, variant }) { ... }
```

### 1.5 API Design & Data Fetching

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Consistent fetching pattern | High | Mixed fetch methods across codebase |
| Error handling | High | Inconsistent or missing error handling |
| Loading states | Medium | No loading indicators |
| Caching strategy | Medium | No caching or improper cache invalidation |
| Type safety | Medium | Untyped API responses |

**Data Fetching Layer Pattern:**
```typescript
// lib/api/client.ts - Base API client
class APIClient {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`);
    if (!response.ok) {
      throw new APIError(response.status, await response.text());
    }
    return response.json();
  }

  async post<T>(endpoint: string, data: unknown): Promise<T> {
    // ...
  }
}

// lib/api/products.ts - Domain-specific functions
export async function getProducts(): Promise<Product[]> {
  return apiClient.get<Product[]>('/products');
}

export async function getProduct(id: string): Promise<Product> {
  return apiClient.get<Product>(`/products/${id}`);
}
```

**GraphQL Pattern (Apollo/SphereOS CMS):**
```typescript
// lib/queries/index.ts - Query definitions
export const GET_PAGE_CONTENT = gql`...`;

// lib/content/index.ts - Content fetching abstraction
export async function getPageContent(slug: string) {
  const client = getServerClient();
  const { data } = await client.query({
    query: GET_PAGE_CONTENT,
    variables: { slug },
  });
  return data?.page ?? null;
}
```

### 1.6 Type System Architecture

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Type coverage | High | `any` types, missing types |
| Type organization | Medium | Types scattered or duplicated |
| API type safety | High | Untyped API responses |
| Component prop types | Medium | Missing or incomplete prop interfaces |
| Discriminated unions | Low | Not using for complex state |

**Type Organization Pattern:**
```typescript
// types/index.ts - Re-export all types
export * from './api';
export * from './content';
export * from './components';

// types/api.ts - API response types
export interface APIResponse<T> {
  data: T;
  meta: {
    total: number;
    page: number;
  };
}

export interface APIError {
  code: string;
  message: string;
}

// types/content.ts - CMS content types
export interface Page {
  id: string;
  slug: string;
  title: string;
  content: PageContent;
}

export interface PageContent {
  hero: HeroSection;
  sections: Section[];
}

// types/components.ts - Shared component types
export interface BaseComponentProps {
  className?: string;
  children?: React.ReactNode;
}
```

**Type Inference Best Practices:**
```typescript
// ✅ GOOD: Infer from schema/API
type Product = Awaited<ReturnType<typeof getProduct>>;

// ✅ GOOD: Discriminated unions for state
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

// ✅ GOOD: Zod for runtime validation + type inference
const ProductSchema = z.object({
  id: z.string(),
  name: z.string(),
  price: z.number(),
});
type Product = z.infer<typeof ProductSchema>;
```

### 1.7 Dependency Injection & Testability

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Hard-coded dependencies | Medium | Direct imports of external services |
| Global singletons | Medium | Modules with side effects on import |
| Testability | Medium | Functions that can't be unit tested |
| Mocking difficulty | Low | Tight coupling making mocks complex |

**Dependency Injection Pattern:**
```typescript
// ❌ BAD: Hard-coded dependency
export async function getProducts() {
  const response = await fetch('https://api.example.com/products');
  return response.json();
}

// ✅ GOOD: Injectable dependency
export function createProductService(apiClient: APIClient) {
  return {
    async getProducts() {
      return apiClient.get<Product[]>('/products');
    },
    async getProduct(id: string) {
      return apiClient.get<Product>(`/products/${id}`);
    },
  };
}

// Usage
const productService = createProductService(apiClient);

// Testing
const mockClient = { get: jest.fn() };
const testService = createProductService(mockClient);
```

### 1.8 Scalability Patterns

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Code splitting | High | Large bundles, no dynamic imports |
| Lazy loading | Medium | All components loaded upfront |
| Route-based splitting | Medium | Not using Next.js automatic splitting |
| Feature flags | Low | No mechanism for gradual rollouts |

**Code Splitting Patterns:**
```typescript
// ✅ Dynamic import for heavy components
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // If client-only
});

// ✅ Route groups for different layouts
app/
├── (marketing)/     # Marketing pages layout
│   ├── layout.tsx
│   └── page.tsx
├── (app)/           # App dashboard layout
│   ├── layout.tsx
│   └── dashboard/
└── (auth)/          # Auth layout
    ├── layout.tsx
    └── login/

// ✅ Parallel routes for complex UIs
app/
├── @modal/          # Modal slot
├── @sidebar/        # Sidebar slot
└── layout.tsx       # Composes slots
```

### 1.9 Design Patterns & Conventions

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Consistent patterns | Medium | Different approaches for same problem |
| Named exports | Low | Default exports everywhere |
| File naming | Low | Inconsistent naming conventions |
| Import organization | Low | Messy, unordered imports |

**Recommended Patterns:**

**Compound Components (for complex UI):**
```typescript
// Usage
<Accordion>
  <Accordion.Item>
    <Accordion.Trigger>Title</Accordion.Trigger>
    <Accordion.Content>Content</Accordion.Content>
  </Accordion.Item>
</Accordion>
```

**Render Props / Children as Function:**
```typescript
<DataFetcher url="/api/users">
  {({ data, loading, error }) => (
    loading ? <Spinner /> : <UserList users={data} />
  )}
</DataFetcher>
```

**Custom Hooks for Logic Reuse:**
```typescript
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);
  return debouncedValue;
}
```

### 1.10 Architecture Checklist

**Project Structure:**
- [ ] Clear directory organization (app/, components/, lib/, types/)
- [ ] Consistent file naming convention
- [ ] Related files colocated
- [ ] Barrel exports (index.ts) for clean imports

**Component Architecture:**
- [ ] Single responsibility per component
- [ ] No prop drilling (use composition or context)
- [ ] Components < 200 lines
- [ ] Clear separation: UI components vs page components

**Data Flow:**
- [ ] Server Components for data fetching
- [ ] Client Components only for interactivity
- [ ] Centralized data fetching layer (lib/content/ or lib/api/)
- [ ] Proper state management (server state vs client state)

**Separation of Concerns:**
- [ ] No business logic in components
- [ ] No direct API calls in components
- [ ] Utilities extracted to lib/
- [ ] Types centralized in types/

**Type Safety:**
- [ ] No `any` types
- [ ] All API responses typed
- [ ] All component props typed
- [ ] Zod or similar for runtime validation

**Scalability:**
- [ ] Dynamic imports for heavy components
- [ ] Route-based code splitting
- [ ] Feature organization supports team scaling

---

## 2. Security Audit

### 1.1 Environment Variables

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Secrets in code | Critical | API keys, passwords, tokens hardcoded in source files |
| .env in git | Critical | `.env`, `.env.local`, `.env.production` not in `.gitignore` |
| NEXT_PUBLIC exposure | High | Sensitive values prefixed with `NEXT_PUBLIC_` (exposes to client) |
| Missing env validation | Medium | No runtime validation of required environment variables |

**Files to Check:**
```
.env*
.gitignore
src/**/*.ts
src/**/*.tsx
next.config.js
```

**Audit Commands:**
```bash
# Check for hardcoded secrets patterns
grep -r "sk_live\|sk_test\|api_key\|apikey\|secret\|password\|token" --include="*.ts" --include="*.tsx" --include="*.js" src/

# Check if .env files are gitignored
grep -E "^\.env" .gitignore

# List all NEXT_PUBLIC variables
grep -r "NEXT_PUBLIC_" --include="*.ts" --include="*.tsx" src/
```

**Required Fixes:**
- [ ] All secrets stored in environment variables only
- [ ] `.env*` files in `.gitignore`
- [ ] Only non-sensitive values use `NEXT_PUBLIC_` prefix
- [ ] Environment variables validated at startup

### 1.2 Authentication & Authorization

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Auth on protected routes | Critical | Routes accessible without authentication |
| Session handling | High | Insecure session storage, missing expiration |
| CSRF protection | High | Forms without CSRF tokens |
| Input validation | High | User input not validated/sanitized |

**Files to Check:**
```
src/app/**/page.tsx
src/middleware.ts
src/lib/auth/
```

**Checklist:**
- [ ] Protected routes have auth checks
- [ ] Middleware validates authentication tokens
- [ ] Sessions expire appropriately
- [ ] User input is validated before use

### 1.3 HTTP Security Headers

| Header | Severity | Recommended Value |
|--------|----------|-------------------|
| Strict-Transport-Security | High | `max-age=31536000; includeSubDomains` |
| X-Content-Type-Options | Medium | `nosniff` |
| X-Frame-Options | Medium | `DENY` or `SAMEORIGIN` |
| X-XSS-Protection | Low | `1; mode=block` |
| Content-Security-Policy | High | Project-specific CSP |
| Referrer-Policy | Medium | `strict-origin-when-cross-origin` |

**Files to Check:**
```
next.config.js
vercel.json
middleware.ts
```

**Example Configuration (next.config.js):**
```javascript
const securityHeaders = [
  { key: 'Strict-Transport-Security', value: 'max-age=31536000; includeSubDomains' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-XSS-Protection', value: '1; mode=block' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
];

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```

### 1.4 Data Exposure

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Over-fetching | High | GraphQL/API returning more data than needed |
| Console.log leaks | Medium | Sensitive data logged to console |
| Source maps in prod | Medium | Source maps exposing code structure |
| Error message exposure | Medium | Stack traces shown to users |

**Audit Commands:**
```bash
# Check for console.log statements
grep -r "console\." --include="*.ts" --include="*.tsx" src/

# Check source map configuration
grep -r "productionBrowserSourceMaps" next.config.js
```

### 2.5 Authorization, Tenancy & Object-Level Access

> **Adversarial focus:** the UI hiding an action is NOT an authorization check. For every
> endpoint/resolver that accepts an entity id, trace the lookup: does the query filter by
> the caller's organization/tenant/ownership, or fetch by id alone? An id-only fetch of
> tenant-scoped data is a confirmed IDOR, not a theoretical one.

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Missing server-side permission checks | Critical | Mutations/queries relying on the client to hide actions; guards on REST but not the equivalent GraphQL path (or vice versa) |
| Insecure direct object references (IDOR) | Critical | Entity ids accepted and fetched without ownership/tenant scoping |
| Cross-tenant data access | Critical | id-only lookups on org/property-scoped data; list queries missing tenant filters |
| Privilege escalation | Critical | Role/permission/isAdmin fields settable through user-controlled input (mass assignment into auth fields) |
| Trust-boundary validation | High | Inputs validated client-side only; internal services trusting caller-supplied org/user ids |

**Audit Commands:**
```bash
# Bare-id fetches (manual review list for tenant scoping)
grep -rn "findOne\|findOneBy" src/modules --include="*.service.ts" | grep -v spec | grep -iE "\{ *id" | head -30

# Mutations without visible guards (cross-check each against the global guard config)
grep -rn -B3 "@Mutation(" src/modules --include="*.resolver.ts" | grep -v "UseGuards\|@Public" | head -20
```

### 2.6 Injection, Redirects & Client-Side Storage

| Check | Severity | What to Look For |
|-------|----------|------------------|
| SQL injection | Critical | String-built SQL with user input (bind params/registries only) |
| Command injection | Critical | exec/spawn/shell calls with user-influenced arguments |
| HTML/script injection (XSS) | High | dangerouslySetInnerHTML / innerHTML / string-built markup from user or CMS content without sanitization |
| Template injection | High | User input evaluated inside template engines or dynamic render paths |
| Prompt injection | High | User-supplied or crawled content concatenated into LLM prompts without data/instruction separation markers |
| Insecure redirects | High | Redirect targets taken from query params/user input without an allowlist |
| Token storage | High | Access/refresh tokens in localStorage/sessionStorage (readable by any XSS) instead of httpOnly cookies |
| Secrets/PII in URLs | High | Tokens, keys, or personal data in GET query strings (logged by proxies, leaked via Referer) |
| Webhook handlers | High | ANY inbound webhook processed without signature/secret verification — not just payment webhooks |
| Storage buckets | High | Public-write or public-list object storage; signed-URL generation without scoping/expiry |
| Third-party integrations | Medium | Overly broad OAuth scopes; third-party scripts with access to sensitive pages |

**Audit Commands:**
```bash
# HTML injection sinks
grep -rn "dangerouslySetInnerHTML\|\.innerHTML" src admin packages --include="*.tsx" --include="*.ts" 2>/dev/null | grep -v spec | head -20

# Redirect targets influenced by user input
grep -rn "redirect(\|window\.location" src admin packages --include="*.ts" --include="*.tsx" 2>/dev/null | grep -iE "query|params|searchParams|req\." | head -20

# Tokens in web storage
grep -rn "localStorage.setItem\|sessionStorage.setItem" admin packages --include="*.ts" --include="*.tsx" 2>/dev/null | grep -iE "token|jwt|secret|key" | head -20

# Webhook endpoints — verify each has signature verification
grep -rn "webhook" src/modules --include="*.controller.ts" --include="*.resolver.ts" | grep -v spec | head -20

# LLM prompt assembly from external content (prompt-injection review list)
grep -rn "systemPrompt\|buildPrompt\|prompt +=" src/modules --include="*.ts" | grep -v spec | head -20
```

---

## 3. Performance Audit

### 2.1 Bundle Size

| Check | Severity | Threshold |
|-------|----------|-----------|
| First Load JS | High | < 100KB per route |
| Total bundle size | Medium | < 250KB gzipped |
| Large dependencies | Medium | No single dep > 50KB |

**Audit Commands:**
```bash
# Analyze bundle
npm run build
npx @next/bundle-analyzer

# Check bundle size
ls -la .next/static/chunks/
```

**Common Issues:**
- Importing entire libraries (`import _ from 'lodash'` vs `import debounce from 'lodash/debounce'`)
- Large icon libraries
- Unused dependencies
- Missing code splitting

### 2.2 Image Optimization

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Next.js Image component | High | Using `<img>` instead of `<Image>` |
| Image sizing | Medium | Missing width/height or sizes prop |
| Format optimization | Medium | Not using WebP/AVIF |
| Lazy loading | Low | Above-fold images not prioritized |

**Files to Check:**
```
src/components/**/*.tsx
src/app/**/*.tsx
next.config.js (images.remotePatterns)
```

**Checklist:**
- [ ] All images use Next.js `<Image>` component
- [ ] Remote image domains configured
- [ ] Appropriate sizes prop for responsive images
- [ ] `priority` set for LCP images
- [ ] Images are appropriately compressed

### 2.3 Rendering Strategy

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Unnecessary client components | High | `'use client'` on components that don't need it |
| Missing static generation | Medium | Dynamic pages that could be static |
| N+1 queries | High | Multiple sequential data fetches |
| Missing caching | Medium | No revalidation strategy |

**Files to Check:**
```
src/app/**/page.tsx
src/lib/content/
src/lib/queries/
```

**Checklist:**
- [ ] Server Components used by default
- [ ] Client Components only where needed (interactivity, hooks)
- [ ] Parallel data fetching with `Promise.all()`
- [ ] Appropriate `revalidate` or `dynamic` settings
- [ ] Streaming/Suspense for slow data

### 2.4 Core Web Vitals

| Metric | Target | Severity if Failed |
|--------|--------|-------------------|
| LCP (Largest Contentful Paint) | < 2.5s | High |
| FID (First Input Delay) | < 100ms | High |
| CLS (Cumulative Layout Shift) | < 0.1 | Medium |
| TTFB (Time to First Byte) | < 800ms | Medium |

**Audit Commands:**
```bash
# Run Lighthouse
npx lighthouse http://localhost:3000 --output=json --output-path=./lighthouse-report.json

# Or use Chrome DevTools Lighthouse tab
```

**Common Fixes:**
- LCP: Optimize hero images, preload fonts
- FID: Reduce JavaScript, defer non-critical scripts
- CLS: Set explicit dimensions on images/embeds
- TTFB: Optimize server response, use CDN

---

## 4. Code Quality Audit

### 4.1 TypeScript Strict Mode

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Strict mode enabled | High | `"strict": true` in tsconfig.json |
| No `any` types | Medium | Explicit or implicit `any` usage |
| No ts-ignore | Medium | `@ts-ignore` or `@ts-nocheck` comments |
| Proper null handling | Medium | Missing null checks |

**Files to Check:**
```
tsconfig.json
src/**/*.ts
src/**/*.tsx
```

**Audit Commands:**
```bash
# Check for any types
grep -r ": any" --include="*.ts" --include="*.tsx" src/

# Check for ts-ignore
grep -r "@ts-ignore\|@ts-nocheck" --include="*.ts" --include="*.tsx" src/

# Run type check
npx tsc --noEmit
```

**Required tsconfig.json Settings:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

### 4.2 Linting & Formatting

| Check | Severity | What to Look For |
|-------|----------|------------------|
| ESLint configured | Medium | Missing or incomplete ESLint config |
| No lint errors | Medium | `npm run lint` fails |
| Prettier configured | Low | Inconsistent formatting |
| Pre-commit hooks | Low | No husky/lint-staged |

**Audit Commands:**
```bash
# Run linting
npm run lint

# Check for ESLint config
cat .eslintrc.json || cat .eslintrc.js || cat eslint.config.js
```

### 4.3 Code Organization

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Component size | Medium | Components > 300 lines |
| File organization | Low | Inconsistent file structure |
| Naming conventions | Low | Inconsistent naming |
| Dead code | Low | Unused exports, functions, variables |

**Checklist:**
- [ ] Components are focused and single-purpose
- [ ] Consistent file naming (kebab-case or PascalCase)
- [ ] Logical folder structure
- [ ] No commented-out code blocks
- [ ] No unused imports

---

## 5. Developer Experience & Code Consistency Audit

> **Early-Stage Priority:** This section is critical for new projects. Establishing clean, consistent patterns early prevents technical debt and makes the codebase easier to maintain as the team grows.

### 5.1 Module Structure Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Consistent module layout | High | Modules organized differently (some flat, some nested) |
| Index/barrel exports | High | Some modules export via index.ts, others don't |
| File colocation | High | Related files scattered (entity separate from its DTOs) |
| Naming alignment | High | Module name doesn't match folder/file naming |

**Expected Module Structure (NestJS Example):**
```
src/modules/
├── user/
│   ├── index.ts                 # Barrel export
│   ├── user.module.ts           # Module definition
│   ├── entities/
│   │   └── user.entity.ts
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   ├── services/
│   │   └── user.service.ts
│   ├── resolvers/               # or controllers/
│   │   └── user.resolver.ts
│   └── __tests__/               # or *.spec.ts colocated
│       └── user.service.spec.ts
├── auth/                        # Same structure
├── content/                     # Same structure
└── media/                       # Same structure
```

**Anti-patterns to Flag:**
```
❌ Inconsistent structure across modules:
modules/
├── user/
│   ├── user.service.ts          # Flat structure
│   ├── user.controller.ts
│   └── user.entity.ts
├── auth/
│   ├── services/                # Nested structure
│   │   └── auth.service.ts
│   ├── controllers/
│   │   └── auth.controller.ts
│   └── entities/
│       └── auth.entity.ts

❌ Missing index.ts in some modules but not others
❌ Some modules use singular names (user/) others plural (users/)
```

**Audit Commands:**
```bash
# Compare module structures
for dir in src/modules/*/; do echo "=== $dir ===" && ls -la "$dir"; done

# Find modules missing index.ts
find src/modules -maxdepth 2 -type d ! -exec test -e '{}/index.ts' \; -print

# Check for inconsistent nesting
find src/modules -type f -name "*.service.ts" | head -10
find src/modules -type f -name "*.entity.ts" | head -10
```

### 5.2 Naming Convention Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| File naming | High | Mixed kebab-case and PascalCase files |
| Class naming | High | Inconsistent suffixes (Service vs Svc, Controller vs Ctrl) |
| Variable naming | Medium | Mixed camelCase and snake_case |
| Database columns | Medium | Inconsistent column naming (camelCase vs snake_case) |
| GraphQL fields | Medium | Inconsistent field naming across types |

**Naming Conventions Matrix:**

| Element | Convention | Example |
|---------|------------|---------|
| Files | kebab-case | `user-profile.service.ts` |
| Classes | PascalCase + Suffix | `UserProfileService` |
| Interfaces | PascalCase (no I prefix) | `UserProfile` |
| Variables | camelCase | `userProfile` |
| Constants | SCREAMING_SNAKE | `MAX_RETRY_COUNT` |
| Database tables | snake_case (plural) | `user_profiles` |
| Database columns | snake_case | `created_at` |
| GraphQL types | PascalCase | `UserProfile` |
| GraphQL fields | camelCase | `createdAt` |
| Enum values | PascalCase or SCREAMING_SNAKE | `Active` or `ACTIVE` |

**Anti-patterns to Flag:**
```typescript
// ❌ Inconsistent file naming
user-profile.service.ts    // kebab-case
UserProfile.entity.ts      // PascalCase
userProfile.dto.ts         // camelCase

// ❌ Inconsistent class suffixes
class UserService {}       // "Service"
class AuthSvc {}           // "Svc"
class ContentManager {}    // "Manager"

// ❌ Mixed conventions in same entity
@Entity('UserProfiles')    // PascalCase table
class UserProfile {
  @Column()
  firstName: string;       // camelCase property

  @Column({ name: 'last_name' })
  lastName: string;        // snake_case in DB, camelCase in code (OK)

  @Column()
  created_at: Date;        // snake_case property (inconsistent!)
}
```

**Audit Commands:**
```bash
# Check for mixed file naming patterns
find src -name "*.ts" | xargs -I {} basename {} | sort | uniq -c | sort -rn

# Find files not following kebab-case
find src -name "*.ts" | grep -E "[A-Z]" | grep -v node_modules

# Check for inconsistent class suffixes
grep -r "class.*Service\|class.*Svc\|class.*Manager" --include="*.ts" src/
```

### 5.3 Import/Export Pattern Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Import organization | Medium | Unordered, ungrouped imports |
| Barrel exports | High | Inconsistent use of index.ts re-exports |
| Circular dependencies | Critical | Modules importing each other |
| Relative vs absolute | Medium | Mixed import path styles |
| Named vs default exports | Medium | Inconsistent export style |

**Expected Import Order:**
```typescript
// 1. Node built-ins
import { readFile } from 'fs';

// 2. External packages (npm)
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';

// 3. Internal aliases (@/ paths)
import { DatabaseService } from '@/database';
import { BaseEntity } from '@/common';

// 4. Relative imports (parent first, then siblings)
import { UserEntity } from '../entities';
import { CreateUserDto } from './dto';
```

**Anti-patterns to Flag:**
```typescript
// ❌ Deep relative imports (use barrel exports or aliases)
import { UserEntity } from '../../../modules/user/entities/user.entity';

// ❌ Importing from index when you could import directly (in same module)
import { UserService } from './index';  // Should be './user.service'

// ❌ Inconsistent export style within project
// file1.ts: export default class Foo {}
// file2.ts: export class Bar {}

// ❌ Circular dependency
// user.service.ts imports auth.service.ts
// auth.service.ts imports user.service.ts
```

**Audit Commands:**
```bash
# Find potential circular dependencies
npx madge --circular src/

# Check for deep relative imports
grep -r "from '\.\./\.\./\.\." --include="*.ts" src/

# Find default exports (should use named exports for consistency)
grep -r "export default" --include="*.ts" src/
```

### 5.4 Error Handling Pattern Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Exception types | High | Mixed custom and generic exceptions |
| Error messages | High | Inconsistent error message format |
| Error codes | Medium | No standardized error codes |
| Try-catch patterns | High | Inconsistent error handling approaches |
| Logging on errors | Medium | Some errors logged, others not |

**Expected Error Pattern:**
```typescript
// Consistent custom exception hierarchy
// src/common/exceptions/
export class BaseException extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly statusCode: number = 500,
  ) {
    super(message);
  }
}

export class NotFoundException extends BaseException {
  constructor(resource: string, identifier: string) {
    super('NOT_FOUND', `${resource} with id '${identifier}' not found`, 404);
  }
}

export class ValidationException extends BaseException {
  constructor(message: string, public readonly errors: ValidationError[]) {
    super('VALIDATION_ERROR', message, 400);
  }
}

// Consistent usage across services
throw new NotFoundException('User', userId);
throw new ValidationException('Invalid input', errors);
```

**Anti-patterns to Flag:**
```typescript
// ❌ Inconsistent exception throwing
// Service A:
throw new Error('User not found');

// Service B:
throw new NotFoundException('User not found');

// Service C:
throw new HttpException('User not found', 404);

// ❌ Inconsistent error message formats
throw new Error('user not found');           // lowercase
throw new Error('User Not Found');           // Title Case
throw new Error('USER_NOT_FOUND');           // SCREAMING
throw new Error('User with ID 123 not found'); // with ID
throw new Error('Cannot find user');         // different phrasing
```

**Audit Commands:**
```bash
# Find all throw statements to check consistency
grep -r "throw new" --include="*.ts" src/ | head -30

# Check for generic Error usage (should use custom exceptions)
grep -r "throw new Error(" --include="*.ts" src/

# Find HttpException usage (check for consistency)
grep -r "HttpException\|NotFoundException\|BadRequestException" --include="*.ts" src/
```

### 5.5 Type Definition Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Type location | High | Types scattered vs centralized |
| Interface vs Type | Medium | Inconsistent use of interface vs type |
| Optional properties | Medium | Inconsistent use of ? vs undefined |
| Null handling | High | Mixed null and undefined usage |
| DTO validation | High | Some DTOs validated, others not |

**Expected Type Organization:**
```typescript
// Shared types in types/ or common/
// src/common/types/
export interface PaginationParams {
  page: number;
  limit: number;
}

export interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  totalPages: number;
}

// Module-specific types colocated
// src/modules/user/types/
export interface UserFilters {
  status?: UserStatus;
  role?: UserRole;
}

// Consistent DTO pattern
export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  email: string;

  @IsOptional()
  @IsString()
  bio?: string;
}
```

**Anti-patterns to Flag:**
```typescript
// ❌ Inconsistent interface vs type usage
interface User { ... }    // Some use interface
type Product = { ... }    // Others use type alias

// ❌ Inconsistent optional handling
interface Config {
  apiKey?: string;        // Optional with ?
  timeout: number | undefined;  // Optional with | undefined
  retries: number | null;       // Nullable (different meaning!)
}

// ❌ DTOs without validation in some modules
// user.dto.ts - has validation decorators
// product.dto.ts - no validation decorators
```

**Audit Commands:**
```bash
# Check for interface vs type consistency
grep -r "^interface\|^export interface" --include="*.ts" src/ | wc -l
grep -r "^type\|^export type" --include="*.ts" src/ | wc -l

# Find DTOs without class-validator decorators
find src -name "*.dto.ts" -exec grep -L "@Is\|@Min\|@Max\|@Valid" {} \;

# Check for mixed null/undefined
grep -r "| null\|| undefined" --include="*.ts" src/ | head -20
```

### 5.6 Service Pattern Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Method naming | High | Inconsistent CRUD method names |
| Return types | High | Some return entities, others DTOs |
| Transaction handling | High | Inconsistent transaction patterns |
| Dependency injection | Medium | Some deps injected, others instantiated |
| Method signatures | Medium | Inconsistent parameter ordering |

**Expected Service Pattern:**
```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly eventEmitter: EventEmitter,  // Injected, not instantiated
  ) {}

  // Consistent CRUD naming
  async create(dto: CreateUserDto): Promise<User> { ... }
  async findAll(filters: UserFilters): Promise<User[]> { ... }
  async findOne(id: string): Promise<User> { ... }
  async findOneOrFail(id: string): Promise<User> { ... }  // Throws if not found
  async update(id: string, dto: UpdateUserDto): Promise<User> { ... }
  async remove(id: string): Promise<void> { ... }

  // Consistent parameter order: identifier first, then data
  async updateStatus(id: string, status: UserStatus): Promise<User> { ... }
}
```

**Anti-patterns to Flag:**
```typescript
// ❌ Inconsistent CRUD naming across services
// UserService:
async create() {}
async getAll() {}
async getById() {}

// ProductService:
async createProduct() {}
async findAll() {}
async findOne() {}

// OrderService:
async add() {}
async list() {}
async fetch() {}

// ❌ Inconsistent return types
async findUser(id): Promise<User> {}      // Returns entity
async findProduct(id): Promise<ProductDto> {}  // Returns DTO

// ❌ Instantiating dependencies instead of injection
class UserService {
  private logger = new Logger();  // ❌ Should be injected
}
```

**Audit Commands:**
```bash
# Compare method names across services
grep -r "async create\|async find\|async get\|async update\|async delete\|async remove" --include="*.service.ts" src/

# Check for new keyword in services (potential DI violation)
grep -r "new.*(" --include="*.service.ts" src/ | grep -v "new Date\|new Error\|new Map\|new Set"
```

### 5.7 Test Pattern Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Test file location | Medium | Mixed colocated and centralized tests |
| Test naming | Medium | Inconsistent describe/it descriptions |
| Mock patterns | High | Different mocking approaches per module |
| Setup/teardown | Medium | Inconsistent beforeEach/afterEach usage |
| Coverage gaps | High | Some modules tested, others not |

**Expected Test Pattern:**
```typescript
// Consistent test file naming: *.spec.ts (unit) or *.e2e-spec.ts (e2e)
// Colocated: user.service.spec.ts next to user.service.ts
// Or centralized: __tests__/user.service.spec.ts

describe('UserService', () => {
  let service: UserService;
  let repository: MockType<Repository<User>>;

  // Consistent setup
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(User),
          useFactory: repositoryMockFactory,
        },
      ],
    }).compile();

    service = module.get(UserService);
    repository = module.get(getRepositoryToken(User));
  });

  // Consistent describe structure
  describe('create', () => {
    it('should create a user with valid data', async () => { ... });
    it('should throw ValidationException for invalid email', async () => { ... });
  });

  describe('findOne', () => {
    it('should return user when found', async () => { ... });
    it('should throw NotFoundException when not found', async () => { ... });
  });
});
```

**Anti-patterns to Flag:**
```typescript
// ❌ Inconsistent test descriptions
it('creates user')           // No "should"
it('should create a user')   // With "should"
it('user creation works')    // Different style

// ❌ Inconsistent mocking
// Test A: Uses jest.mock()
// Test B: Uses manual mock objects
// Test C: Uses @nestjs/testing mock factories

// ❌ Missing tests for some modules
// user.service.spec.ts ✓
// product.service.spec.ts ✓
// order.service.spec.ts ✗ (missing!)
```

**Audit Commands:**
```bash
# Find modules missing tests
for service in $(find src -name "*.service.ts" | grep -v spec); do
  spec="${service%.ts}.spec.ts"
  [ ! -f "$spec" ] && echo "Missing: $spec"
done

# Check test description consistency
grep -r "it('" --include="*.spec.ts" src/ | head -20
```

### 5.8 Documentation & Comments Consistency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| JSDoc on public APIs | Medium | Some services documented, others not |
| README per module | Low | Inconsistent module documentation |
| Inline comments | Low | Over-commented or under-commented |
| TODO/FIXME tracking | Medium | Stale TODOs, no tracking system |

**Expected Documentation Pattern:**
```typescript
/**
 * Service for managing user accounts and authentication.
 *
 * @remarks
 * All methods require the caller to have appropriate permissions.
 * User data is automatically scoped to the current tenant.
 */
@Injectable()
export class UserService {
  /**
   * Creates a new user account.
   *
   * @param dto - The user creation data
   * @returns The created user entity
   * @throws {ValidationException} When email is already taken
   * @throws {ForbiddenException} When caller lacks permission
   */
  async create(dto: CreateUserDto): Promise<User> { ... }
}
```

**Anti-patterns to Flag:**
```typescript
// ❌ Obvious comments that don't add value
// Increment the counter
counter++;

// ❌ Outdated comments
// TODO: Implement validation (but validation exists)
// Returns user by ID (but method returns Promise<User[]>)

// ❌ Stale TODOs with no tracking
// TODO: Fix this later
// FIXME: Temporary hack
// HACK: Need to refactor
```

**Audit Commands:**
```bash
# Find TODOs and FIXMEs
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.ts" src/

# Check JSDoc coverage on services
grep -l "@Injectable" src/**/*.service.ts | while read f; do
  grep -q "/\*\*" "$f" && echo "✓ $f" || echo "✗ $f"
done
```

### 5.9 Code Duplication & DRY Violations

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Duplicate logic | High | Same code copy-pasted across modules |
| Similar functions across services | High | Same functionality implemented separately in each service |
| Similar DTOs | Medium | Nearly identical DTOs not shared |
| Repeated queries | Medium | Same database queries in multiple places |
| Utility sprawl | Medium | Similar helper functions scattered |

**Cross-Service Function Duplication:**

This is a common and critical issue. When multiple services need the same functionality, developers often implement it separately in each service rather than creating a shared utility.

```typescript
// ❌ BAD: Search logic implemented separately in each service

// user.service.ts
async findAll(filter: UserFilter) {
  const whereConditions: FindOptionsWhere<User>[] = [];
  if (filter.search) {
    const term = `%${filter.search.trim()}%`;
    whereConditions.push({ name: ILike(term) });
    whereConditions.push({ email: ILike(term) });
  }
  // ... rest of query
}

// product.service.ts - SAME LOGIC DUPLICATED!
async findAll(filter: ProductFilter) {
  const whereConditions: FindOptionsWhere<Product>[] = [];
  if (filter.search) {
    const term = `%${filter.search.trim()}%`;
    whereConditions.push({ name: ILike(term) });
    whereConditions.push({ slug: ILike(term) });
  }
  // ... rest of query
}

// ✅ GOOD: Centralized search utility
// common/utils/search.utils.ts
export function buildSearchConditions<T>(
  baseWhere: Record<string, unknown>,
  searchTerm: string,
  fields: string[],
): FindOptionsWhere<T>[] {
  if (!searchTerm?.trim()) return [baseWhere as FindOptionsWhere<T>];
  const term = `%${searchTerm.trim()}%`;
  return fields.map((field) => ({
    ...baseWhere,
    [field]: ILike(term),
  }) as FindOptionsWhere<T>);
}

export function hasSearchTerm(search?: string): boolean {
  return Boolean(search?.trim());
}

// Usage in services:
const whereConditions = hasSearchTerm(filter.search)
  ? buildSearchConditions<User>(baseWhere, filter.search!, ['name', 'email'])
  : [baseWhere as FindOptionsWhere<User>];
```

**Common Patterns to Check for Centralization:**
- Search/filter logic
- Pagination utilities
- Date formatting/parsing
- Validation helpers
- Access control checks
- Error message formatting
- Slug generation
- JSONB field queries

**Common Duplication Patterns:**
```typescript
// ❌ Pagination logic repeated
// user.service.ts
const skip = (page - 1) * limit;
const [items, total] = await this.repo.findAndCount({ skip, take: limit });
return { items, total, page, totalPages: Math.ceil(total / limit) };

// product.service.ts - same code!
const skip = (page - 1) * limit;
const [items, total] = await this.repo.findAndCount({ skip, take: limit });
return { items, total, page, totalPages: Math.ceil(total / limit) };

// ✅ Extract to shared utility
// common/utils/pagination.ts
export function paginate<T>(items: T[], total: number, page: number, limit: number) {
  return { items, total, page, totalPages: Math.ceil(total / limit) };
}
```

**Audit Commands:**
```bash
# Find similar code blocks (using jscpd)
npx jscpd src/ --min-lines 5 --min-tokens 50

# Find duplicate string patterns
grep -roh "throw new.*Exception" --include="*.ts" src/ | sort | uniq -c | sort -rn

# Find similar method signatures
grep -r "async find.*(" --include="*.service.ts" src/

# Look for similar search/filter implementations
grep -r "ILike\|ILIKE" --include="*.service.ts" src/ | head -20

# Check for similar validation patterns
grep -r "trim()\|toLowerCase()" --include="*.service.ts" src/ | head -20
```

**Action:** When you find similar functions across 2+ services, extract to `src/common/utils/` and update all services to use the shared utility.

### 5.10 Developer Experience Checklist

**Module Structure:**
- [ ] All modules follow identical directory structure
- [ ] Barrel exports (index.ts) in every module
- [ ] Related files colocated (entity + DTOs + tests)
- [ ] Consistent module naming (singular vs plural)

**Naming Conventions:**
- [ ] File naming follows single convention (kebab-case recommended)
- [ ] Class suffixes consistent (Service, Controller, Resolver, etc.)
- [ ] Database naming follows convention (snake_case recommended)
- [ ] No mixed camelCase/snake_case in same layer

**Code Patterns:**
- [ ] CRUD methods named consistently across all services
- [ ] Error handling uses same exception hierarchy
- [ ] All DTOs have validation decorators
- [ ] Transaction patterns consistent

**Imports & Exports:**
- [ ] Import order consistent (externals → internals → relatives)
- [ ] No circular dependencies
- [ ] No deep relative imports (use aliases or barrels)
- [ ] Consistent use of named exports

**Testing:**
- [ ] Every service has corresponding spec file
- [ ] Test descriptions follow consistent pattern
- [ ] Mock setup standardized across tests
- [ ] No test files with skipped tests (.skip)

**Documentation:**
- [ ] Public APIs have JSDoc comments
- [ ] No stale TODO/FIXME comments
- [ ] Module-level README if complex

**Code Reuse:**
- [ ] No copy-pasted logic between modules
- [ ] Common utilities extracted to shared location
- [ ] Similar DTOs consolidated or inherited

---

## 6. Error Handling & Reliability Audit

> Reliability means the app fails visibly, recoverably, and consistently. Audit not just
> whether errors are caught, but what the user and the data look like AFTER the failure.

### 6.1 Error Boundaries

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Global error boundary | High | Missing `app/error.tsx` |
| Not found handling | Medium | Missing `app/not-found.tsx` |
| Segment error boundaries | Low | Missing error.tsx in route segments |

**Required Files:**
```
src/app/error.tsx        # Global error boundary
src/app/not-found.tsx    # 404 page
src/app/global-error.tsx # Root layout errors
```

**Example error.tsx:**
```tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

### 6.2 API Error Handling

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Try-catch blocks | High | Unhandled promise rejections |
| Error responses | Medium | Inconsistent error response format |
| Logging | Medium | Errors not logged |
| User feedback | Medium | Silent failures |

**Files to Check:**
```
src/app/api/**/route.ts
src/lib/content/
src/lib/queries/
```

**Checklist:**
- [ ] All async operations wrapped in try-catch
- [ ] Consistent error response format
- [ ] Errors logged with context
- [ ] User-friendly error messages displayed
- [ ] Retry logic for transient failures

### 6.3 Form Validation

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Client-side validation | Medium | Forms submit without validation |
| Server-side validation | High | API accepts invalid data |
| Error display | Medium | Validation errors not shown to user |

**Checklist:**
- [ ] Required fields validated
- [ ] Email/phone formats validated
- [ ] Server validates all input (never trust client)
- [ ] Clear error messages displayed
- [ ] Form state preserved on error

### 6.4 Async Failure Modes

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Unhandled promise rejections | High | Fire-and-forget async calls with no .catch or surrounding try/catch |
| Swallowed errors | High | catch blocks that are empty, or log-and-continue where the flow must stop |
| Silent failures | High | Operations that fail without any user feedback or log entry |
| Broken retry loops | High | Retries with no backoff or cap; retrying non-idempotent operations |
| Incomplete rollback | High | Multi-step operations failing midway with no transaction or compensation — partial state persists |
| Timing/ordering assumptions | Medium | Code assuming responses arrive in request order, or that a dependent write has landed |

**Audit Commands:**
```bash
# Fire-and-forget promises (async calls whose result is discarded)
grep -rn "void this\.\|\.then(" src --include="*.ts" | grep -v "catch\|spec" | head -20

# Empty or swallowing catch blocks (manual review list)
grep -rn -A2 "} catch" src --include="*.ts" | grep -v spec | head -30
```

### 6.5 UI Failure & Progress States

Every data-driven view and every async action needs an explicit answer for ALL of these states — a missing state is a finding:

| State | What "missing" looks like in production |
|-------|------------------------------------------|
| Loading | Blank screen or layout jump while fetching |
| Empty | Zero-item view renders nothing or a broken layout |
| Error | Failure leaves an infinite spinner or a dead screen |
| Offline / network loss | Action fails with no feedback; auth state or form data wiped |
| Timeout | Request hangs forever — no abort, no message, no retry path |
| Partial success | Batch operation reports all-or-nothing when some items succeeded (user loses the successful ones) |

**Trace test:** for each mutation reachable from the UI, ask "what does the user see if this fails?" If the answer is "nothing" or "infinite spinner", that is a High finding.

### 6.6 Assumption Audit

Flag code on production paths that assumes: API responses always match the expected shape (no runtime validation at the boundary), arrays are non-empty, fields are non-null, external services respond quickly, or the network is available. Each unguarded assumption is a finding — severity scaled by blast radius (a crash in a deploy/render path outranks a broken widget).

---

## 7. Accessibility Audit

> **Audit against WCAG 2.2 AA.** The grep checks below find the cheap issues; the real
> audit is systematic — walk the key flows (booking, forms, modals, navigation) keyboard-only,
> then reason through what a screen reader announces at every dynamic change.

### 7.1 Semantic HTML

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Heading hierarchy | High | Skipped heading levels (h1 → h3) |
| Landmark regions | Medium | Missing main, nav, footer elements |
| Button vs link | Medium | `<div onClick>` instead of button/link |
| Lists | Low | Items not in ul/ol |

**Audit Commands:**
```bash
# Check for div with onClick (should be button)
grep -r "div.*onClick\|span.*onClick" --include="*.tsx" src/

# Check heading usage
grep -r "<h[1-6]" --include="*.tsx" src/
```

### 7.2 ARIA & Labels

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Image alt text | High | Images without alt attribute |
| Form labels | High | Inputs without associated labels |
| ARIA labels | Medium | Interactive elements without accessible names |
| Focus management | Medium | Focus not managed in modals/dialogs |

**Audit Commands:**
```bash
# Check for images without alt
grep -r "<Image\|<img" --include="*.tsx" src/ | grep -v "alt="

# Check for inputs without labels
grep -r "<input\|<Input" --include="*.tsx" src/
```

### 7.3 Keyboard Navigation

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Tab order | High | Illogical tab order |
| Focus visible | High | Focus indicator removed |
| Keyboard traps | High | Can't escape modal with keyboard |
| Skip links | Medium | No skip to main content link |

**Checklist:**
- [ ] All interactive elements focusable
- [ ] Visible focus indicators
- [ ] Logical tab order
- [ ] Escape closes modals
- [ ] Skip link for main content

### 7.4 Color & Contrast

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Color contrast | High | Text contrast ratio < 4.5:1 |
| Color-only info | Medium | Information conveyed only by color |
| Focus contrast | Medium | Focus indicator not visible |

**Tools:**
- Chrome DevTools Accessibility panel
- axe DevTools extension
- Lighthouse accessibility audit

### 7.5 Mobile Touch Interactions

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Touch target size | High | Interactive elements < 44x44px |
| Touch + drag + click conflicts | High | Elements with drag behavior that also need click |
| preventDefault on touch | High | `touchstart` preventDefault blocking click events |
| Touch-action CSS | Medium | `touch-action: none` without explicit tap handling |

**The Problem:**
When an element has both drag and click behavior (e.g., a draggable button), mobile touch events can interfere with click detection:
1. `touchstart` fires → drag tracking begins
2. `touchend` fires → drag tracking ends
3. `click` event may NOT fire if `preventDefault()` was called on `touchstart`

**Anti-pattern to Flag:**
```javascript
// ❌ BAD: Click won't fire reliably on mobile
element.addEventListener('touchstart', (e) => {
  e.preventDefault(); // This can block the click event!
  startDrag(e);
}, { passive: false });

element.addEventListener('click', () => {
  doAction(); // May never fire on mobile!
});
```

**Correct Pattern:**
```javascript
// ✅ GOOD: Explicit tap detection in touchend
let touchStartTime = 0;
let hasMoved = false;

element.addEventListener('touchstart', (e) => {
  touchStartTime = Date.now();
  hasMoved = false;
  e.preventDefault();
  startDrag(e);
}, { passive: false });

element.addEventListener('touchmove', (e) => {
  hasMoved = true;
  handleDrag(e);
});

element.addEventListener('touchend', (e) => {
  endDrag();
  // Manually trigger click if it was a tap (not a drag)
  if (!hasMoved && (Date.now() - touchStartTime) < 300) {
    doAction(); // Explicit tap handling
  }
});
```

**Audit Commands:**
```bash
# Find elements with both touchstart and click handlers
grep -rn "addEventListener.*touchstart" --include="*.js" --include="*.ts" src/

# Check for preventDefault on touch events
grep -rn "preventDefault.*touch\|touch.*preventDefault" --include="*.js" --include="*.ts" src/

# Find touch-action: none in CSS (needs explicit tap handling)
grep -rn "touch-action.*none" --include="*.js" --include="*.ts" --include="*.css" src/
```

**Checklist:**
- [ ] Touch targets are at least 44x44px
- [ ] Elements with drag + click have explicit tap handling in `touchend`
- [ ] `preventDefault()` on `touchstart` is paired with manual click triggering
- [ ] `touch-action: none` elements have tap detection logic
- [ ] Test all interactive elements on actual mobile device

### 7.6 Focus Management & Dynamic Content

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Focus trapping | High | Modals/drawers that let Tab escape to the page behind them |
| Focus restoration | High | Closing a modal/menu drops focus to `<body>` instead of returning it to the trigger |
| Toasts & async announcements | Medium | Status/success/error toasts not rendered in an aria-live region — invisible to screen readers |
| Validation errors | High | Errors shown visually but not associated via aria-describedby or announced on submit |
| Loading states | Medium | Spinners with no accessible text; content swaps that happen unannounced |
| Custom widgets (menus/dropdowns/tabs) | High | Widgets without their ARIA pattern: roles, aria-expanded/selected, arrow-key navigation |

**Audit Commands:**
```bash
# Live regions present anywhere toasts/status messages render?
grep -rn "aria-live\|role=\"status\"\|role=\"alert\"" admin packages --include="*.tsx" 2>/dev/null | head

# Custom widget ARIA adoption
grep -rn "aria-expanded\|aria-selected\|role=\"menu\|role=\"tab" admin packages --include="*.tsx" 2>/dev/null | head
```

### 7.7 Forms, Zoom & Motion

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Autocomplete attributes | Medium | Name/email/address/phone fields without autocomplete= (WCAG 1.3.5) |
| Instructions & error recovery | Medium | Required formats only revealed after a failed submit; errors that clear the user's input |
| 200% zoom | High | Layout breaks, clipped content, or lost functionality at 200% browser zoom |
| Large text settings | Medium | Fixed-height containers truncating scaled-up text |
| Reduced motion | Medium | Animations/transitions that ignore prefers-reduced-motion |

**Audit Commands:**
```bash
grep -rn "autoComplete\|autocomplete=" admin packages --include="*.tsx" 2>/dev/null | head
grep -rn "prefers-reduced-motion" admin packages --include="*.tsx" --include="*.css" 2>/dev/null | head
```

---

## 8. SEO Audit

### 8.1 Meta Tags

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Title tags | High | Missing or duplicate titles |
| Meta descriptions | Medium | Missing or too long (> 160 chars) |
| Open Graph | Medium | Missing OG tags for social sharing |
| Canonical URLs | Medium | Missing canonical for duplicate content |

**Files to Check:**
```
src/app/layout.tsx
src/app/**/page.tsx
src/app/**/layout.tsx
```

**Required Metadata:**
```tsx
// src/app/layout.tsx
export const metadata: Metadata = {
  title: {
    default: 'Site Name',
    template: '%s | Site Name',
  },
  description: 'Site description',
  openGraph: {
    title: 'Site Name',
    description: 'Site description',
    url: 'https://example.com',
    siteName: 'Site Name',
    type: 'website',
  },
  twitter: {
    card: 'summary_large_image',
  },
};
```

### 8.2 Structured Data

| Check | Severity | What to Look For |
|-------|----------|------------------|
| JSON-LD | Medium | Missing structured data |
| Schema validity | Medium | Invalid schema markup |
| Relevant schemas | Low | Missing business-specific schemas |

**Common Schemas:**
- Organization
- LocalBusiness
- BreadcrumbList
- Article
- Product
- FAQ

### 8.3 Technical SEO

| Check | Severity | What to Look For |
|-------|----------|------------------|
| robots.txt | Medium | Missing or misconfigured |
| sitemap.xml | Medium | Missing sitemap |
| Mobile-friendly | High | Not responsive |
| Page speed | High | Slow loading |

**Required Files:**
```
public/robots.txt
app/sitemap.ts (or sitemap.xml)
```

---

## 9. Environment & Configuration Audit

### 9.1 Environment Files

| Check | Severity | What to Look For |
|-------|----------|------------------|
| .env.example | Medium | No example env file for onboarding |
| Env validation | Medium | No schema validation for env vars |
| Production values | High | Dev values in production config |

**Required Files:**
```
.env.example          # Template with placeholder values
.env.local           # Local development (gitignored)
.env.production      # Production overrides (if needed)
```

**Env Validation Example (using zod):**
```typescript
// src/lib/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NEXT_PUBLIC_CMS_GRAPHQL_URL: z.string().url(),
  CMS_ORGANIZATION_ID: z.string().uuid(),
  NODE_ENV: z.enum(['development', 'production', 'test']),
});

export const env = envSchema.parse(process.env);
```

### 9.2 Next.js Configuration

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Image domains | High | Remote images not configured |
| Redirects | Medium | Old URLs not redirecting |
| Headers | Medium | Security headers not set |
| Rewrites | Low | API proxying if needed |

**Files to Check:**
```
next.config.js
next.config.mjs
```

---

## 10. Dependencies Audit

### 10.1 Security Vulnerabilities

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Known vulnerabilities | Critical | `npm audit` findings |
| Outdated packages | Medium | Major versions behind |
| Deprecated packages | Medium | Using deprecated libraries |

**Audit Commands:**
```bash
# Check for vulnerabilities
npm audit

# Check for outdated packages
npm outdated

# Update to latest (be careful)
npm update
```

### 10.2 Dependency Hygiene

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Unused dependencies | Low | Packages in package.json not imported |
| Dev vs prod deps | Low | Dev dependencies in production |
| Lock file | Medium | Missing or outdated package-lock.json |

**Audit Commands:**
```bash
# Find unused dependencies
npx depcheck

# Verify lock file is in sync
npm ci
```

### 10.3 License Compliance

| Check | Severity | What to Look For |
|-------|----------|------------------|
| License compatibility | Medium | GPL dependencies in proprietary project |
| License documentation | Low | No license documentation |

**Audit Commands:**
```bash
# Check licenses
npx license-checker --summary
```

---

## 11. Build & Deployment Audit

### 11.1 Build Process

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Build succeeds | Critical | `npm run build` fails |
| No build warnings | Medium | TypeScript/ESLint warnings in build |
| Build time | Low | Excessively long builds |

**Audit Commands:**
```bash
# Run production build
npm run build

# Check build output
ls -la .next/
```

### 11.2 Deployment Configuration

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Vercel config | Medium | Missing vercel.json optimizations |
| Environment vars | Critical | Production env vars not set |
| Domain config | High | Custom domain not configured |
| SSL | Critical | HTTPS not enforced |

**Files to Check:**
```
vercel.json
.vercelignore
```

### 11.3 CI/CD Pipeline

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Automated tests | Medium | No tests run on PR |
| Lint checks | Medium | No linting in CI |
| Preview deployments | Low | No preview for PRs |
| Rollback plan | Medium | No rollback strategy |

---

## 12. Monitoring & Observability Audit

### 12.1 Error Tracking

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Error service | High | No Sentry/similar configured |
| Source maps | Medium | Errors not deobfuscated |
| Alert rules | Medium | No alerts for errors |

**Integration Check:**
```bash
# Check for Sentry
grep -r "sentry" package.json
grep -r "@sentry" src/
```

### 12.2 Analytics

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Analytics configured | Medium | No analytics tracking |
| Privacy compliance | High | No cookie consent |
| Performance monitoring | Medium | No RUM (Real User Monitoring) |

### 12.3 Logging

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Structured logging | Medium | No consistent log format |
| Log levels | Low | Everything at same level |
| PII in logs | High | Sensitive data logged |

---

## Audit Report Template

After completing ALL audit categories and the automated verification suite, generate the final report using the template below. Use fenced code blocks with ASCII `+---+` box-drawing tables (NOT markdown pipe tables). Present ALL tables and content as plain text inside a single fenced code block per section so formatting is preserved regardless of terminal.

The report has 10 sections. Every section is mandatory -- do not skip any. If a section has zero findings, say "No issues found" rather than omitting it.

---

### SECTION 1: Header & Executive Summary

```
+==============================================================+
|            PRODUCTION READINESS AUDIT REPORT                  |
+==============================================================+
| Project:    [PROJECT_NAME]                                    |
| Path:       [PROJECT_PATH]                                    |
| Date:       [YYYY-MM-DD]                                      |
| Auditor:    Claude Code                                       |
| Skill Ver:  [SKILL_VERSION from frontmatter]                  |
+--------------------------------------------------------------+

VERDICT: [READY / NOT READY / CONDITIONAL]

RELEASE RECOMMENDATION: [SAFE TO SHIP / SHIP WITH KNOWN RISKS /
                         DO NOT SHIP]
[One-sentence justification naming the specific findings that
decide it. "Ship with known risks" must list the risks being
accepted; "Do not ship" must name the blocking findings.]

[2-3 sentence plain-English summary of the project's overall
health. Lead with the most important thing the user needs to
know. Mention the total finding count and the worst category.
No jargon.]
```

---

### SECTION 2: Automated Verification Results

Report the actual output of each command run during step 2 of the audit. This is factual data, not analysis.

```
+--------------------------------------------------------------+
|                 AUTOMATED VERIFICATION SUITE                  |
+-------+----------------+--------+----------------------------+
| Order | Command        | Result | Details                    |
+-------+----------------+--------+----------------------------+
|   1   | npm run lint   | PASS   | 0 errors, 3 warnings       |
|   2   | npm test       | PASS   | 142 passed, 0 failed       |
|   3   | npm run test:cov | PASS | Stmts 78% Br 65% Fn 74% Ln 77% |
|   4   | npm run build  | PASS   | Compiled in 34s            |
+-------+----------------+--------+----------------------------+
| E2E (skipped -- not requested)                               |
+--------------------------------------------------------------+
```

If a command FAILED, include:
- The exit code
- The first 20 lines of error output (truncated if longer)
- Whether the failure is blocking (Critical) or informational (Medium)

**Coverage Breakdown** (from test:cov output):

```
+--------------------------------------------------------------+
|                   TEST COVERAGE SUMMARY                       |
+--------------+----------+-----------+--------+----------------+
| Metric       | Actual   | Threshold | Status | Delta vs Last  |
+--------------+----------+-----------+--------+----------------+
| Statements   |  78.2%   |   70%     | PASS   | +2.1%          |
| Branches     |  65.4%   |   60%     | PASS   | -0.3%          |
| Functions    |  74.1%   |   70%     | PASS   | +1.5%          |
| Lines        |  77.8%   |   70%     | PASS   | +2.0%          |
+--------------+----------+-----------+--------+----------------+
```

If this is the first audit (no prior data), put "-- first audit --" in the Delta column.

---

### SECTION 2B: Scope & Coverage Disclosure

State plainly what the audit actually examined, so a reader can tell "clean" from "not looked at." A confidence rating is only trustworthy if the coverage behind it is visible. Do not omit this — an audit that hides its own gaps is worse than one that admits them.

```
+--------------------------------------------------------------+
|                    SCOPE & COVERAGE                           |
+--------------------------------------------------------------+

  FLOWS TRACED END-TO-END:
    [x] Auth / login / session
    [x] Booking / payment path
    [ ] Content publish / deploy      <-- NOT traced (reason)
    [x] Public page render / API
    [ ] ...

  LIVE VERIFICATION PASS:
    Status: [Ran against :3501/:3502/:3580 | Skipped — servers down]
    Security probes executed: [e.g. cross-tenant read attempt on
      bookings + media; guarded-mutation auth checks] or "none"
    Accessibility tools run: [axe-core / Lighthouse on pages X, Y]
      or "none — static only"

  CHECKS THAT WERE STATIC-ONLY (not runtime-verified):
    [List the categories/areas confirmed only by reading code, so
     their findings carry High Confidence at most.]

  AREAS EXPLICITLY OUT OF SCOPE THIS RUN:
    [Anything deliberately not audited, with the reason.]
```

---

### SECTION 3: Health Scorecard

Score each category 0-10 based on findings. The score reflects how healthy that area is (10 = no issues, 0 = critical failures). Use the trend arrow to indicate direction vs. last audit (or "--" if first audit).

```
+--------------------------------------------------------------+
|                      HEALTH SCORECARD                         |
+----+------------------------+-------+-------+--------+-------+
| #  | Category               | Score | Grade | Trend  | Notes |
+----+------------------------+-------+-------+--------+-------+
|  1 | Architecture           |  8/10 |   B+  |   --   |       |
|  2 | Security               |  6/10 |   C+  |   --   |       |
|  3 | Performance            |  7/10 |   B   |   --   |       |
|  4 | Code Quality           |  9/10 |   A   |   --   |       |
|  5 | DX & Consistency       |  7/10 |   B   |   --   |       |
|  6 | Error Handling & Relia.|  5/10 |   C   |   --   |       |
|  7 | Accessibility          |  4/10 |   D+  |   --   |       |
|  8 | SEO                    |  6/10 |   C+  |   --   |       |
|  9 | Configuration          |  9/10 |   A   |   --   |       |
| 10 | Dependencies           |  8/10 |   B+  |   --   |       |
| 11 | Build & Deploy         |  7/10 |   B   |   --   |       |
| 12 | Monitoring             |  3/10 |   D   |   --   |       |
| 13 | Test Coverage          |  7/10 |   B   |   --   |       |
| 14 | Technical Debt         |  6/10 |   C+  |   --   |       |
| 15 | Payment/PCI            |  5/10 |   C   |   --   |       |
| 16 | Concurrency            |  6/10 |   C+  |   --   |       |
| 17 | Load & Capacity        |  5/10 |   C   |   --   |       |
| 18 | Pen-Test Readiness     |  6/10 |   C+  |   --   |       |
| 19 | Client Race Conditions |  6/10 |   C+  |   --   |       |
| 20 | Visual Consistency     |  7/10 |   B   |   --   |       |
| 21 | Responsive & Edge Cases|  7/10 |   B   |   --   |       |
| 22 | Codebase Invariants    |  6/10 |   C+  |   --   |       |
+----+------------------------+-------+-------+--------+-------+
|    | OVERALL                | XX/220|  [G]  |        |       |
+----+------------------------+-------+-------+--------+-------+

Grade scale: A (9-10), B (7-8), C (5-6), D (3-4), F (0-2)
Trend: [UP] improving, [--] stable/first audit, [DN] regressing
```

**Scoring guidelines:**

| Score | Meaning |
|-------|---------|
| 10    | No issues found. Exemplary. |
| 8-9   | Minor/low issues only. Production-ready. |
| 6-7   | Some medium issues. Acceptable but should improve. |
| 4-5   | High-severity issues present. Needs attention. |
| 2-3   | Critical issues present. Not production-ready. |
| 0-1   | Category is fundamentally broken or absent. |

---

### SECTION 4: Risk Heat Map

A quick visual scan of where the worst problems are. Only categories with findings appear here.

```
+--------------------------------------------------------------+
|                       RISK HEAT MAP                           |
+------------------------+------+------+--------+------+-------+
| Category               | Crit | High | Medium | Low  | Total |
+------------------------+------+------+--------+------+-------+
| Security               |  1   |  2   |   0    |  1   |   4   |
| Error Handling         |  0   |  3   |   1    |  0   |   4   |
| Accessibility          |  0   |  1   |   3    |  2   |   6   |
| Monitoring             |  1   |  1   |   0    |  0   |   2   |
| ...                    |      |      |        |      |       |
+------------------------+------+------+--------+------+-------+
| TOTALS                 |  2   |  7   |   4    |  3   |  16   |
+------------------------+------+------+--------+------+-------+
```

Categories with zero findings should be listed at the bottom as:

```
Clean categories (no findings): Architecture, Code Quality,
Configuration, Build & Deploy
```

---

### SECTION 5: Findings by Category

This is the main body of the report. Group findings under their category header, sorted by severity within each category (Critical first, then High, Medium, Low).

Each finding uses this structure:

```
+--------------------------------------------------------------+
|  [CATEGORY #] [CATEGORY NAME]                     Score: X/10 |
+--------------------------------------------------------------+

  [SEV-001] [SEVERITY: CRITICAL/HIGH/MEDIUM/LOW]
  Title: [Short descriptive title]
  
  What: [1-2 sentences describing what was found. Be specific --
        name the file, the pattern, the missing thing.]
  
  Why it matters: [1 sentence in plain English explaining the
        user-visible risk. "If this isn't fixed, X could happen."]
  
  Where: [file path or pattern, e.g., "src/modules/auth/*.ts"
         or "all resolver files"]

  Evidence: [The code path, behavior, or reproducible condition
        supporting the finding. Quote the relevant line(s) or
        describe the exact flow traced. A finding with no
        evidence does not go in the report.]

  Repro: [Steps to trigger or verify the issue, where applicable.
        Omit this line for purely structural findings.]

  Fix: [Concrete recommendation. Not "consider doing X" but
       "add Y to Z". If it's a one-liner, show the fix.
       If it's structural, describe the approach.]
  
  Effort: [Quick (< 30 min) / Moderate (1-3 hrs) / Significant (half day+)]

  Confidence: [Confirmed / High Confidence / Needs Verification]

  ---

  [SEV-002] ...
```

**Finding ID format:** `[SEV-NNN]` where SEV is the severity abbreviation (CRIT, HIGH, MED, LOW) and NNN is a sequential number across the entire report. This lets findings be referenced in the Action Plan.

**Confidence levels (do not inflate — `Confirmed` has a hard bar):**
- `Confirmed` — REQUIRES one of: (a) reproduced live in the running app during the Live Verification pass, (b) a traced execution path with no possible upstream guard, or (c) a defect self-evident on its face (e.g. a secret literally hardcoded in source). "It looks wrong" is never Confirmed. If you did not execute or exhaustively trace it, it is not Confirmed.
- `High Confidence` — strong, specific code evidence, but not executed and not exhaustively traced. Most static-analysis findings land here.
- `Needs Verification` — plausible from the code but dependent on runtime conditions you couldn't check; state exactly what must be verified and why you couldn't.

Pure speculation with no supporting evidence must NOT be reported at any level. Prioritize findings by real-world impact and exploitability (per the severity rubric), and never present a Needs-Verification item with the same weight as a Confirmed one.

If a category has no findings:

```
+--------------------------------------------------------------+
|  [CATEGORY #] [CATEGORY NAME]                    Score: 10/10 |
+--------------------------------------------------------------+

  No issues found.
```

---

### SECTION 6: Root Cause Patterns

After listing all individual findings, look for clusters -- multiple findings that share the same underlying cause. This section helps the user see systemic issues rather than just a list of symptoms.

```
+--------------------------------------------------------------+
|                    ROOT CAUSE PATTERNS                         |
+--------------------------------------------------------------+

  PATTERN 1: [Name, e.g., "Missing input validation layer"]
  Affects: [SEV-003], [SEV-007], [SEV-012]
  Root cause: [1-2 sentences explaining the systemic issue]
  Systemic fix: [What architectural change would fix all
                 related findings at once]

  ---

  PATTERN 2: [Name]
  ...
```

If no patterns are detected (findings are all independent), say:
"No systemic patterns detected -- findings are independent."

---

### SECTION 7: Technical Debt Scorecard

Pull the weighted debt score from Section 14's analysis. This is the only section that uses the 0-120 weighted scale.

```
+--------------------------------------------------------------+
|                  TECHNICAL DEBT SCORECARD                      |
+-----------------------+-------+--------+----------+-----------+
| Component             | Score | Weight | Weighted | vs. Last  |
+-----------------------+-------+--------+----------+-----------+
| Code Complexity       |  X/10 |   3x   |   XX/30  |    --     |
| Dependency Health     |  X/10 |   2x   |   XX/20  |    --     |
| Code Markers (TODO)   |  X/10 |   1x   |   XX/10  |    --     |
| Dead Code             |  X/10 |   1x   |   XX/10  |    --     |
| Migration Backlog     |  X/10 |   2x   |   XX/20  |    --     |
| Coupling & Cohesion   |  X/10 |   3x   |   XX/30  |    --     |
+-----------------------+-------+--------+----------+-----------+
| TOTAL DEBT SCORE      |       |        |  XX/120  |    --     |
+-----------------------+-------+--------+----------+-----------+

Rating: [Excellent (0-20) / Good (21-40) / Fair (41-60) /
         Concerning (61-80) / Critical (81-120)]
```

---

### SECTION 8: Comparison to Last Audit

If a previous audit report exists (check for `audit-report-*.md` files in the project or on the Desktop), show what changed. If this is the first audit, say so and skip the table.

```
+--------------------------------------------------------------+
|                 COMPARISON TO LAST AUDIT                      |
+--------------------------------------------------------------+
| Last audit: [DATE] | This audit: [DATE]                      |
+------------------------+-----------+-----------+--------------+
| Metric                 | Last      | Now       | Change       |
+------------------------+-----------+-----------+--------------+
| Overall Score          | 87/140    | 94/140    | +7 [UP]      |
| Total Findings         | 22        | 16        | -6 [UP]      |
| Critical Findings      | 3         | 2         | -1 [UP]      |
| Debt Score             | 52/120    | 44/120    | -8 [UP]      |
| Test Coverage (stmts)  | 76.1%     | 78.2%     | +2.1% [UP]   |
+------------------------+-----------+-----------+--------------+

Resolved since last audit: [SEV-xxx], [SEV-xxx], ...
New since last audit: [SEV-xxx], [SEV-xxx], ...
Still open: [SEV-xxx], [SEV-xxx], ...
```

---

### SECTION 9: Action Plan

Prioritized list of what to do next. Group into three tiers.

```
+--------------------------------------------------------------+
|                       ACTION PLAN                             |
+--------------------------------------------------------------+

  FIX NOW (before next deploy):
  
    1. [SEV-001] [Title] .................... Effort: Quick
    2. [SEV-004] [Title] .................... Effort: Moderate

  FIX THIS SPRINT:
  
    3. [SEV-002] [Title] .................... Effort: Moderate
    4. [SEV-005] [Title] .................... Effort: Significant
    5. [SEV-008] [Title] .................... Effort: Moderate

  SCHEDULE LATER:
  
    6. [SEV-010] [Title] .................... Effort: Quick
    7. [SEV-012] [Title] .................... Effort: Quick

+--------------------------------------------------------------+
| ESTIMATED TOTAL EFFORT:                                       |
|   Fix Now:        ~1 hour                                     |
|   This Sprint:    ~4-6 hours                                  |
|   Schedule Later: ~1 hour                                     |
+--------------------------------------------------------------+
```

**Tier assignment rules:**
- "Fix Now" = all Critical findings + any High findings that are security-related
- "Fix This Sprint" = remaining High findings + Medium findings with easy fixes
- "Schedule Later" = remaining Medium + all Low findings

After the tiered list, add two cross-cutting lists:

```
  QUICK WINS (safe to fix now, low regression risk):

    [Findings of ANY severity that are fixable in isolation with
     a small blast radius — the "do these regardless" list.]

  NEEDS ARCHITECTURE / DEEPER INVESTIGATION:

    [Findings that cannot be fixed locally — they require design
     changes, data migrations, or runtime verification first.
     For each, name the decision or investigation that unblocks it.]
```

---

### SECTION 10: Quick Audit Checklist

A pass/fail checklist for rapid re-verification. Check or uncheck based on actual audit results.

```
+--------------------------------------------------------------+
|                   QUICK AUDIT CHECKLIST                        |
+--------------------------------------------------------------+

  MUST HAVE (Critical):
    [x] No secrets in code
    [x] .env* files gitignored
    [x] npm run build succeeds
    [ ] npm audit shows no critical vulnerabilities   <-- FAIL
    [x] Error boundaries in place
    [x] HTTPS enforced
    [x] Production env vars configured
    [x] All unit tests pass
    [x] No .only or .skip in committed tests
    [x] Coverage thresholds met

  SHOULD HAVE (High):
    [x] Security headers configured
    [ ] TypeScript strict mode                        <-- FAIL
    [x] All images use next/image
    [x] Form validation (client + server)
    [x] Alt text on images
    [x] Meta tags configured
    [ ] Error tracking configured                     <-- FAIL
    [x] Every service has a test file

  NICE TO HAVE (Medium/Low):
    [x] Structured data / JSON-LD
    [x] Sitemap
    [ ] Analytics integration
    [x] Pre-commit hooks
    [x] CI/CD pipeline
    [ ] Technical debt score < 40
    [x] No TODOs older than 3 months
    [x] Zero circular dependencies

  Result: XX/YY checks passing
```

---

### Report Output Rules

1. **Write the report to a timestamped file** on the Desktop: `~/Desktop/audit-report-[PROJECT_NAME]-[YYYY-MM-DD].md`
2. **Also display the full report** in the conversation so the user can read it immediately
3. **Use ASCII box tables** throughout (the `+---+` style), never markdown pipe tables
4. **All effort estimates must be realistic** -- see the feedback memory about not padding estimates
5. **Finding descriptions must be in plain English** -- explain user-visible impact, not implementation details
6. **Never reference file:line in the user-facing summary sections** (Sections 1-4, 9). File paths are fine in Section 5 (Findings) where specificity is needed.
7. **If a finding references a known exception** documented in CLAUDE.md (e.g., the npm audit uuid findings), note it as "Known / Accepted" and do not count it as an open finding
8. **The audit itself is read-only** — the complete report is produced BEFORE any fix is applied; fixes are separate, subsequent work
9. **Every finding carries Evidence and a Confidence level** — findings without supporting evidence are dropped, not hedged
10. **Also emit a machine-readable findings file** alongside the Markdown report: `~/Desktop/audit-report-[PROJECT_NAME]-[YYYY-MM-DD].json`, an array of findings each with `{ id, severity, category, confidence, title, location, effort }`. This is what makes audit-over-audit comparison (Section 8) automatic instead of eyeballed — the next run diffs against the prior JSON to compute resolved/new/still-open.

---

## 13. Test Coverage Audit

### 13.1 Test Suite Documentation

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Test suite doc exists | Medium | Missing `TEST-SUITE.md` or equivalent in project root |
| Test catalog current | Medium | Test documentation out of sync with actual tests |
| Coverage summary | High | No coverage statistics documented |

**Expected Test Documentation File:**

Projects should maintain a `TEST-SUITE.md` file in the project root that catalogs all tests. This file should be auto-generated or manually updated when tests change.

```
PROJECT_ROOT/
├── TEST-SUITE.md          # Test suite documentation
├── package.json           # With test scripts and coverage config
└── src/
    └── **/*.spec.ts       # Test files
```

**TEST-SUITE.md Template:**
```markdown
# Test Suite Documentation

## Summary
| Metric | Value |
|--------|-------|
| Total Test Suites | N |
| Total Tests | N |
| Statement Coverage | X% |
| Branch Coverage | X% |
| Function Coverage | X% |
| Line Coverage | X% |

## Test Suites

### ModuleName Tests
- `describe block name`
  - ✓ test description 1
  - ✓ test description 2

[... all test suites listed ...]
```

**Audit Commands:**
```bash
# Check if test documentation exists
ls -la TEST-SUITE.md 2>/dev/null || echo "❌ Missing TEST-SUITE.md"

# Generate test list from Jest (verbose output)
npm test -- --verbose 2>&1 | grep -E "✓|✕|PASS|FAIL"

# Count total tests
npm test -- --json 2>/dev/null | jq '.numTotalTests' || \
  npm test 2>&1 | grep -E "Tests:.*passed" | tail -1
```

### 13.2 Test Coverage Thresholds

| Check | Severity | Threshold |
|-------|----------|-----------|
| Statement coverage | High | ≥ 70% (configurable) |
| Branch coverage | High | ≥ 60% (configurable) |
| Function coverage | High | ≥ 70% (configurable) |
| Line coverage | High | ≥ 70% (configurable) |

**Files to Check:**
```
package.json (jest.coverageThreshold)
jest.config.js
jest.config.ts
```

**Expected Coverage Configuration (package.json):**
```json
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "statements": 70,
        "branches": 60,
        "functions": 70,
        "lines": 70
      }
    }
  }
}
```

**Audit Commands:**
```bash
# Run tests with coverage
npm run test:cov || npm test -- --coverage

# Check coverage thresholds are configured
grep -A 10 "coverageThreshold" package.json || \
  grep -A 10 "coverageThreshold" jest.config.*

# Verify coverage passes thresholds
npm run test:cov 2>&1 | grep -E "Coverage|Statements|Branches|Functions|Lines"
```

### 13.3 Test File Coverage

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Service test files | High | Services without corresponding `.spec.ts` |
| Resolver/Controller tests | High | API endpoints without tests |
| Entity tests | Medium | Entities with custom logic untested |
| Utility tests | Medium | Utility functions without tests |

**Expected Test File Pattern:**
```
src/modules/user/
├── services/
│   ├── user.service.ts
│   └── user.service.spec.ts      # ← Required
├── resolvers/
│   ├── user.resolver.ts
│   └── user.resolver.spec.ts     # ← Required
├── entities/
│   ├── user.entity.ts
│   └── user.entity.spec.ts       # ← If has custom methods
```

**Audit Commands:**
```bash
# Find services without tests
for service in $(find src -name "*.service.ts" | grep -v spec); do
  spec="${service%.ts}.spec.ts"
  [ ! -f "$spec" ] && echo "❌ Missing: $spec"
done

# Find resolvers/controllers without tests
for file in $(find src -name "*.resolver.ts" -o -name "*.controller.ts" | grep -v spec); do
  spec="${file%.ts}.spec.ts"
  [ ! -f "$spec" ] && echo "❌ Missing: $spec"
done

# Count test files vs source files
echo "Source files: $(find src -name "*.ts" ! -name "*.spec.ts" ! -name "*.d.ts" | wc -l)"
echo "Test files: $(find src -name "*.spec.ts" | wc -l)"
```

### 13.4 Test Quality Checks

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Skipped tests | High | `.skip` or `xit` in test files |
| Focused tests | Critical | `.only` or `fit` (will skip other tests in CI) |
| Empty tests | High | Test blocks with no assertions |
| Async issues | High | Missing `await` or unhandled promises |

**Audit Commands:**
```bash
# Find skipped tests
grep -rn "\.skip\|xit\|xdescribe" --include="*.spec.ts" src/

# Find focused tests (CRITICAL - breaks CI)
grep -rn "\.only\|fit\|fdescribe" --include="*.spec.ts" src/

# Find tests without assertions
grep -A 5 "it('" --include="*.spec.ts" src/ | grep -B 5 "^--$" | grep -v expect

# Find potential async issues
grep -rn "async.*=>" --include="*.spec.ts" src/ | grep -v await
```

### 13.5 E2E Test Coverage

| Check | Severity | What to Look For |
|-------|----------|------------------|
| E2E tests exist | High | Missing `test/*.e2e-spec.ts` files |
| Auth scenarios | Critical | Missing 401/403 tests |
| Happy path | High | Missing success case tests |
| Error cases | Medium | Missing validation/error tests |

**Expected E2E Test Structure:**
```
test/
├── app.e2e-spec.ts           # Health/basic tests
├── auth.e2e-spec.ts          # Authentication tests
├── user.e2e-spec.ts          # User CRUD tests
├── content.e2e-spec.ts       # Content API tests
└── jest-e2e.json             # E2E Jest config
```

**E2E Test Checklist per Endpoint:**
- [ ] 200/201: Happy path with valid data
- [ ] 401: Unauthenticated request rejected
- [ ] 403: Unauthorized role rejected
- [ ] 400: Invalid input validation
- [ ] 404: Non-existent resource
- [ ] 409: Duplicate/conflict handling

**Audit Commands:**
```bash
# Check E2E tests exist
ls -la test/*.e2e-spec.ts 2>/dev/null || echo "❌ No E2E tests found"

# Count E2E test scenarios
grep -c "it('" test/*.e2e-spec.ts 2>/dev/null || echo "0 E2E tests"

# Check for auth tests
grep -l "401\|Unauthorized\|403\|Forbidden" test/*.e2e-spec.ts || \
  echo "⚠️ No auth tests found"

# Run E2E tests
npm run test:e2e
```

### 13.6 Test Suite Health

| Check | Severity | What to Look For |
|-------|----------|------------------|
| All tests pass | Critical | Any failing tests |
| Test execution time | Medium | Tests taking > 5 minutes |
| Flaky tests | High | Tests that pass/fail inconsistently |
| Test isolation | High | Tests that depend on each other |

**Audit Commands:**
```bash
# Run all tests and check for failures
npm test 2>&1 | tail -20

# Check test execution time
time npm test

# Run tests multiple times to detect flakiness (optional)
for i in 1 2 3; do npm test || echo "Failed on run $i"; done
```

### 13.7 Test Coverage Checklist

**Test Documentation:**
- [ ] `TEST-SUITE.md` exists in project root
- [ ] Test suite documentation is current
- [ ] Coverage statistics documented

**Coverage Thresholds:**
- [ ] Coverage thresholds configured in Jest
- [ ] Statements ≥ 70%
- [ ] Branches ≥ 60%
- [ ] Functions ≥ 70%
- [ ] Lines ≥ 70%

**Test File Coverage:**
- [ ] Every service has `.spec.ts` file
- [ ] Every resolver/controller has `.spec.ts` file
- [ ] Entities with custom methods have tests
- [ ] Utility functions have tests

**Test Quality:**
- [ ] No `.skip` or `.only` in committed tests
- [ ] All tests have assertions
- [ ] Async tests properly awaited
- [ ] No console.log in tests

**E2E Tests:**
- [ ] E2E tests exist for main endpoints
- [ ] Auth scenarios (401/403) tested
- [ ] Happy paths tested
- [ ] Validation errors tested

**Test Suite Health:**
- [ ] All tests pass (`npm test`)
- [ ] Coverage meets thresholds (`npm run test:cov`)
- [ ] E2E tests pass (`npm run test:e2e`)
- [ ] Tests run in reasonable time (< 5 min)

---

## 14. Technical Debt Audit

> **Health Tracking:** This section quantifies accumulated technical debt so it can be prioritized alongside feature work. Run periodically to track trends over time.

### 14.1 Code Complexity

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Cyclomatic complexity | High | Functions with complexity > 10 |
| Cognitive complexity | High | Deeply nested logic (3+ levels) |
| God files | High | Files > 500 lines |
| God functions | High | Functions > 50 lines |
| Parameter count | Medium | Functions with > 4 parameters |

**Complexity Thresholds:**

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| Cyclomatic complexity per function | ≤ 10 | 11-20 | > 20 |
| Nesting depth | ≤ 3 | 4-5 | > 5 |
| File length (lines) | ≤ 300 | 301-500 | > 500 |
| Function length (lines) | ≤ 30 | 31-50 | > 50 |
| Parameters per function | ≤ 3 | 4-5 | > 5 |

**Audit Commands:**
```bash
# Find large files (> 300 lines)
find src -name "*.ts" -o -name "*.tsx" | xargs wc -l | sort -rn | head -20

# Find deeply nested code (4+ levels of indentation)
grep -rn "^            " --include="*.ts" --include="*.tsx" src/ | wc -l

# Find long functions (rough heuristic: count lines between function declarations)
grep -rn "async \|function \|=> {" --include="*.ts" --include="*.service.ts" src/ | head -20

# Find functions with many parameters
grep -rE "\([^)]{100,}\)" --include="*.ts" src/ | head -10

# Optional: use ESLint complexity rule
npx eslint src/ --rule '{"complexity": ["warn", 10]}' --no-eslintrc --parser @typescript-eslint/parser 2>/dev/null | head -30
```

### 14.2 Dependency Health

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Major versions behind | High | Dependencies 2+ major versions behind latest |
| Deprecated packages | High | Using packages marked as deprecated |
| Unmaintained packages | Medium | Dependencies with no updates in 12+ months |
| Upgrade burden | Medium | Total count of outdated packages |
| Pinned versions | Low | Missing caret/tilde in version ranges |

**Dependency Age Thresholds:**

| Status | Definition | Severity |
|--------|-----------|----------|
| Current | Latest version | None |
| Minor behind | Same major, minor behind | Low |
| 1 major behind | One major version behind | Medium |
| 2+ major behind | Two or more major versions behind | High |
| Deprecated | Package officially deprecated | High |
| Unmaintained | No release in 12+ months | Medium |

**Audit Commands:**
```bash
# List all outdated packages with current/wanted/latest
npm outdated 2>/dev/null

# Count outdated packages
npm outdated 2>/dev/null | tail -n +2 | wc -l

# Check for deprecated packages
npm ls 2>&1 | grep -i "deprecated" | head -10

# Check for known vulnerabilities
npm audit --json 2>/dev/null | jq '.metadata.vulnerabilities' 2>/dev/null || npm audit 2>/dev/null | tail -5

# Check age of lock file
ls -la package-lock.json
```

**Debt Scoring Formula:**
```
Dependency Debt Score = (outdated_minor × 1) + (outdated_major_1 × 3) + (outdated_major_2plus × 5) + (deprecated × 8)
```

### 14.3 TODO / FIXME / HACK Tracking

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Total count | Medium | More than 10 TODOs across codebase |
| Age of TODOs | High | TODOs older than 3 months (check git blame) |
| Severity markers | High | FIXME or HACK comments (indicate known broken code) |
| Untracked debt | Medium | TODOs without issue references |

**Expected Pattern:**
```typescript
// ✅ GOOD: Tracked TODO with issue reference
// TODO(#142): Migrate to new auth system before Q2 launch

// ✅ GOOD: Tracked FIXME with context
// FIXME(#87): Race condition when concurrent writes hit this endpoint

// ❌ BAD: Untracked, undated, no context
// TODO: Fix this later
// HACK: Temporary workaround
// FIXME: This is broken
```

**Audit Commands:**
```bash
# Count all debt markers
echo "TODOs: $(grep -r 'TODO' --include='*.ts' --include='*.tsx' src/ | wc -l)"
echo "FIXMEs: $(grep -r 'FIXME' --include='*.ts' --include='*.tsx' src/ | wc -l)"
echo "HACKs: $(grep -r 'HACK' --include='*.ts' --include='*.tsx' src/ | wc -l)"
echo "XXXs: $(grep -r 'XXX' --include='*.ts' --include='*.tsx' src/ | wc -l)"

# List all with file locations
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.ts" --include="*.tsx" src/

# Check age of TODOs via git blame (oldest first)
grep -rn "TODO\|FIXME" --include="*.ts" src/ | while IFS=: read -r file line rest; do
  date=$(git blame -L "$line,$line" "$file" 2>/dev/null | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}')
  [ -n "$date" ] && echo "$date  $file:$line  $rest"
done | sort | head -20

# Check for TODOs without issue references
grep -rn "TODO\|FIXME" --include="*.ts" src/ | grep -v "#[0-9]" | head -20
```

### 14.4 Dead Code & Unused Exports

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Unused exports | Medium | Exported functions/types never imported elsewhere |
| Unreachable code | High | Code after return/throw statements |
| Commented-out code | Medium | Large blocks of commented code (> 5 lines) |
| Unused files | Medium | Files not imported by any other file |
| Feature flags never removed | Medium | Old feature flags still in code |

**Audit Commands:**
```bash
# Find potentially unused exports (check if exported names are imported elsewhere)
for file in $(find src -name "*.ts" -not -name "*.spec.ts" -not -name "*.d.ts"); do
  exports=$(grep -oE "export (function|class|const|type|interface|enum) \w+" "$file" | awk '{print $NF}')
  for exp in $exports; do
    count=$(grep -r "$exp" --include="*.ts" --include="*.tsx" src/ | grep -v "$file" | wc -l)
    [ "$count" -eq 0 ] && echo "Possibly unused: $exp in $file"
  done
done 2>/dev/null | head -30

# Find large commented-out code blocks
grep -rn "^\/\/" --include="*.ts" --include="*.tsx" src/ | awk -F: '{print $1}' | uniq -c | sort -rn | awk '$1 > 10 {print "⚠️ " $1 " commented lines in " $2}' | head -10

# Find files not imported anywhere
for file in $(find src -name "*.ts" -not -name "*.spec.ts" -not -name "*.d.ts" -not -name "index.ts" -not -name "*.module.ts"); do
  basename=$(basename "$file" .ts)
  count=$(grep -r "$basename" --include="*.ts" --include="*.tsx" src/ | grep -v "$file" | wc -l)
  [ "$count" -eq 0 ] && echo "Possibly orphaned: $file"
done 2>/dev/null | head -20
```

### 14.5 Migration & Deprecation Backlog

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Deprecated API usage | High | Using deprecated framework/library APIs |
| Legacy patterns | Medium | Old patterns that should be migrated (e.g., Class Components → Hooks) |
| Compatibility shims | Medium | Workarounds for old behavior still in place |
| Version-locked code | High | Code that only works with a specific old version |

**Common Migration Debt in Next.js/NestJS:**

| Pattern | Debt | Migration Target |
|---------|------|-----------------|
| `getServerSideProps` | Medium | App Router Server Components |
| `pages/` directory | High | `app/` directory |
| Class components | Medium | Function components + hooks |
| `@Controller` REST | Low | `@Resolver` GraphQL (if migrating) |
| `moment.js` | Medium | `date-fns` or native `Intl` |
| `lodash` (full import) | Low | Tree-shakeable imports or native |
| `any` type annotations | Medium | Proper TypeScript types |
| `require()` | Low | ES module `import` |

**Audit Commands:**
```bash
# Check for deprecated Next.js patterns
grep -r "getServerSideProps\|getStaticProps\|getInitialProps" --include="*.ts" --include="*.tsx" src/ | head -10

# Check for pages/ directory (legacy routing)
ls -d pages/ 2>/dev/null && echo "⚠️ Legacy pages/ directory exists"

# Check for deprecated library usage
grep -r "require(" --include="*.ts" src/ | grep -v node_modules | head -10

# Check for moment.js (commonly replaced)
grep -r "from 'moment'\|require('moment')" --include="*.ts" --include="*.tsx" src/ | head -5

# Check for full lodash imports
grep -r "from 'lodash'" --include="*.ts" --include="*.tsx" src/ | head -5

# Count all `any` types as debt items
echo "any types: $(grep -r ': any' --include='*.ts' --include='*.tsx' src/ | grep -v node_modules | wc -l)"
```

### 14.6 Coupling & Cohesion

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Circular dependencies | Critical | Modules importing each other |
| High fan-out | High | A single file importing 10+ other files |
| Cross-module direct access | High | Bypassing module APIs to access internals |
| Shared mutable state | High | Global variables modified by multiple modules |

**Audit Commands:**
```bash
# Check for circular dependencies
npx madge --circular src/ 2>/dev/null || echo "Install madge: npm i -g madge"

# Find files with high fan-out (many imports)
for file in $(find src -name "*.ts" -not -name "*.spec.ts"); do
  count=$(grep -c "^import" "$file" 2>/dev/null)
  [ "$count" -gt 10 ] && echo "⚠️ $count imports: $file"
done | sort -t: -k1 -rn | head -10

# Find cross-module internal imports (reaching into another module's internals)
grep -rn "from '.*modules/[^']*/" --include="*.ts" src/ | grep -v "index\|module" | head -20
```

### 14.7 Technical Debt Scorecard

Generate a summary scorecard after running all checks:

```markdown
# Technical Debt Scorecard

**Date:** [DATE]
**Project:** [PROJECT_NAME]

## Debt Inventory

| Category | Count | Severity | Trend |
|----------|-------|----------|-------|
| Complex functions (complexity > 10) | N | High | ↑↓→ |
| Large files (> 300 lines) | N | Medium | ↑↓→ |
| Outdated dependencies | N | Medium | ↑↓→ |
| Deprecated dependencies | N | High | ↑↓→ |
| TODO/FIXME markers | N | Medium | ↑↓→ |
| Stale TODOs (> 3 months) | N | High | ↑↓→ |
| `any` type annotations | N | Medium | ↑↓→ |
| Dead/unused exports | N | Low | ↑↓→ |
| Circular dependencies | N | Critical | ↑↓→ |
| Legacy patterns | N | Medium | ↑↓→ |

## Debt Score

| Component | Score | Weight | Weighted |
|-----------|-------|--------|----------|
| Complexity | 0-10 | 3x | 0-30 |
| Dependencies | 0-10 | 2x | 0-20 |
| Code markers | 0-10 | 1x | 0-10 |
| Dead code | 0-10 | 1x | 0-10 |
| Migration backlog | 0-10 | 2x | 0-20 |
| Coupling | 0-10 | 3x | 0-30 |
| **Total** | | | **0-120** |

**Rating:**
- 0-20: Excellent — minimal debt
- 21-40: Good — manageable debt
- 41-60: Fair — schedule debt sprints
- 61-80: Concerning — dedicate 20% of sprint capacity
- 81-120: Critical — debt is slowing development, prioritize immediately

## Top 5 Debt Items to Address

1. [Highest impact item]
2. [Second highest]
3. [Third]
4. [Fourth]
5. [Fifth]

## Recommended Actions

- [ ] [Action 1 with estimated effort]
- [ ] [Action 2 with estimated effort]
- [ ] [Action 3 with estimated effort]
```

### 14.8 Technical Debt Checklist

**Code Complexity:**
- [ ] No functions with cyclomatic complexity > 10
- [ ] No files > 500 lines
- [ ] No functions > 50 lines
- [ ] No deeply nested code (> 3 levels)
- [ ] No functions with > 5 parameters

**Dependency Health:**
- [ ] No packages 2+ major versions behind
- [ ] No deprecated packages in use
- [ ] `npm audit` shows no high/critical vulnerabilities
- [ ] Lock file is up to date

**Code Markers:**
- [ ] Fewer than 10 TODO/FIXME markers total
- [ ] No TODOs older than 3 months
- [ ] All TODOs reference an issue/ticket
- [ ] No HACK or XXX markers

**Dead Code:**
- [ ] No large blocks of commented-out code
- [ ] No unused exports
- [ ] No orphaned files

**Migration Backlog:**
- [ ] No deprecated framework API usage
- [ ] No legacy patterns (pages/ dir, class components)
- [ ] No full library imports where tree-shaking is available
- [ ] `any` type count trending downward

**Coupling:**
- [ ] No circular dependencies
- [ ] No files with 10+ imports
- [ ] No cross-module internal access

---

## 15. Payment Security & PCI Compliance Audit

> **Go-live priority:** Required before any surface that touches money goes live.
> The first task is SCOPE DETERMINATION — everything else follows from whether raw
> card data (PAN/CVV/expiry) ever transits or rests on systems you operate.

### 15.1 Cardholder Data Flow & Scope

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Card data flow map | Critical | No documented understanding of where PAN/CVV travels |
| Raw PAN transits backend | Critical | Card fields in DTOs/payloads proxied through your API |
| Card data at rest | Critical | PAN/CVV/expiry in any DB column, JSONB blob, or file |
| Card data in logs | Critical | Payment payloads logged (even at debug level) |
| Card data in analytics/session replay | High | Payment form fields captured by tracking/replay tooling |
| Card data in error trackers | High | Request bodies with card fields sent to Sentry/similar |

**SAQ level cheat sheet (determines compliance burden):**

| Integration Pattern | SAQ Level | Burden |
|---------------------|-----------|--------|
| Full redirect to provider-hosted page | SAQ A | Minimal |
| Provider iframe/JS fields (card data never touches your DOM values or server) | SAQ A | Minimal |
| Your page renders card fields, provider JS tokenizes client-side | SAQ A-EP | Moderate |
| Card data POSTs to/through YOUR server (even just proxied) | SAQ D | Heavy — avoid if possible |

**Audit Commands:**
```bash
# Find card-data fields in source (DTOs, interfaces, payload builders)
grep -rniE "cardNumber|card_number|\bpan\b|cvv|cvc|securityCode|expiry|cardholder" src/ packages/ --include="*.ts" --include="*.tsx" | grep -v spec | grep -v node_modules

# Find card data in logging calls (same file: logger + card term)
grep -rlniE "cvv|cardNumber|card_number" src/ --include="*.ts" | xargs grep -ln "logger\.\|console\." 2>/dev/null

# Check payment request bodies aren't captured by error tracking
grep -rn "beforeSend\|sendDefaultPii\|requestBodies" src/ --include="*.ts" | head
```

### 15.2 Payment Operation Integrity

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Money-step retries | Critical | Auto-retry on charge/authorize/capture (double-charge risk) |
| Idempotency keys | Critical | Payment mutations without idempotency protection |
| Amount server-derived | Critical | Client-supplied price/amount trusted by the charge path |
| Webhook signature verification | High | Payment webhooks accepted without signature check |
| Payment audit trail | High | Money operations not written to an audit log |
| TLS end-to-end | Critical | Any payment call over http:// |
| Card data in URLs | Critical | Payment params in GET query strings (logged by proxies) |

**Checklist:**
- [ ] Charges/authorizations have `retries: 0` (timeout ≠ retry; whole-attempt replay uses the SAME idempotency key)
- [ ] Amounts recomputed server-side from rates/config, never trusted from the client
- [ ] Provider webhooks verify signatures before processing
- [ ] Refund/cancel paths require elevated authorization
- [ ] Payment provider credentials encrypted at rest, never in env-committed files

### 15.3 PCI Operational Requirements (go-live gate)

- [ ] SAQ level determined and documented (with the reasoning)
- [ ] Quarterly ASV scan scheduled if SAQ A-EP or D
- [ ] Access to payment credentials restricted and auditable
- [ ] Incident response contact/plan for suspected card-data exposure
- [ ] Provider agreements confirm THEY hold PCI DSS certification (get the AOC)

---

## 16. Concurrency & Multi-Instance Safety Audit

> **Go-live priority:** Code that works with one server instance and low traffic
> silently corrupts data or drops limits when horizontally scaled or hit
> concurrently. Inventory every piece of in-process state FIRST.

### 16.1 In-Memory State Inventory

| Check | Severity | What to Look For |
|-------|----------|------------------|
| In-memory rate limiters | High | Maps/counters used as limiters — reset on restart, per-instance under scale |
| In-memory caches with authority | High | Cache treated as source of truth (not just perf) |
| In-memory locks/semaphores | High | Static semaphores that only bound ONE instance |
| Idempotency caches in memory | Critical | Replay protection lost on restart/second instance |
| Module-level mutable state | Medium | Singletons accumulating request data |

**Audit Commands:**
```bash
# Inventory candidate in-memory state (Maps/Sets/counters at class or module level)
grep -rn "new Map(\|new Set(\|private.*: Map<\|static.*Semaphore\|= new.*Cache" src/modules --include="*.ts" | grep -v spec | head -40

# Known-risky patterns: rate limit / attempt tracking in memory
grep -rniE "attempts?(Map|Cache|Count)|limiter|bucket|semaphore" src/modules --include="*.ts" | grep -v spec | head -20
```

**For each hit, classify:** (a) safe — pure perf cache with TTL, wrong answers harmless;
(b) degraded — limits become per-instance (N instances = N× the limit); document as
accepted or move to Redis; (c) broken — correctness depends on it (idempotency, dedupe,
distributed locks); MUST move to Redis/DB before scaling past one instance.

### 16.2 Database Race Conditions

| Check | Severity | What to Look For |
|-------|----------|------------------|
| find-then-save upserts | Critical | `findOne` + `save` where concurrent requests hit the same row (use atomic `ON CONFLICT DO UPDATE`) |
| check-then-insert uniqueness | High | Existence check + insert without a DB unique constraint backing it |
| Read-modify-write counters | High | `entity.count++; save()` instead of atomic `SET count = count + 1` |
| Multi-step writes without transaction | High | Related writes that can partially fail |
| Missing unique constraints | High | Uniqueness enforced only in application code |

**Audit Commands:**
```bash
# Find upsert-shaped find+save (manual review targets)
grep -rn -B2 -A8 "findOne" src/modules --include="*.service.ts" | grep -l "save" 2>/dev/null | head

# Find atomic patterns (the GOOD pattern — compare coverage)
grep -rn "ON CONFLICT\|onConflict\|upsert(" src/modules --include="*.ts" | grep -v spec | head -15

# Transactions in use?
grep -rn "transaction(\|queryRunner\|@Transaction" src/modules --include="*.ts" | grep -v spec | wc -l
```

### 16.3 Horizontal Scaling Readiness

- [ ] Background jobs use a shared queue (BullMQ/Redis), not in-process cron alone — or cron jobs are single-instance-safe (advisory locks / job-claim rows)
- [ ] Scheduled tasks won't double-fire with 2 instances (`@Cron` in NestJS fires on EVERY instance)
- [ ] Sessions/auth are stateless (JWT) or in shared storage — no sticky-session dependency
- [ ] File writes go to object storage, not local disk
- [ ] WebSocket/SSE fan-out has a cross-instance adapter (or is documented single-instance)
- [ ] DB pool size × instance count < Postgres max_connections (with headroom for migrations/console)

**Audit Commands:**
```bash
# Cron/interval jobs that would double-fire on 2 instances
grep -rn "@Cron\|@Interval\|setInterval" src/modules --include="*.ts" | grep -v spec | head -20

# DB pool configuration
grep -rn "poolSize\|max:\|connectionLimit" src/database src/config --include="*.ts" | head
```

---

## 17. Load & Capacity Readiness Audit

> **Go-live priority:** Verify the system has been pushed past expected peak
> BEFORE real users do it. An untested capacity number is a guess.

### 17.1 Load Test Harness

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Harness exists | High | No load-test tooling (k6/artillery/autocannon) in repo |
| Critical journeys covered | High | Load tests hit health endpoints only, not the money paths |
| Realistic target defined | High | No documented expected peak (concurrent users, RPS) |
| Recent results | Medium | Harness exists but hasn't run against current code |
| Soak test | Medium | Only burst tests — no sustained-load memory-leak check |

**Journeys that MUST be covered for a booking/CMS platform:** availability lookup,
booking submission (with payment mocked), chatbot ask (AI mocked or budget-capped),
analytics ingest burst, public page render/API, auth login storm.

### 17.2 Backpressure & Limits

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Rate limiting per endpoint class | High | One global limit (or none) — public endpoints need tighter buckets |
| Body size limits | High | Default/unbounded JSON body limit on public POST endpoints |
| Timeout budgets | High | Outbound calls without timeouts (one slow upstream stalls the pool) |
| Queue depth bounds | Medium | Unbounded in-process queues/buffers under burst |
| Expensive-endpoint gating | High | AI/media/export endpoints without concurrency caps or quotas |
| Graceful degradation | Medium | Dependency outage (Redis, AI provider) crashes requests instead of degrading |

**Audit Commands:**
```bash
# Throttler / rate limit configuration
grep -rn "ThrottlerModule\|@Throttle\|rateLimit" src --include="*.ts" | grep -v spec | head -20

# Body size limits
grep -rn "json({ limit\|urlencoded({ limit\|bodyParser" src/main.ts src/app.module.ts 2>/dev/null

# Outbound fetch/axios without visible timeout handling (manual review list)
grep -rn "await fetch(\|axios\." src/modules --include="*.ts" | grep -v spec | grep -viE "timeout|signal|abort" | head -20
```

### 17.3 Hot-Path Efficiency

- [ ] Indexes exist for the highest-volume query patterns (check the top-N queries' WHERE/ORDER BY columns)
- [ ] No N+1 query patterns on list endpoints (look for awaited queries inside loops)
- [ ] High-volume writes are batched or queued, not row-at-a-time per request
- [ ] Caching applied to hot reads that tolerate staleness (with TTL + invalidation story)

```bash
# Awaited queries inside loops (N+1 candidates)
grep -rn -A3 "for (\|for await\|\.map(async" src/modules --include="*.service.ts" | grep -B1 "await.*\(find\|query\|save\)" | head -20
```

---

## 18. Penetration-Test Readiness Audit

> **Go-live priority:** This is the self-assessment BEFORE commissioning an external
> pen test — find the cheap findings yourself so the paid test digs deeper. It does
> NOT replace an external test for a payment-adjacent platform.

### 18.1 API Attack Surface

| Check | Severity | What to Look For |
|-------|----------|------------------|
| GraphQL introspection in prod | Medium | Introspection/playground enabled in production |
| GraphQL depth/complexity limits | High | No query depth or cost limits (nested-query DoS) |
| GraphQL batching abuse | Medium | Unbounded operation batching (brute-force amplifier) |
| Public endpoint inventory | High | No authoritative list of `@Public()`/unauthenticated routes |
| Mass assignment | High | DTOs with `whitelist: false` or raw body passthrough to entities |
| IDOR | Critical | Object IDs accepted without tenant/ownership scoping check |

**Audit Commands:**
```bash
# GraphQL prod hardening
grep -rn "introspection\|playground\|depthLimit\|complexity" src/app.module.ts src/config --include="*.ts"

# Inventory every public route (this list IS the pen-test scope)
grep -rn "@Public()" src/modules --include="*.ts" | grep -v spec

# ValidationPipe strictness
grep -rn "whitelist\|forbidNonWhitelisted" src/main.ts src/app.module.ts
```

### 18.2 Authentication Hardening

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Login rate limiting | Critical | No per-IP/per-account throttle on login/password-reset |
| Account lockout / backoff | High | Unlimited password attempts |
| User enumeration | Medium | Login/reset responses differ for existing vs unknown emails |
| Token lifetime | High | Access tokens valid > 1 hour without rotation story |
| Password reset flow | High | Predictable/long-lived/reusable reset tokens |
| Cookie flags | High | Session cookies missing httpOnly/secure/sameSite |
| Invitation/signup abuse | Medium | Unthrottled invite or signup endpoints (spam vector) |

### 18.3 Input & Outbound Hardening

| Check | Severity | What to Look For |
|-------|----------|------------------|
| SSRF | Critical | Server fetches user-influenced URLs (crawlers, webhooks, imports) without private-IP/localhost blocklist |
| File upload validation | High | Uploads without type allowlist, size cap, and content-type verification |
| Raw SQL interpolation | Critical | String-built SQL with user input (registry/bind-param patterns only) |
| Path traversal | High | User input in file paths without normalization |
| XXE/zip-slip | Medium | XML parsing or archive extraction of user uploads |
| Stack traces to users | Medium | Prod error responses leaking internals |
| CORS | High | Wildcard or reflected-origin CORS on credentialed endpoints |

**Audit Commands:**
```bash
# SSRF candidates: fetch/axios where the URL isn't a literal or env
grep -rn "fetch(\|axios.get(\|axios.post(" src/modules --include="*.ts" | grep -v spec | grep -vE "fetch\('https?://|env\.|process\.env" | head -20

# Raw SQL building
grep -rn "query(\`\|query(\"\|query('" src/modules --include="*.ts" | grep -v spec | grep "\${" | head -20

# Upload constraints
grep -rn "FileInterceptor\|fileFilter\|limits:" src/modules/media --include="*.ts" | head

# CORS config
grep -rn "enableCors\|cors(" src/main.ts src/config --include="*.ts"
```

### 18.4 Pen-Test Readiness Checklist (go-live gate)

- [ ] Public/unauthenticated endpoint inventory documented (= pen-test scope)
- [ ] All 18.1–18.3 self-checks run, findings fixed or accepted in writing
- [ ] Staging environment available for the external tester (prod-like, fake data)
- [ ] External pen test scheduled BEFORE real customer data goes live
- [ ] Retest of pen-test findings budgeted (findings without retest = findings not fixed)
- [ ] Sentry/alerting verified to fire on auth anomalies (so the test itself validates monitoring)

---

## 19. Client-Side Race Conditions & State Integrity Audit

> **Real-user conditions:** Section 16 covers server-side/multi-instance concurrency.
> This section covers the client and request layer: repeated clicks, retries, refreshes,
> slow networks, and multiple tabs. These bugs never appear in a demo and always appear
> in production. Audit by tracing each mutation from click to server and asking "what
> happens if this fires twice, 100ms apart?"

### 19.1 Duplicate Submission & Idempotency

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Double-click submits | High | Submit buttons/actions not disabled while their request is in flight |
| Refresh/back resubmission | High | Form POST results not redirected (no post-redirect-get); replayable submissions |
| Retry duplicates | Critical | Client or fetch-layer retry re-firing a non-idempotent mutation — duplicate bookings, payments, messages, records, or jobs |
| Missing idempotency keys | Critical | Create-style mutations with real-world side effects and no idempotency protection end-to-end |
| Concurrent same-action requests | High | Two overlapping requests for the same logical action both succeeding (no in-flight dedupe or server-side conflict handling) |

If double-fire produces two rows, severity is High minimum — Critical when it's money, messages, or anything sent to an external party.

### 19.2 Stale State & Lost Updates

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Out-of-order async responses | High | Search/filter/autocomplete results applied without checking they belong to the LATEST request (no AbortController or request-id guard) |
| Optimistic update failures | High | Optimistic UI with no rollback path when the server rejects — UI and DB permanently disagree |
| Lost updates | High | Two editors save the same record; last-write-wins silently discards the first (no version/updatedAt conflict check) |
| Cache invalidation | Medium | Mutations that don't invalidate/refetch the queries displaying the same data — stale lists after create/edit/delete |
| Client/server/persisted divergence | Medium | State duplicated across client store, server, and storage with no single source of truth |

### 19.3 Cleanup & Leaks

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Effect cleanup | High | useEffect subscribing, listening, or polling without a cleanup return |
| Timers | Medium | setInterval/setTimeout not cleared on unmount |
| Event listeners | Medium | window/document listeners added without matching removal |
| In-flight requests | Medium | Fetches not aborted on unmount/step-change (setState-after-unmount, wasted work, stale writes) |
| Subscriptions | High | WebSocket/observable subscriptions surviving component death — double-fire on re-mount |

**Audit Commands:**
```bash
# Effects with subscriptions/timers — verify each returns a cleanup (manual review list)
grep -rn -A10 "useEffect" admin packages --include="*.tsx" 2>/dev/null | grep "addEventListener\|setInterval\|subscribe" | head -30

# AbortController adoption on client fetches
grep -rn "AbortController\|signal:" admin packages --include="*.ts" --include="*.tsx" 2>/dev/null | head -10

# Submit-in-progress guards (compare against the number of submit handlers)
grep -rn "isSubmitting\|isPending\|disabled={" admin packages --include="*.tsx" 2>/dev/null | grep -i "loading\|submitting\|pending" | head -10
```

### 19.4 Multi-Tab, Multi-Device & Poor Networks

- [ ] Auth state changes (logout, token refresh) propagate across tabs — or stale tabs fail loudly rather than half-working
- [ ] Concurrent edits from two tabs/devices don't silently lose one side's work
- [ ] Slow-network behavior verified (throttle it): every in-flight action shows progress and cannot be double-fired
- [ ] Actions are unavailable while in progress: buttons/menu items disabled during their own operation
- [ ] A request that succeeds on the server but whose response is lost (network drop) doesn't duplicate on the user's retry

---

## 20. Visual & Interaction Consistency Audit

> **Precision is the standard here.** Name the exact token, measurement, or state that
> diverges. "Looks inconsistent" is not a finding; "the settings card uses 12px radius
> where every sibling surface uses 8px" is.

### 20.1 Design Token & Component Discipline

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Hardcoded values bypassing tokens | Medium | Raw hex colors, px spacing, inline shadows where a token/scale exists |
| Divergent implementations | Medium | Components that look identical but are separate implementations drifting apart |
| Same look, different behavior | High | Identical-looking controls behaving differently across screens (one saves on blur, another on submit) |
| Spacing/typography/radius/shadow drift | Low | Inconsistent scale usage across sibling surfaces; mixed type sizes for the same role |
| Icon sizing & alignment | Low | Mixed icon sizes in the same context; unaligned baselines; decorative icons that vary meaninglessly |

**Audit Commands:**
```bash
# Raw hex colors outside token/theme definitions
grep -rn "#[0-9a-fA-F]\{6\}" admin/src --include="*.tsx" 2>/dev/null | grep -viE "token|theme|test" | head -20

# Inline spacing values where a scale exists
grep -rn "style={{" admin/src --include="*.tsx" 2>/dev/null | grep -E "padding|margin" | head -20
```

### 20.2 Interaction State Matrix

For each interactive component family (buttons, inputs, menu items, cards, rows, tabs), verify ALL applicable states exist and are visually consistent across the app:

```
hover | focus-visible | active | selected | disabled | loading | success | warning | destructive | error
```

Missing or one-off states are Medium findings — High when the gap hides an affordance (e.g. no disabled style, so users click dead buttons; no loading state, so users double-fire — cross-reference Section 19.1).

### 20.3 Layout Robustness

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Layout shift | Medium | Content jumping as data/images load (no reserved space or skeleton) |
| Clipping & overflow | Medium | Fixed heights truncating content; unintended horizontal scrollbars |
| Truncation | Medium | Text cut without ellipsis/tooltip; ellipsis hiding the distinguishing part of the value |
| Wrapping | Low | Labels/buttons wrapping mid-word or breaking layout at plausible lengths |
| Empty states | Medium | Inconsistent or missing zero-data designs across similar screens |

### 20.4 Copy Consistency

- [ ] Terminology: one name per concept everywhere (not "property"/"hotel"/"site" for the same thing)
- [ ] Capitalization: one convention per element type (buttons, titles, labels) — Title Case vs Sentence case, not both
- [ ] Action labels: same action = same verb everywhere (Save vs Update vs Apply)
- [ ] Date & number formats: same format for the same data type everywhere; consistent locale handling
- [ ] Punctuation: consistent periods/no-periods in helper text, tooltips, and empty states

---

## 21. Responsive & Edge-Case Content Audit

> Every layout works with ideal content at the developer's viewport. This section
> audits everything else. Don't grep for these — pick the 5 highest-traffic screens
> and reason through each with hostile content and hostile viewports.

### 21.1 Viewport Sweep

Check key screens at: 320px (small phone), 375px, 768px (tablet), 1280px, 1440px, and ultra-wide (1920px+). Also 200% browser zoom and OS large-text settings (overlaps 7.7 — cross-reference rather than duplicate findings).

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Narrow-screen breakage | High | Horizontal scroll, overlapping controls, or unreachable actions at 320–375px |
| Breakpoint gaps | Medium | Layouts fine AT the defined breakpoints but broken between them |
| Ultra-wide stretch | Low | Text lines and grids stretching unbounded (no max-width container) |
| Hover-only affordances | Medium | Actions revealed only on hover with no touch/keyboard equivalent |

### 21.2 Content Stress Test

| Check | Severity | What to Look For |
|-------|----------|------------------|
| Long values | Medium | 60-char names, long emails, unbroken URLs — overflow, broken cards, pushed-out actions |
| Expanded/translated text | Medium | Strings 30–50% longer than English (de/fr/pt) breaking buttons, labels, navs |
| Empty values | Medium | Missing optional fields rendering "undefined"/"null", dangling labels, or collapsed layouts |
| Very large datasets | High | 1,000+ item lists without pagination/virtualization — browser lockup |
| Zero results | Medium | Empty search/filter results with no designed state |
| Malformed content | High | User/CMS/API content of unexpected shape crashing the render instead of degrading |

**Stress inputs per screen:** the longest plausible value in every field, every optional field absent, 10× the expected data volume, zero data, and one deliberately malformed record.

---

## 22. Codebase-Specific Invariants & Known-Bug Regression Checks

> **This section is a LIVING checklist and is expected to grow as the codebase does.**
> The generic categories (1–21) catch general classes of problem. This section catches
> the specific mistakes THIS codebase has already made once — the highest-signal checks
> available, because each maps to a real past incident. A regression here is not
> theoretical; it has shipped before.

### 22.1 Load the project's own invariants FIRST

The single richest source is the project's `CLAUDE.md` "Invariants (Lessons From Past Bugs)"
section (in this repo: `/Users/gautamlulla/aphalo/halo/CLAUDE.md`). It is a curated, continuously
updated log of real bugs and the rules that prevent their recurrence.

**Procedure:**
1. Read the target project's `CLAUDE.md` (and any nested module `CLAUDE.md`) in full.
2. Treat **every** invariant as a named regression check. For each, verify the current code still
   honors it, and if not, file a finding referencing the invariant by name.
3. Because `CLAUDE.md` is updated whenever a bug is fixed, this check auto-expands over time — you
   do NOT need to keep this section's static list exhaustive. `CLAUDE.md` is the source of truth;
   the list in 22.2 is a fast-start subset of the highest-severity ones.

### 22.2 High-severity starter checks (verify each against current code)

These are the load-bearing ones — confirm they still hold, then continue through the full
`CLAUDE.md` invariant list. Severity shown is the impact if the invariant is violated.

| # | Invariant / past bug | Severity | How to check |
|---|----------------------|----------|--------------|
| 1 | **Cross-tenant authz**: id-only resolvers that skip org/tenant scoping (the CONFIRMED hole; hard gate before a 2nd tenant) | Critical | Trace each `findOne({ id })` on tenant-scoped data; confirm an org/property filter or ownership check. Verify live if servers up (Methodology step 6). |
| 2 | **Money steps never auto-retry**: charge/authorize/reservation-create must be `retries: 0` (double-charge risk) | Critical | Inspect the reservation/payment resilience policies; confirm money steps pass the zero-retry policy while only safe reads retry. |
| 3 | **Booking idempotency**: whole-attempt timeout retry REUSES the same key+payload; partial retry mints a new key with only failed lodges | Critical | Trace the idempotency-key lifecycle from UI mint through `IdempotencyCache` replay. |
| 4 | **Reservation/page secrets never surface**: no plaintext secret in any GraphQL/REST response, log line, or audit payload — only derived `credentialConfigured`/`passwordProtected` booleans | Critical | Grep resolvers/DTOs for secret fields; confirm only boolean projections are exposed. |
| 5 | **Analytics DB split**: no single SQL statement joins an analytics table (`site_events`, `analytics_sessions_v2`, `page_rollups_daily`) to an app table; those repos use the named `analytics` connection | High | Grep for queries mixing the three analytics tables with app tables; confirm cross-side joins happen in JS, not SQL. |
| 6 | **AiProviderService wiring**: provided only from `AiCoreModule`, never re-registered as a bare local provider; callers pass `orgId` explicitly; `isSystemAdmin` only from the admin panel | High | Grep for stray `AiProviderService` providers; confirm background/public callers pass `false` and a real `orgId`. |
| 7 | **GraphQL resolver decorator placement**: never insert a method between an existing method's decorator and its body (the f9ef8769 schema-corruption incident) | High | On any touched resolver, confirm each `@Query`/`@Mutation` sits directly above its intended method; run the module's e2e schema-regression test. |
| 8 | **@Cron double-fire**: every real cron job runs inside `DistributedLockService.withLock`; new `@Cron` jobs must too | High | Grep `@Cron`; confirm each is wrapped; flag any bare scheduled job. |
| 9 | **In-memory limiters are single-instance**: MCP mint limiter, AI semaphore, booking `IdempotencyCache` break across instances — must be documented single-instance or moved to Redis before scale-out | High | Confirm these are still single-instance and that no silent horizontal scale-out has been enabled. |
| 10 | **Deploy SITE_URL**: persisted from the STABLE production alias, never the per-deploy `status.url`; Vercel env vars upserted every deploy; `triggerDeployment` fires on every deploy | High | Trace the deploy verify path; confirm the stable-base resolution order and that env sync + trigger are unconditional. |
| 11 | **Ingest CORS**: preflight allows `Content-Encoding` (gzipped batches) and ACAO reflects origin, never `*` for credentialed beacons | High | Inspect `ingest-cors.ts`; confirm both. |
| 12 | **MCP endpoint is open/keyless BY DESIGN** but must never wire tools to `AiProviderService` or add a payment/inline-booking tool | High | Grep the MCP tools dir; confirm no AI-provider imports and share-link-only handoff. |
| 13 | **No `npm audit fix --force`; `ws` override ≥ 8.21.0**: known accepted findings must not be "fixed" by a forced downgrade | Medium | Confirm the `ws` override and that accepted @apollo/bullmq→uuid findings aren't counted as open (see Report Output Rule 7). |

### 22.3 Keep this section current

When the target project's `CLAUDE.md` gains a new invariant tied to a real incident of Critical/High
severity, add a matching row to 22.2 (or rely on 22.1 to pick it up automatically). When an
invariant is retired in `CLAUDE.md`, remove its row here. This section should track the project's
bug history, not drift from it — same discipline the project applies to `CLAUDE.md` itself.

---

## Version History

**v2.3** (July 2026)
- Live Verification pass (read-only) added to methodology: security findings (IDOR/cross-tenant/authz) and accessibility (WCAG 2.2 AA via axe/Lighthouse + keyboard/screen-reader walk) are now verified against the running dev servers, not just read from source — only safe read actions and test data, never writes/mutations
- Resource Constraints rewritten: the stale "no parallel agents" ban replaced with the real rule for a 32GB machine — up to ~4 parallel read-only analysis agents (6 if servers down), but never more than ONE heavy process (test/build/live-verification) at a time
- Severity is now derived from a rubric (exploitability × blast radius × data sensitivity), not eyeballed — makes ratings reproducible across runs
- Confidence tightened: `Confirmed` now has a hard bar (reproduced live, exhaustively traced, or self-evident) — no more optimistic "Confirmed" on unexecuted findings
- Adversarial second pass: every High/Critical finding must survive an explicit refutation attempt before shipping (kills confident-but-wrong findings)
- New Section 22 Codebase-Specific Invariants & Known-Bug Regression Checks — a LIVING checklist that loads the target project's CLAUDE.md invariants and treats each past bug as a named regression check, plus a high-severity starter set; auto-expands as CLAUDE.md grows
- New report Section 2B Scope & Coverage Disclosure: states which flows were traced, whether live verification ran, and what was static-only — so confidence ratings are backed by visible coverage
- Grep scope widened beyond `src/` to `admin/` and `packages/` for frontend/rendered-output checks
- Machine-readable JSON findings file emitted alongside the Markdown report for automatic audit-over-audit diffing
- Health Scorecard expands to 22 rows (overall /220)

**v2.2** (July 2026)
- New "Audit Methodology" section applying to every category: trace user flows end-to-end (not file-by-file review), adversarial posture for security / systematic for accessibility / precise for UI, evidence-over-speculation standard, challenge happy-path assumptions, and an explicit read-only rule (no code changes until the report is complete)
- Three new categories (21 total): Section 19 Client-Side Race Conditions & State Integrity (duplicate submissions and idempotency, out-of-order responses, optimistic-update rollback, lost updates, effect/timer/listener cleanup, multi-tab + poor-network behavior), Section 20 Visual & Interaction Consistency (design-token discipline, full interaction-state matrix, layout robustness, copy/terminology/format consistency), Section 21 Responsive & Edge-Case Content (viewport sweep incl. 200% zoom, content stress tests: long/expanded/empty/huge/zero/malformed)
- Security audit deepened: 2.5 Authorization, Tenancy & Object-Level Access (server-side permission checks, IDOR, cross-tenant scoping, privilege escalation, trust-boundary validation) and 2.6 Injection, Redirects & Client-Side Storage (SQL/command/template/HTML/prompt injection, insecure redirects, token storage, secrets in URLs, webhook signature verification, storage buckets, third-party scope)
- Section 6 renamed Error Handling & Reliability: async failure modes (unhandled rejections, swallowed errors, broken retries, incomplete rollback), mandatory UI failure/progress state matrix (loading/empty/error/offline/timeout/partial success), assumption audit at trust boundaries
- Accessibility framed against WCAG 2.2 AA with two new subsections: 7.6 Focus Management & Dynamic Content (focus trap/restore, live regions for toasts/validation/loading, custom-widget ARIA) and 7.7 Forms, Zoom & Motion (autocomplete, error recovery, 200% zoom, large text, reduced motion)
- Findings now require Evidence and carry Repro steps plus a Confidence rating (Confirmed / High Confidence / Needs Verification); speculative findings are dropped, and findings are prioritized by real-world impact and exploitability
- Report adds a Release Recommendation (Safe to ship / Ship with known risks / Do not ship, with justification) and two new Action Plan lists: Quick Wins and Needs Architecture / Deeper Investigation
- Health Scorecard expands to 21 rows (overall /210)

**v2.1** (July 2026)
- Added four go-live categories (18 total): Section 15 Payment Security & PCI Compliance (card-data flow/scope, SAQ level cheat sheet, money-op integrity, PCI operational gates), Section 16 Concurrency & Multi-Instance Safety (in-memory state inventory with safe/degraded/broken classification, DB race patterns, horizontal-scaling readiness), Section 17 Load & Capacity Readiness (harness coverage of money paths, backpressure/limits, hot-path efficiency), Section 18 Penetration-Test Readiness (API attack surface incl. GraphQL hardening and public-route inventory, auth hardening, SSRF/upload/SQL input hardening, external pen-test gate)
- Health Scorecard expands to 18 rows (overall /180)
- Motivated by first go-live: these areas were invisible to the 14-category audit

**v2.0** (April 2026)
- Major report format overhaul: 10-section canonical report template replacing the minimal v1 template
- Added Automated Verification Suite: lint, unit tests, coverage report (test:cov), and build run as part of every audit
- New sections: Executive Summary (plain-English verdict), Health Scorecard (0-10 per category with grades), Risk Heat Map (severity counts at a glance), Root Cause Patterns (cluster related findings), Comparison to Last Audit (delta tracking), Action Plan (tiered: Fix Now / This Sprint / Schedule Later)
- Finding IDs (SEV-NNN) for cross-referencing between sections
- Effort estimates on every finding (Quick / Moderate / Significant)
- Report saved to Desktop as timestamped file for audit history
- All tables use ASCII box-drawing format for terminal readability
- Known/accepted exceptions (documented in CLAUDE.md) are excluded from open finding counts
- E2E tests remain opt-in (skipped by default)
- 14 audit categories unchanged

**v1.8** (February 2026)
- Added Section 14 "Technical Debt Audit"
- Covers code complexity (cyclomatic/cognitive), dependency health and age tracking, TODO/FIXME/HACK inventory with git blame age analysis, dead code and unused exports detection, migration and deprecation backlog, coupling and cohesion analysis
- Introduces Technical Debt Scorecard with weighted scoring (0-120 scale) and rating system
- Includes debt scoring formula for dependencies
- 14 audit categories total

**v1.7** (January 2026)
- Changed Resource Constraints to prohibit ALL parallel agents
- All audit checks must now run sequentially using a single agent
- No Task tool agents or background commands permitted
- Prevents memory exhaustion on constrained systems

**v1.6** (January 2026)
- Added Section 13 "Test Coverage Audit"
- Covers test documentation (TEST-SUITE.md), coverage thresholds, test file coverage, test quality, E2E tests, and test suite health
- References external test documentation file for project-specific test catalogs
- Added comprehensive audit commands and checklists
- 13 audit categories total

**v1.5** (January 2026)
- Added Section 7.5 "Mobile Touch Interactions" to Accessibility Audit
- Covers touch target size, touch + drag + click conflicts, preventDefault issues
- Includes anti-pattern and correct pattern code examples
- Added audit commands and checklist for mobile touch issues

**v1.4** (January 2026)
- Enhanced Section 5.9 "Code Duplication & DRY Violations" with specific guidance for cross-service function duplication
- Added audit commands to detect similar implementations across services
- Added list of common patterns to check for centralization (search, pagination, date formatting, etc.)
- Added action item to extract to `common/utils/` when similar functions found

**v1.3** (January 2026)
- Minor improvements

**v1.2** (January 2026)
- Added Developer Experience & Code Consistency Audit (Section 5)
- Covers module structure, naming conventions, import/export patterns, error handling consistency, type definitions, service patterns, test patterns, documentation, and code duplication
- Marked as "Early-stage priority" for new projects
- 12 audit categories total

**v1.1** (January 2026)
- Added comprehensive Software Architecture Audit (Section 1)
- 11 audit categories total

**v1.0** (January 2026)
- Initial production audit skill
- 10 audit categories
- Report template

---

*End of Production Readiness Audit Skill*
