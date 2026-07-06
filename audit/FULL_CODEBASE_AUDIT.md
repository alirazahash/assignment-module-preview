# Furqan LMS — Full Monorepo Code Audit

**Date:** July 3, 2026  
**Scope:** Backend, Web Frontend (4 apps + shared packages), Mobile (React Native)  
**Overall Rating:** 6.8 / 10

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Backend Audit](#3-backend-audit)
4. [Web Frontend Audit](#4-web-frontend-audit)
5. [Mobile App Audit](#5-mobile-app-audit)
6. [Cross-Cutting Concerns](#6-cross-cutting-concerns)
7. [Type Safety (`any` Audit)](#7-type-safety-any-audit)
8. [Dead & Unused Code](#8-dead--unused-code)
9. [Security Review](#9-security-review)
10. [Performance Concerns](#10-performance-concerns)
11. [Naming & Convention Issues](#11-naming--convention-issues)
12. [Recommendations by Priority](#12-recommendations-by-priority)

---

## 1. Executive Summary

| Dimension | Score | Notes |
|-----------|-------|-------|
| Architecture | 7.0 | Monorepo with multi-tenant backend; good structure, but god-service problem |
| Type Safety | 5.5 | Heavy `any` usage across all layers (~2,900+ occurrences) |
| Test Coverage | 2.0 | Virtually zero automated tests in production code |
| Code Duplication | 5.0 | Significant cross-app and intra-file duplication |
| Security | 6.5 | JWT + RBAC implemented, but missing rate-limiting, no structured logging |
| Performance | 6.5 | Redundant DB calls, no caching layer, potential N+1 queries |
| Maintainability | 5.5 | God files (6k+ lines), scattered patterns, inconsistent naming |
| Dead Code | 7.0 | Some dead methods/commented blocks, but not severe |

**Key Strengths:**
- Multi-tenant architecture with Prisma schema separation (base/client)
- Role-based permission system with middleware
- i18n/RTL support across all platforms
- Comprehensive LMS feature set (100+ modules)
- Shared UI package for web frontend

**Key Weaknesses:**
- Almost zero automated test coverage
- God services (student: 6,189 lines, class: 5,516 lines)
- No structured logging (using console.log/error)
- No rate limiting on API endpoints
- 2,900+ `any` type annotations
- Duplicated business logic between admin/client update flows

---

## 2. Architecture Overview

```
furqan-lms-web-mobile/
├── furqan-lms-backend/          # Express + Prisma + TypeScript
│   └── src/
│       ├── controllers/         # 50+ controller directories
│       ├── services/            # Business logic (65,129 total lines)
│       ├── routes/              # 458 route registrations
│       ├── schemas/             # Joi validation schemas
│       ├── middleware/          # Auth, permissions, validation
│       ├── i18n/                # Internationalization
│       └── prisma/              # Base + Client schema (multi-tenant)
│
├── furqan-lms-frontend/         # Turborepo monorepo
│   ├── apps/
│   │   ├── client/              # Teacher/Org owner portal (~2,020 files)
│   │   ├── admin/               # Super admin portal (~590 files)
│   │   ├── student/             # Student/Guardian portal (~624 files)
│   │   └── auth/                # Shared auth flow (~538 files)
│   └── packages/
│       ├── ui/                  # Shared components, hooks, services (195 exports)
│       └── translations/        # i18n locale files
│
├── FlmsApp/                     # React Native (Expo)
│   └── src/
│       ├── screens/             # 107 screens
│       ├── hooks/               # 40+ hook directories
│       ├── components/          # 70+ component directories
│       ├── services/            # API layer (1,888 lines in Api.ts)
│       ├── navigation/          # 8 navigation files
│       └── redux/               # Redux Toolkit slices
│
└── minio/                       # Object storage config
```

**Key Architectural Decisions:**
- Multi-tenant: Each client gets isolated Prisma client DB; base DB for auth/users
- EntityScopeService pattern: 364 calls across services for tenant resolution
- Frontend: Next.js 15 with App Router, Turborepo for shared packages
- Mobile: React Native with Redux Toolkit + custom hooks (no React Query)
- Auth: JWT tokens with role-based permissions middleware

---

## 3. Backend Audit

### 3.1 God Services (Critical)

| File | Lines | Methods | `any` Count |
|------|-------|---------|-------------|
| `services/client/student.service.ts` | 6,189 | 23 public + 30 private | 65 |
| `services/class.service.ts` | 5,516 | ~40 | 102 |
| `services/serviceRequest.service.ts` | 3,061 | ~20 | 51 |
| `services/client/interviewPayment.service.ts` | 2,666 | ~15 | — |
| `services/client/studentProgress.service.ts` | 2,250 | ~12 | — |
| `services/history/studentHistory.service.ts` | 2,223 | ~15 | — |
| `services/assignment.service.ts` | 2,066 | ~15 | 37 |
| `services/auth.service.ts` | 1,792 | ~12 | 11 |
| `services/assignmentScheme.service.ts` | 1,614 | ~10 | 25 |

**Impact:** These files are extremely difficult to review, test, or safely modify. A single bug fix in `student.service.ts` requires understanding 6,189 lines of context.

### 3.2 Dead Code in `student.service.ts`

| Item | Location | Type | Size |
|------|----------|------|------|
| `calculateEnrollmentFee()` | Lines 6088–6189 | Fully dead method (no caller) | ~100 lines |
| `subscriptionLimitService` | Lines 94, 106 | Dead dependency (usage commented out) | ~15 lines |
| Subscription limit check | Lines 1117–1127 | Commented-out validation block | ~11 lines |
| Phone randomization hack | Lines 1189–1191, 3719–3721 | Old bulk-import workaround | ~6 lines |
| `isExistingGuardian` vars | Lines 1232, 1422 | Superseded one-liners | ~2 lines |
| `trackEducationTypeFromClass` | Line 4523 | Disabled history tracking | ~3 lines |
| Unused `getStudentById` call | Line 2991 | Heavy fetch, result never used | 1 call |
| `registrationFeeId` in `calculateEnrollmentFee` | Line 6140 | Assigned but never returned | — |

### 3.3 Duplication in `student.service.ts`

| Pattern | Occurrences | Estimated Duplicated Lines |
|---------|-------------|---------------------------|
| `updateStudent` vs `updateStudentProfile` (same logic) | 2 flows | ~700+ lines |
| Guardian create/link logic | 4 flows (create, complete, updateStudent, updateProfile) | ~400 lines |
| `bcrypt.hash` + email credential sending | ~10 call sites | ~150 lines |
| Education path validation + student level lookup | 4 call sites | ~80 lines |
| Thin history wrappers (`trackStudentClassChange`, etc.) | 5 methods | ~90 lines |

### 3.4 Backend Route / Controller Stats

- **458** registered routes
- **456** controller methods
- **14** public routes (no auth required)
- **50+** controller directories
- **57** Joi schema directories

### 3.5 Logging & Observability

- **No structured logging library** (no winston, pino, bunyan)
- **71** `console.log`/`console.debug` statements in production code
- Error handler is a basic 87-line middleware
- No request ID tracking or correlation
- No APM/tracing integration

### 3.6 Database Patterns

- **Multi-tenant** via `EntityScopeService` (364 calls)
- **8** raw SQL queries (`$queryRaw`/`$executeRaw`)
- **Prisma transactions** used in major flows (15s timeout)
- Missing: connection pooling configuration visibility
- Missing: query performance monitoring
- **Typo in Prisma schema:** `ENROLLEMENT_STATUS` (should be `ENROLLMENT_STATUS`)
- **Typo in folder:** `src/schemas/stuedntProgress/` (should be `studentProgress`)

---

## 4. Web Frontend Audit

### 4.1 App Structure & Duplication

| App | Files | Purpose |
|-----|-------|---------|
| `apps/client` | 2,020 | Teacher/Org owner/Branch manager |
| `apps/student` | 624 | Student & Guardian portal |
| `apps/admin` | 590 | Super admin |
| `apps/auth` | 538 | Login, register, profile completion |
| `packages/ui` | ~195 exports | Shared components, hooks, services |

**Duplicated across apps:**

| Pattern | Details |
|---------|---------|
| `AuthProvider.tsx` | Nearly identical in client, admin, student (3 copies) |
| `LocaleProvider.tsx` | Identical in all 4 apps |
| `NotificationProvider.tsx` | Identical in client, admin, student |
| `useAssignmentScheme.ts` | Duplicated between client & student apps |
| `useSemester.ts` | Duplicated between client & student apps |
| `useStudentProgress.ts` | Duplicated between client & student apps |
| Hardcoded localhost fallbacks | 10 occurrences across AuthProviders |

**Duplicated services (apps/client vs packages/ui):**
- `assignmentScheme.service.ts`
- `attendanceOption.service.ts`  
- `branchManager.service.ts`

### 4.2 Largest Frontend Files

| File | Lines | Concern |
|------|-------|---------|
| `apps/client/components/createAssignment/TableCellRenderer.tsx` | 1,384 | God component |
| `apps/client/components/Student/TransferStudent.tsx` | 1,303 | Complex modal |
| `apps/client/utils/fieldValidationUtils.ts` | 782 | Validation logic |
| `apps/client/utils/tableColumnUtils.tsx` | 634 | Column definitions |
| `apps/client/hooks/useAutoInitialization.ts` | 477 | Init hook with 25 `any` |
| `apps/client/hooks/useSubscriptionManagement.ts` | 328 | Subscription logic |

### 4.3 Component Issues

- **`FQInput`**: Main shared input field used in 113 files, but `isLoginPage` prop is defined and never used
- **Animation CSS**: `.stardust` and other animation classes exist in `globals.css` (896 lines) but are not wired to any components
- **`globals.css`**: 896 lines with potentially unused animation/style blocks
- **No component testing**: Zero `.test.tsx` or `.spec.tsx` files in apps

### 4.4 Hardcoded Values

```typescript
// Found in ALL AuthProvider.tsx files (client, admin, student):
window.location.href = process.env.NEXT_PUBLIC_AUTH_BASE_URL || 'http://localhost:3000';
```

This pattern repeats 10 times. The fallback to `localhost:3000` should never reach production, but there's no build-time validation.

---

## 5. Mobile App Audit

### 5.1 Overview

| Metric | Value |
|--------|-------|
| Total screens | 107 |
| Total hook directories | 40+ |
| Total component directories | 70+ |
| `any` occurrences | 528 (raw), ~451 (typed) |
| Dependencies | 85 |
| Redux slice files | 7 |
| Files using Redux hooks | 55 |

### 5.2 God Files

| File | Lines | Issue |
|------|-------|-------|
| `services/Api.ts` | 1,888 | Monolithic API file with all endpoints |
| `screens/createAssignment/index.tsx` | 1,511 | Single screen file |
| `hooks/useAddClass.ts` | 988 | Complex class creation hook |
| `hooks/useCompleteProfile.ts` | 765 | Profile completion flow |
| `screens/classes/ClassView.tsx` | 753 | Class detail screen |

### 5.3 Input Component Fragmentation

| Component | Usage (files) | Notes |
|-----------|---------------|-------|
| `TxtInput` | 50 files | Custom wrapper |
| `TextField` | 4 files | Different component |
| `textInput` (references) | 54 files | Likely `TxtInput` imports |

Three different text input patterns in the same app. Should be unified to one.

### 5.4 State Management

- **Redux Toolkit** with 7 slices (auth, dashboard, language, profile)
- **55 files** use Redux hooks (`useSelector`/`useDispatch`)
- **No React Query** — all API calls are manual with custom hooks
- **No Context API usage** detected (0 files)
- Pattern: Each hook manually manages loading/error/data states

### 5.5 Dead/Unused Code

| Item | File | Evidence |
|------|------|----------|
| `useStudentClasses.ts` | `FlmsApp/src/hooks/dashboard/useStudentClasses.ts` | Not imported anywhere in codebase |
| Same `.replace()` bug | `useStudentClasses.ts` | Has the same `session_name.replace()` bug as `useStudentMeDashboard.ts` (fixed) |
| `StudentAllClasses` stub | Student web app | Exports `function index()` (lowercase, empty) |

### 5.6 Navigation Structure

8 navigation files:
- `MainStack.tsx`, `AppStack.tsx`, `StudentStack.tsx`
- `TabsNavigator.tsx`, `DrawerNavigator.tsx`, `MainTabs.tsx`
- `ClientStack.tsx`, `AuthStack.tsx`

Navigation types file (`navigation/types.ts`) is 349 lines — reasonable for 107 screens but uses several `any` params.

---

## 6. Cross-Cutting Concerns

### 6.1 Testing

| Layer | Test Files Found | Effective Coverage |
|-------|-----------------|-------------------|
| Backend | 537 (likely generated/prisma) | ~0% for business logic |
| Frontend | 1,151 (likely node_modules) | ~0% for components |
| Mobile | 924 (likely node_modules) | ~0% for screens |

**No meaningful automated tests exist.** All quality assurance is manual.

### 6.2 Error Handling

**Backend:**
- Custom `AppError` class used across 100+ files
- Global error handler middleware (87 lines)
- Pattern: `try/catch` with `AppError.badRequest()` / `AppError.notFound()`
- Missing: Error categorization for monitoring, retry logic for external services

**Frontend:**
- No global error boundary detected in web apps
- API errors handled inline in hooks

**Mobile:**
- No crash reporting integration visible
- Error handling per-hook, no global handler

### 6.3 API Contract

- **No OpenAPI/Swagger documentation**
- **No shared types** between backend and frontend
- Frontend/mobile each define their own response interfaces
- Risk: Backend changes silently break frontend (as seen with `session_name` localization bug)

### 6.4 i18n

- Backend: Custom i18n with validation messages per entity
- Frontend: `next-intl` with shared translations package
- Mobile: Custom localization in `src/localization/`
- **Inconsistency:** Backend returns localized objects (`{name_en, name_ar}`); clients must handle both string and object formats

---

## 7. Type Safety (`any` Audit)

### 7.1 Summary by Layer

| Layer | Raw `any` | Typed `any`* | Files Affected | Density |
|-------|-----------|-------------|----------------|---------|
| **Backend** | 1,221 | ~537 | 148 / 502 files | HIGH |
| **Web client** | 608 | ~545 | 155 / 2,020 files | HIGH |
| **Mobile** | 528 | ~451 | 182 / 1,280 files | MEDIUM |
| **Shared UI pkg** | 339 | ~275 | 115 files | HIGH |
| **Auth app** | 99 | ~85 | 31 files | MEDIUM |
| **Student app** | 90 | ~84 | 28 files | LOW |
| **Admin app** | 34 | ~34 | 19 files | LOW |
| **TOTAL** | **~2,919** | **~2,011** | — | — |

*Typed `any` = `: any`, `as any`, `Promise<any>`, `Record<string, any>`, `any[]` patterns (excluding Joi message strings).

### 7.2 Worst Offenders

**Backend:**
| File | `any` Count |
|------|-------------|
| `class.service.ts` | 99 |
| `student.service.ts` | 63 |
| `serviceRequest.service.ts` | 50 |
| `assignment.service.ts` | 36 |
| `assignmentScheme.service.ts` | 24 |
| `auth.schema.ts` | 47 |
| `studentRegistration.schema.ts` | 47 |
| `package.schema.ts` | 54 |

**Web Frontend:**
| File | `any` Count |
|------|-------------|
| `tableColumnUtils.tsx` | 26 |
| `useAutoInitialization.ts` | 25 |
| `fieldValidationUtils.ts` | 24 |
| `useSubscriptionManagement.ts` | 19 |
| `TableCellRenderer.tsx` | 18 |
| `TransferStudent.tsx` | 17 |
| `subscriptionManagement.service.ts` | 16 |

**Mobile:**
| File | `any` Count |
|------|-------------|
| `createAssignment/index.tsx` | 31 |
| `classes/ClassView.tsx` | 28 |
| `studentForm/StudentFormStep4.tsx` | 22 |
| `assignment/useAssignment.ts` | 16 |
| `useCompleteProfile.ts` | 14 |
| `useAddClass.ts` | 13 |
| `profileNormalizer.ts` | 12 |
| `Api.ts` | 11 |

### 7.3 Common `any` Patterns

1. **Prisma transaction clients** passed as `any` instead of proper types
2. **API response data** typed as `any` instead of response interfaces
3. **Event handlers** (`(e: any)`) instead of proper React event types
4. **Utility functions** accepting `any` params for "flexibility"
5. **Form data objects** accumulating fields as `Record<string, any>`

---

## 8. Dead & Unused Code

### 8.1 Confirmed Dead Code

| Location | What | Lines | Safe to Remove? |
|----------|------|-------|-----------------|
| Backend: `student.service.ts:6088-6189` | `calculateEnrollmentFee()` — no caller | ~100 | YES |
| Backend: `student.service.ts:94,106` | `subscriptionLimitService` dependency — all usage commented | ~15 | YES |
| Backend: `student.service.ts:1117-1127` | Commented subscription limit check | ~11 | YES |
| Backend: `student.service.ts:1189-1191` | Commented phone randomization | ~6 | YES |
| Backend: `student.service.ts:3719-3721` | Duplicate of above | ~6 | YES |
| Backend: `student.service.ts:2991` | Unused `getStudentById` call in `updateStudentStatus` | 1 call | YES |
| Mobile: `useStudentClasses.ts` | Entire file — never imported anywhere | ~150 | YES |
| Frontend: Student app `StudentAllClasses` | Empty stub (`export default function index()`) | ~5 | Review |

### 8.2 Potentially Dead (Needs Verification)

| Location | What | Notes |
|----------|------|-------|
| Backend: `stuedntProgress/` schema folder | Typo'd folder name | Verify it's the only import path |
| Frontend: `globals.css` animations (`.stardust`, etc.) | CSS blocks with no component usage | ~50+ lines |
| Frontend: `FQInput.isLoginPage` prop | Defined but never consumed | Prop removal |
| Mobile: `TextField` component (4 files) | May be superseded by `TxtInput` (50 files) | Check if migrating |

### 8.3 Commented-Out Code

| Layer | Count of Commented Blocks | Action |
|-------|--------------------------|--------|
| Backend (student.service.ts alone) | 8 blocks | Remove |
| Backend (other services) | ~5 | Remove |
| Mobile | ~12 TODO/FIXME items | Review & resolve |
| Frontend | ~2 TODO items | Review & resolve |

---

## 9. Security Review

### 9.1 Findings

| Category | Status | Details |
|----------|--------|---------|
| **Authentication** | OK | JWT with refresh tokens, bcrypt password hashing |
| **Authorization** | OK | Role-based permissions middleware on routes |
| **Rate Limiting** | MISSING | Zero rate-limiting on any endpoint (0 files) |
| **Input Validation** | OK | Joi schemas on routes |
| **SQL Injection** | LOW RISK | Prisma ORM handles parameterization; 8 raw queries to audit |
| **Secrets Management** | OK | `.env` with `.env.example`; not committed |
| **CORS** | PRESENT | Configured (needs production review) |
| **Helmet** | NOT FOUND | No security headers middleware detected |
| **Logging** | MISSING | No structured logging for security events |
| **Dependency Audit** | UNKNOWN | 85 mobile deps, needs `npm audit` |

### 9.2 Critical Security Gaps

1. **No rate limiting** — login, password reset, OTP endpoints are unprotected against brute force
2. **No security headers** (Helmet) — missing X-Frame-Options, CSP, etc.
3. **No audit logging** — no trail of who did what (admin actions, data access)
4. **Hardcoded fallback URLs** — `http://localhost:3000` in production-shipped code (10 instances)
5. **Console.log in production** — 71 instances that could leak sensitive data
6. **15-second transaction timeouts** — could be exploited for resource exhaustion

### 9.3 Recommendations

| Priority | Action | Effort |
|----------|--------|--------|
| P0 | Add rate limiting to auth endpoints (login, OTP, reset) | 2-4 hours |
| P0 | Add Helmet middleware | 30 min |
| P1 | Replace console.log with structured logger (pino) | 1-2 days |
| P1 | Audit 8 raw SQL queries for injection | 2 hours |
| P2 | Add audit trail for admin/write operations | 1 week |
| P2 | Remove localhost fallbacks; fail hard if env missing | 1 hour |

---

## 10. Performance Concerns

### 10.1 Backend

| Issue | Location | Impact |
|-------|----------|--------|
| **Redundant DB fetch** | `student.service.ts:2991` — `getStudentById` called but result unused in `updateStudentStatus` | Wasted query on every status update |
| **Heavy includes** | `getStudentWithIncludes` fetches everything (guardian, classes, attendance) even when not needed | Over-fetching |
| **No caching** | No Redis/memory cache for frequent lookups (roles, education paths, permissions) | Repeated DB hits |
| **Sequential awaits** | Guardian + residence + class updates done sequentially in transactions where some could parallel | Slower writes |
| **In-memory filtering** | `getAllStudents` fetches then filters for scheduling conflicts in JS | Should be DB-side |
| **No pagination standard** | Only 1 file uses skip/take pattern; manual offset in most services | Inconsistent |

### 10.2 Frontend

| Issue | Location | Impact |
|-------|----------|--------|
| **No data caching** | `packages/ui/src/lib/api/apiClient.ts` — no SWR/React Query | Every navigation re-fetches |
| **Large bundle risk** | `TableCellRenderer.tsx` (1,384 lines), `TransferStudent.tsx` (1,303 lines) | No code splitting |
| **Duplicate API calls** | Provider + hook both fetching same data | Wasted requests |

### 10.3 Mobile

| Issue | Location | Impact |
|-------|----------|--------|
| **No React Query/SWR** | All API calls via manual hooks | No caching, deduplication, or background refetch |
| **Monolithic Api.ts** | 1,888 lines in single file | Bundle includes all endpoints even if unused |
| **Redux for server state** | Mixing UI state (language) with server data (dashboard) | Stale data, manual invalidation |
| **No image caching** | No FastImage or similar detected | Network-dependent image loading |

---

## 11. Naming & Convention Issues

### 11.1 Typos in Production Code

| Typo | Correct | Location | Impact |
|------|---------|----------|--------|
| `ENROLLEMENT_STATUS` | `ENROLLMENT_STATUS` | Prisma schema + 5 service files | Baked into DB enum |
| `stuedntProgress/` | `studentProgress/` | `src/schemas/stuedntProgress/` | Folder name in imports |
| `table-essentails.tsx` | `table-essentials.tsx` | `apps/client`, `apps/admin` | File name |

### 11.2 Naming Inconsistencies

| Pattern | Examples | Issue |
|---------|----------|-------|
| File casing | `StudentFormStep4.tsx` vs `studentCard.tsx` | Mixed PascalCase and camelCase |
| Folder casing | `DuplicateClass/` vs `addBranch/` vs `VerifyOTP/` | Inconsistent across screens |
| Interface location | `apps/client/interface/` vs `apps/client/types/` | Two different folders for types |
| Hook exports | Some use default export, others named export | Inconsistent import patterns |
| Service naming | `semester.service.ts` (backend) vs `useSemester.ts` (frontend) | No shared vocabulary |

### 11.3 Architecture Pattern Inconsistencies

| Area | Inconsistency |
|------|---------------|
| Web API layer | Some hooks call `apiClient` directly, others go through service files |
| Mobile state | Some screens use Redux, others use local hook state for same data types |
| Error messages | Backend uses message codes (`StudentMessageCode.NOT_FOUND`); frontend hardcodes strings |
| Date handling | Backend uses `date.util.ts`; frontend uses raw `new Date()`; mobile mixes both |

---

## 12. Recommendations by Priority

### P0 — Pre-Launch (This Month)

| # | Action | Effort | Risk if Skipped |
|---|--------|--------|-----------------|
| 1 | Add rate limiting to auth endpoints | 2-4 hours | Brute-force attacks |
| 2 | Add Helmet security headers | 30 min | OWASP compliance failure |
| 3 | Remove unused `calculateEnrollmentFee` + dead code | 1-2 hours | Code confusion |
| 4 | Fix `useStudentClasses.ts` `.replace()` bug or delete file | 30 min | Crash if imported |
| 5 | Remove hardcoded `localhost` fallbacks; throw on missing env | 1 hour | Accidental localhost in prod |
| 6 | Replace `console.log` with at minimum `console.warn` suppression in prod | 2 hours | Log data leakage |

### P1 — Post-Launch Sprint 1 (Weeks 1-2)

| # | Action | Effort | Benefit |
|---|--------|--------|---------|
| 7 | Add structured logging (pino) | 2 days | Debugging, monitoring |
| 8 | Add React Query / TanStack Query to mobile | 3-5 days | Caching, dedup, background refresh |
| 9 | Type the top-10 `any` files (start with API boundaries) | 3-5 days | Catch bugs at compile time |
| 10 | Consolidate duplicate providers (AuthProvider, LocaleProvider) | 1-2 days | DRY, fewer bugs |
| 11 | Add basic integration tests for critical flows (login, create student, transfer) | 1 week | Regression safety |
| 12 | Move shared hooks to `packages/ui` (useSemester, useAssignmentScheme, useStudentProgress) | 1 day | Remove client/student duplication |

### P2 — Post-Launch Sprint 2 (Weeks 3-4)

| # | Action | Effort | Benefit |
|---|--------|--------|---------|
| 13 | Split `student.service.ts` into domain services | 2-3 weeks | Maintainability |
| 14 | Split `class.service.ts` similarly | 2 weeks | Maintainability |
| 15 | Unify mobile input components (TxtInput + TextField → one) | 3-5 days | Consistency |
| 16 | Add OpenAPI spec + code-generated types | 1-2 weeks | API contract safety |
| 17 | Add Redis caching for education paths, roles, permissions | 3-5 days | Performance |
| 18 | Fix typos (ENROLLEMENT → ENROLLMENT, stuedntProgress → studentProgress) | 1 day + migration | Professionalism |
| 19 | Add error boundaries to web apps | 1 day | Graceful degradation |
| 20 | Audit and remove unused CSS from globals.css | 1 day | Bundle size |

### P3 — Future (Quarter 2+)

| # | Action | Effort | Benefit |
|---|--------|--------|---------|
| 21 | Comprehensive test suite (unit + integration + e2e) | 4-8 weeks | Long-term stability |
| 22 | Break mobile Api.ts into domain modules | 1 week | Tree-shaking, maintainability |
| 23 | Add audit trail for admin actions | 1-2 weeks | Compliance, debugging |
| 24 | Migrate mobile from Redux (server state) to React Query | 2-3 weeks | Modern patterns |
| 25 | Add CI/CD pipeline with lint + type-check + test gates | 1-2 weeks | Quality enforcement |

---

## Appendix A: File Counts

| Layer | TypeScript Files | Total Services Lines |
|-------|-----------------|---------------------|
| Backend (`src/`) | 502 | 65,129 (services only) |
| Frontend (all apps + packages) | ~3,772 | — |
| Mobile (`src/`) | ~1,280 | — |
| **TOTAL** | **~5,554** | — |

## Appendix B: Dependency Summary

| Layer | Package Manager | Dependencies |
|-------|----------------|--------------|
| Backend | npm | Express, Prisma, bcryptjs, Joi, jsonwebtoken |
| Frontend | pnpm (Turborepo) | Next.js 15, React 18, Tailwind, Radix UI, Recharts |
| Mobile | npm | React Native, Redux Toolkit, React Navigation, Expo |

## Appendix C: `student.service.ts` Split Plan (Post-Launch)

```
services/client/
├── student.service.ts              → Thin orchestrator (100-200 lines)
├── student/
│   ├── studentCrud.service.ts      → getAllStudents, getStudentById, getStudentsByIds, deleteStudent, updateStudentStatus
│   ├── studentCreate.service.ts    → createStudent, completeStudentProfile, resendInvitation
│   ├── studentUpdate.service.ts    → updateStudent, updateStudentProfile (consolidated)
│   ├── studentTransfer.service.ts  → transferStudentToClass, transferStudentToBranch, getOldStudentsByBranch
│   ├── studentGuardian.service.ts  → All guardian CRUD, verification, email
│   ├── studentDashboard.service.ts → getStudentDashboard, getStudentSemesterDashboard, getEligibleStudents
│   └── studentProfile.service.ts   → getProfileByToken, getStudentProfile, formatStudentResponse
```

---

## Appendix D: Detailed Evidence File

For the file-by-file version of this audit, including exact duplicate service wrappers, large-file / bundle-risk inventory, concrete design inconsistencies, dead code candidates, and exact cleanup targets, see:

- `FULL_CODEBASE_AUDIT_DETAILS.md`

*End of Audit*
