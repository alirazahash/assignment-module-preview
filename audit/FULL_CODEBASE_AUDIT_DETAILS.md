# Furqan LMS - Detailed Audit Evidence

This file is the evidence-heavy companion to `FULL_CODEBASE_AUDIT.md`.

Goal: list exact files, repeated implementations, dead code candidates, large-file risks, design inconsistencies, and concrete fix targets so the repo can be cleaned up systematically.

Notes:
- Paths are repo-relative.
- Line counts are from the current repository state and are approximate where noted.
- "Dead/unused" means no active callers/imports were found during the audit.

---

## 1. Backend - Exact Findings

### 1.1 Oversized backend files

These are not automatically wrong, but they are the highest-risk maintenance files because bug fixes require broad context:

| File | Approx. lines | Why it is risky |
|---|---:|---|
| `furqan-lms-backend/src/services/client/student.service.ts` | 6189 | Combines CRUD, onboarding, guardian flows, transfer logic, dashboard logic, email side effects, history, and formatting |
| `furqan-lms-backend/src/services/class.service.ts` | 5516 | Class CRUD, teacher logic, education metadata, availability logic, and per-class enrichment in one file |
| `furqan-lms-backend/src/services/serviceRequest.service.ts` | 3061 | Workflow engine, status transitions, email logic, notifications, scheduling, scholarship logic |
| `furqan-lms-backend/src/services/client/interviewPayment.service.ts` | 2666 | Payment + enrollment + interview flow logic concentrated in one service |
| `furqan-lms-backend/src/services/client/studentProgress.service.ts` | 2250 | Calculation-heavy service with wide usage footprint |
| `furqan-lms-backend/src/services/history/studentHistory.service.ts` | 2223 | History formatting, lookups, and resolution logic mixed together |
| `furqan-lms-backend/src/services/assignment.service.ts` | 2066 | Assignment CRUD, validation, and result logic mixed together |
| `furqan-lms-backend/src/services/auth.service.ts` | 1792 | Login/auth, guardian branching, JWT flow, verification flow |

### 1.2 Confirmed dead or effectively unused backend code

| File | Finding | Why it appears unused |
|---|---|---|
| `furqan-lms-backend/src/services/client/student.service.ts` | `calculateEnrollmentFee()` | No callers found anywhere in the repo |
| `furqan-lms-backend/src/services/client/student.service.ts` | `subscriptionLimitService` dependency | Constructor initializes it, but active code does not use it; only commented-out limit check remains |
| `furqan-lms-backend/src/services/client/student.service.ts` | `updateStudentStatus()` calls `getStudentById()` and never uses the result | Redundant heavy fetch before the real query |
| `furqan-lms-backend/src/services/cron/semesterStatusCron.service.ts` | Whole service looks unused | Only self-definition plus commented references in `src/app.ts` |
| `furqan-lms-backend/src/middleware/auth.middleware.ts` | `ORGANIZATION_OWNER_AND_BRANCH_MANAGER_ONLY` export | Only remaining references are commented out in `src/routes/index.ts` |
| `furqan-lms-backend/src/routes/index.ts` | Commented branch route protection | Old commented middleware chain remains in source |

### 1.3 Exact duplication inside `student.service.ts`

These are the concrete places where the same business logic is repeated instead of being centralized:

#### A. Class enrollment side effects duplicated

| Flow | Location | Repeated behavior |
|---|---|---|
| `createStudent()` | `furqan-lms-backend/src/services/client/student.service.ts` around the class assignment block | Creates `studentClass`, assigns time-slot rule, sends enrollment email |
| `updateStudentProfile()` | Same file around the class update block | Repeats class assignment + enrollment email pattern |

#### B. Class transfer side effects duplicated

| Flow | Location | Repeated behavior |
|---|---|---|
| `updateStudent()` | `student.service.ts` in the class-change branch | Capacity update, class history, education-type history, transfer notification |
| `transferStudentToClass()` | `student.service.ts` in the transfer flow | Previous-class traversal, capacity update, class history, education-type history, transfer notification |

#### C. Password reset + credentials email duplicated

Both branches below generate a new password, hash it, fetch user data, and send the same `new-user-credentials` email:

- `furqan-lms-backend/src/services/client/student.service.ts` email-changed branch
- `furqan-lms-backend/src/services/client/student.service.ts` missing-email-record branch

#### D. Guardian create/update logic repeated across four flows

The same guardian decision tree is repeated in:

- `furqan-lms-backend/src/services/client/student.service.ts` -> `createStudent()`
- `furqan-lms-backend/src/services/client/student.service.ts` -> `completeStudentProfile()`
- `furqan-lms-backend/src/services/client/student.service.ts` -> `updateStudent()`
- `furqan-lms-backend/src/services/client/student.service.ts` -> `updateStudentProfile()`

Common repeated work:
- find existing guardian by email
- create base guardian user
- create/update client guardian user
- create/update verification
- send guardian credentials

### 1.4 Repeated CRUD service patterns across backend

These files follow nearly the same create/update/delete/search-index structure and are candidates for a shared CRUD base/helper layer:

- `furqan-lms-backend/src/services/book.service.ts`
- `furqan-lms-backend/src/services/chapter.service.ts`
- `furqan-lms-backend/src/services/common/educationPath.service.ts`
- `furqan-lms-backend/src/services/remark.service.ts`

Common repetition:
- validate entity exists
- create/update/delete via Prisma
- update search index
- return similar success response shape

### 1.5 Exact backend security gaps

#### No app-level hardening middleware

| File | Finding |
|---|---|
| `furqan-lms-backend/src/app.ts` | No `helmet()` middleware found |
| `furqan-lms-backend/src/app.ts` | CORS uses `origin: '*'` with `credentials: true` |
| `furqan-lms-backend/src/app.ts` | Exposes `/uploads` and `/public` as static directories |

#### Rate limiting is partial, not global

| File | Finding |
|---|---|
| `furqan-lms-backend/src/app.ts` | No global rate limiter |
| `furqan-lms-backend/src/routes/publicHelpCenter/publicHelpCenter.route.ts` | Limiter only used on `POST /articles/search` |
| `furqan-lms-backend/src/middleware/rateLimit.ts` | In-memory `Map` storage, so limits do not coordinate across multiple server instances |

#### Public route surface to review

The following route groups are exposed without parent auth middleware at the router index layer:

- `furqan-lms-backend/src/routes/index.ts` -> `/health`
- `furqan-lms-backend/src/routes/index.ts` -> `/public/furqan`
- `furqan-lms-backend/src/routes/index.ts` -> `/public/branches`
- `furqan-lms-backend/src/routes/index.ts` -> `/auth`
- `furqan-lms-backend/src/routes/index.ts` -> `/public/help-center`
- `furqan-lms-backend/src/routes/index.ts` -> `/packages`
- `furqan-lms-backend/src/routes/index.ts` -> `/payments`

Specific public files/endpoints to review:

- `furqan-lms-backend/src/routes/package/package.route.ts` -> public `GET /` and `GET /:id`
- `furqan-lms-backend/src/routes/payment/payment.route.ts` -> public `POST /verify`, `POST /webhook`, `GET /:id`
- `furqan-lms-backend/src/routes/public/studentOnboarding.route.ts` -> public onboarding and semester listing
- `furqan-lms-backend/src/routes/public/branch.route.ts` -> public branch discovery

#### Sensitive logging

Concrete files with production `console.*` logging on request/entity data:

- `furqan-lms-backend/src/controllers/assignmentScheme/assignmentScheme.controller.ts`
- `furqan-lms-backend/src/services/attendance.service.ts`
- `furqan-lms-backend/src/services/assignment.service.ts`
- `furqan-lms-backend/src/services/examResult.service.ts`
- `furqan-lms-backend/src/services/client/student.service.ts`
- `furqan-lms-backend/src/services/serviceRequest.service.ts`

### 1.6 Exact backend performance hotspots

#### N+1 and per-row enrichment patterns

| File | Method / area | Problem |
|---|---|---|
| `furqan-lms-backend/src/services/client/student.service.ts` | `getAllStudents()` | Fetches students, then resolves base user and guardian base user per student |
| `furqan-lms-backend/src/services/client/teacher.service.ts` | `getTeacherDashboardData()` | Repeats per-class performer/progress lookups |
| `furqan-lms-backend/src/services/class.service.ts` | `getAllClasses()` + `enhanceClassWithDetails()` | Per-class metadata enrichment stack can multiply queries |
| `furqan-lms-backend/src/services/history/classHistory.service.ts` | `_resolveTeacherNames()`, `_resolveStudentNames()` | Per-row `mainPrisma.user.findUnique()` calls |
| `furqan-lms-backend/src/services/client/student.service.ts` | `getGuardianChildren()` | Per-child normalization work |
| `furqan-lms-backend/src/services/client/student.service.ts` | `getStudentDashboard()` | Per-semester progress recalculation |
| `furqan-lms-backend/src/services/article.service.ts` | `semanticSearch()` and `semanticSearchPublic()` | Embedding generation + vector SQL + follow-up queries on request path |

### 1.7 Naming and convention inconsistencies in backend

| File | Finding |
|---|---|
| `furqan-lms-backend/src/services/class.service.ts` | Method typo: `getStuedntsInClass` |
| `furqan-lms-backend/src/schemas/stuedntProgress/` | Folder typo: `stuedntProgress` |
| `furqan-lms-backend/prisma/...` and service usage | Enum typo: `ENROLLEMENT_STATUS` |
| `furqan-lms-backend/src/routes/index.ts` | Route segment uses `/staffs` instead of a cleaner `/staff` or `/staff-members` |
| `furqan-lms-backend/src/routes/client/organizationOwner.route.ts` | Uses `teacherDashboardQuerySchema` naming in org-owner context |
| `furqan-lms-backend/src/schemas/branchManager/branchManager.schema.ts` | Same teacher-dashboard naming leak in branch-manager schema |

---

## 2. Web Frontend - Exact Findings

### 2.1 Exact duplicate providers

#### Notification providers

These wrappers duplicate the same responsibility instead of sharing one app-agnostic provider entrypoint:

- `furqan-lms-frontend/apps/client/providers/NotificationProvider.tsx`
- `furqan-lms-frontend/apps/student/providers/NotificationProvider.tsx`
- `furqan-lms-frontend/apps/admin/providers/NotificationProvider.tsx`
- backing shared implementation: `furqan-lms-frontend/packages/ui/src/providers/NotificationProvider.tsx`

The client and student wrappers are byte-identical. The admin version only changes the token handling.

#### Auth providers

Separate auth bootstrapping/logout/redirect logic exists in:

- `furqan-lms-frontend/apps/admin/providers/AuthProvider.tsx`
- `furqan-lms-frontend/apps/client/providers/AuthProvider.tsx`
- `furqan-lms-frontend/apps/student/providers/AuthProvider.tsx`

This is one of the biggest drift risks in the web layer.

#### Locale providers

Similar locale-provider logic is copied into:

- `furqan-lms-frontend/apps/admin/providers/LocaleProvider.tsx`
- `furqan-lms-frontend/apps/auth/providers/LocaleProvider.tsx`
- `furqan-lms-frontend/apps/client/providers/LocaleProvider.tsx`
- `furqan-lms-frontend/apps/student/providers/LocaleProvider.tsx`

### 2.2 Exact duplicated hooks and services

#### Hooks duplicated across apps or app/shared package

| Hook | Duplicate locations |
|---|---|
| `useSemester` | `apps/client/hooks/useSemester.ts`, `apps/student/hooks/useSemester.ts` |
| `useStudentProgress` | `apps/client/hooks/useStudentProgress.ts`, `apps/student/hooks/useStudentProgress.ts` |
| `useAttendanceOptions` | `apps/client/hooks/useAttendanceOptions.ts`, `packages/ui/src/hooks/useAttendanceOptions.ts` |
| `usePermission` | `apps/client/hooks/usePermission.ts`, `packages/ui/src/hooks/usePermission.ts` |

#### Services duplicated across apps or app/shared package

| Service | Duplicate locations | What overlaps |
|---|---|---|
| `attendanceOption.service.ts` | `apps/client/services/attendanceOption.service.ts`, `packages/ui/src/services/attendanceOption.service.ts` | Same attendance option endpoint wrapper |
| `semester.service.ts` | `apps/client/services/semester.service.ts`, `apps/student/services/semester.service.ts` | Same semester CRUD/status endpoints |
| `formConfig.service.ts` | `apps/auth/services/formConfig.service.ts`, `apps/client/services/formConfig.service.ts` | Same form-config endpoints |
| `branchManager.service.ts` | `apps/client/services/branchManager.service.ts`, `packages/ui/src/services/branchManager.service.ts` | Overlapping branch-manager API layer |
| `assignmentScheme.service.ts` | `apps/client/services/assignmentScheme.service.ts`, `packages/ui/src/services/assignmentScheme.service.ts` | Overlapping assignment-scheme API layer |
| `package.service.ts` | `apps/admin/services/package.service.ts`, `packages/ui/src/services/package.service.ts` | Overlapping package API layer |

### 2.3 Exact duplicated API wrappers / repeated endpoint layers

This is the file-level answer to "which duplicated API calls are repeated?"

#### A. Attendance option API duplicated exactly

- `furqan-lms-frontend/apps/client/services/attendanceOption.service.ts`
- `furqan-lms-frontend/packages/ui/src/services/attendanceOption.service.ts`

These wrap the same attendance-option endpoint set and are effectively the same abstraction in two places.

#### B. Semester API repeated across client and student

- `furqan-lms-frontend/apps/client/services/semester.service.ts`
- `furqan-lms-frontend/apps/student/services/semester.service.ts`

Repeated routes include semester list/detail/update-status style flows, which means fixes to one layer can drift from the other.

#### C. Form-config API repeated across auth and client

- `furqan-lms-frontend/apps/auth/services/formConfig.service.ts`
- `furqan-lms-frontend/apps/client/services/formConfig.service.ts`

Repeated wrapper responsibilities:
- get all form configs
- get form config by id
- get form config by type

#### D. Service singleton barrels duplicated

- `furqan-lms-frontend/packages/ui/src/services/index.ts`
- `furqan-lms-frontend/apps/client/services/index.ts`

Both build a default `ApiClient`, instantiate overlapping services, and export singleton instances.

### 2.4 Large bundle-risk inventory in web

These are the exact files most likely to create bundle weight, parsing cost, and review complexity:

#### Giant app components / forms

| File | Approx. lines | Why it is risky |
|---|---:|---|
| `furqan-lms-frontend/apps/client/components/ServiceRequest/ClientServiceRequestForm.tsx` | 2160 | Very large form component; difficult to split and likely pulls many sub-dependencies at once |
| `furqan-lms-frontend/apps/client/components/createAssignment/TableCellRenderer.tsx` | 1384 | Large table-rendering component; likely bloats assignment route bundle |
| `furqan-lms-frontend/apps/client/components/Student/TransferStudent.tsx` | 1303 | Large modal/flow component with likely broad dependency surface |
| `furqan-lms-frontend/apps/student/components/ServiceRequest/form/index.tsx` | 1062 | Large student-side form |
| `furqan-lms-frontend/apps/client/components/Class/index.tsx` | 944 | Large class screen module |
| `furqan-lms-frontend/apps/client/components/registrationFee/form/index.tsx` | 930 | Large form file |

#### Giant hooks / utilities / configs

| File | Approx. lines | Why it is risky |
|---|---:|---|
| `furqan-lms-frontend/apps/client/hooks/useCreateAssignment.ts` | 872 | Heavy hook logic likely bundled with assignment pages |
| `furqan-lms-frontend/apps/client/utils/fieldValidationUtils.ts` | 782 | Big validation utility; hard to tree-shake if imported broadly |
| `furqan-lms-frontend/apps/client/utils/tableColumnUtils.tsx` | 634 | Large shared table definition utility |
| `furqan-lms-frontend/apps/client/utils/routeProtection.config.ts` | 611 | Large route config surface |

#### Giant shared UI modules

| File | Approx. lines | Why it is risky |
|---|---:|---|
| `furqan-lms-frontend/packages/ui/src/components/custom/FQSidebar/index.tsx` | 749 | Shared component can affect many routes/apps |
| `furqan-lms-frontend/packages/ui/src/components/custom/FQFilter/index.tsx` | 689 | Heavy shared component imported across tables/pages |
| `furqan-lms-frontend/packages/ui/src/components/custom/FQTable/index.tsx` | 669 | Core shared table abstraction; bundle impact spans many screens |
| `furqan-lms-frontend/packages/ui/src/components/custom/FQInput.tsx` | 420 | Core shared input abstraction; design drift risk |

#### Global styles and oversized barrels

| File | Approx. lines | Why it is risky |
|---|---:|---|
| `furqan-lms-frontend/packages/ui/src/styles/globals.css` | 896 | Global CSS with repeated blocks/selectors; affects entire app surface |
| `furqan-lms-frontend/packages/ui/src/index.ts` | 195 | Large barrel can encourage broad imports |
| `furqan-lms-frontend/packages/ui/src/services/index.ts` | 163 | Service barrel encourages importing many services indirectly |
| `furqan-lms-frontend/apps/client/services/index.ts` | 98 | Duplicate barrel pattern at app layer |

### 2.5 Exact web design inconsistencies

#### Inputs and forms

Shared input component exists here:

- `furqan-lms-frontend/packages/ui/src/components/custom/FQInput.tsx`

But raw/styled input implementations still exist in feature files such as:

- `furqan-lms-frontend/apps/client/components/Education/path/index.tsx`
- `furqan-lms-frontend/apps/admin/components/education/path/form/index.tsx`

This means the same kind of field can behave/look different depending on the feature module.

#### Route guards and auth UX

Guarding logic is split between:

- `furqan-lms-frontend/apps/client/components/RouteGuard/index.tsx`
- `furqan-lms-frontend/apps/student/components/RouteGuard/index.tsx`
- `furqan-lms-frontend/packages/ui/src/components/custom/RouteGuard/index.tsx`

The client guard also includes subscription-expiry redirect behavior that the student guard does not.

#### Table conventions and naming drift

Examples of inconsistent table helper naming:

- `furqan-lms-frontend/apps/admin/app/[locale]/requests/components/table-essentails.tsx`
- `furqan-lms-frontend/apps/client/app/[locale]/branches/components/table-essentails.tsx`
- `furqan-lms-frontend/apps/client/app/[locale]/students/components/table-essentails.tsx`

The repo uses both `table-essentials` and misspelled `table-essentails`.

#### Admin/client CRUD UIs duplicated instead of shared

Exact duplicated or near-duplicated action/table/form layers include:

- `apps/admin/app/[locale]/books/components/actions.tsx`
- `apps/client/app/[locale]/books/components/actions.tsx`
- `apps/admin/app/[locale]/pages/components/actions.tsx`
- `apps/client/app/[locale]/pages/components/actions.tsx`
- `apps/admin/app/[locale]/surah/components/actions.tsx`
- `apps/client/app/[locale]/surah/components/actions.tsx`

Mirrored form pairs include:

- `apps/client/components/pages/form/index.tsx` and `apps/admin/components/pages/form/index.tsx`
- `apps/client/components/books/form/index.tsx` and `apps/admin/components/books/form/index.tsx`
- `apps/client/components/chapters/form/index.tsx` and `apps/admin/components/chapters/form/index.tsx`
- `apps/client/components/Surah/form/index.tsx` and `apps/admin/components/Surah/form/index.tsx`
- `apps/client/components/Session/index.tsx` and `apps/admin/components/Session/index.tsx`
- `apps/client/components/Education/type/form/index.tsx` and `apps/admin/components/Education/type/form/index.tsx`

#### Student report duplicated in two apps

- `furqan-lms-frontend/apps/client/app/[locale]/student-report/components/MonthlyView.tsx`
- `furqan-lms-frontend/apps/student/app/[locale]/student-report/components/MonthlyView.tsx`
- `furqan-lms-frontend/apps/client/app/[locale]/student-report/components/SemesterView.tsx`
- `furqan-lms-frontend/apps/student/app/[locale]/student-report/components/SemesterView.tsx`

Related exact duplicate:

- `CustomTooltip.tsx` exists as duplicated implementation across both report areas

#### Payment/card forms duplicated

- `furqan-lms-frontend/apps/auth/components/CardForm.tsx`
- `furqan-lms-frontend/apps/student/components/CardForm.tsx`

### 2.6 Localhost and environment fallback inventory

#### Runtime code fallbacks

Concrete files using `http://localhost:3000` or similar fallback behavior:

- `furqan-lms-frontend/packages/ui/src/lib/api/apiClient.ts`
- `furqan-lms-frontend/apps/client/providers/AuthProvider.tsx`
- `furqan-lms-frontend/apps/student/providers/AuthProvider.tsx`
- `furqan-lms-frontend/apps/admin/providers/AuthProvider.tsx`
- `furqan-lms-frontend/apps/client/hooks/useAuth.ts`
- `furqan-lms-frontend/packages/ui/src/services/firebaseAuth.service.ts`

#### App `.env` files with localhost values

- `furqan-lms-frontend/apps/client/.env`
- `furqan-lms-frontend/apps/student/.env`
- `furqan-lms-frontend/apps/auth/.env`
- `furqan-lms-frontend/apps/admin/.env`

#### Weak secret fallback

- `furqan-lms-frontend/apps/auth/utils/secureStorage.ts` uses `process.env.NEXT_PUBLIC_STORAGE_SECRET_KEY || 'your-secret-key-here'`

### 2.7 Stub / thin-pass-through files to review

These are not all harmful, but they add indirection:

- `furqan-lms-frontend/apps/student/components/Exam/table-essentials.tsx` - re-export only
- `furqan-lms-frontend/apps/auth/types/index.ts` - type re-export only
- `furqan-lms-frontend/apps/client/components/Visitation/VisitFormSteps/index.ts` - one-symbol barrel

---

## 3. Mobile - Exact Findings

### 3.1 Confirmed dead or unreferenced mobile files

#### Unused hooks

- `FlmsApp/src/hooks/dashboard/useStudentClasses.ts`
- `FlmsApp/src/hooks/staff/useStaffsList.ts`
- `FlmsApp/src/hooks/timeSlot/useTimeSlotsList.ts`
- `FlmsApp/src/hooks/useAddStaff.ts`
- `FlmsApp/src/hooks/useStaffCrud.ts`
- `FlmsApp/src/hooks/useAlert.ts` (empty file)

#### Dead wrapper screens

- `FlmsApp/src/screens/progressAndAcadmicInfo/index.tsx`
- `FlmsApp/src/screens/history/classHistory/index.tsx`
- `FlmsApp/src/screens/history/studentHistory/index.tsx`

#### Orphan/placeholder screens

- `FlmsApp/src/screens/allPackages/index.tsx`
- `FlmsApp/src/screens/requestSubmitted/index.tsx`
- `FlmsApp/src/screens/path/index.tsx`

#### Unreferenced components and demos

- `FlmsApp/src/components/dashboard/parent/index.tsx`
- `FlmsApp/src/components/AppHeader.tsx`
- `FlmsApp/src/components/BranchCard.tsx`
- `FlmsApp/src/components/ClassCard.tsx`
- `FlmsApp/src/components/EducationPathCard.tsx`
- `FlmsApp/src/components/EducationTypeCard.tsx`
- `FlmsApp/src/components/Package.tsx`
- `FlmsApp/src/components/SearchBar.tsx`
- `FlmsApp/src/components/SvgExample.tsx`
- `FlmsApp/src/components/cards/EducationCardExample.tsx`
- `FlmsApp/src/components/demos/AlertModalDemo.tsx`
- `FlmsApp/src/components/ThemeToggle.tsx`
- `FlmsApp/src/components/DrawerToggleButton.tsx`
- `FlmsApp/src/components/dropdowns/PackageStatusDropdownIOS.tsx`
- `FlmsApp/src/components/holidays/DeleteHolidayButton.tsx`
- `FlmsApp/src/components/visitForm/steps/VisitFormStepPlaceholder.tsx`
- `FlmsApp/src/components/packagesCards/index.tsx`
- `FlmsApp/src/components/packagesCards/styles.ts`
- `FlmsApp/src/hardcoded-colors.js`

### 3.2 Large file / bundle-risk inventory in mobile

| File | Approx. lines | Why it is risky |
|---|---:|---|
| `FlmsApp/src/hardcoded-colors.js` | 2393 | Huge dead/static color dump; likely should not ship |
| `FlmsApp/src/services/Api.ts` | 1888 | Monolithic API layer with many endpoints |
| `FlmsApp/src/screens/createAssignment/index.tsx` | 1511 | Giant screen file |
| `FlmsApp/src/screens/addRequest/AddRequestDynamicFields.tsx` | 1305 | Giant dynamic form component |
| `FlmsApp/src/hooks/useAddClass.ts` | 988 | Large hook with broad responsibilities |
| `FlmsApp/src/screens/addRequest/useAddRequestViewModel.ts` | 981 | Large view-model hook |
| `FlmsApp/src/components/cards/list-card-generic/index.tsx` | 900 | Heavy generic card abstraction |
| `FlmsApp/src/components/session-timing/index.tsx` | 899 | Large component |
| `FlmsApp/src/screens/transferStudentClassEduPath/index.tsx` | 843 | Large transfer screen |
| `FlmsApp/src/screens/createAssigmentListing/index.tsx` | 799 | Large screen |
| `FlmsApp/src/screens/profileComplete/index.tsx` | 776 | Large screen |
| `FlmsApp/src/hooks/useCompleteProfile.ts` | 765 | Large hook |
| `FlmsApp/src/screens/classes/ClassView.tsx` | 753 | Large screen |
| `FlmsApp/src/components/dashboard/styles.ts` | 714 | Very large styling file |
| `FlmsApp/src/hooks/examinerAvailability/useExaminerAvailability.ts` | 706 | Large hook |
| `FlmsApp/src/components/article/detail/ArticleDetailFeedback.tsx` | 695 | Large component |
| `FlmsApp/src/hooks/articles/useArticleForm.ts` | 689 | Large hook |
| `FlmsApp/src/components/Dropdown/DropdownSearch.tsx` | 668 | Large shared UI component |

### 3.3 Exact API-layer inconsistencies in mobile

The mobile app currently carries three overlapping request patterns:

#### Pattern A - monolithic RTK Query API

- `FlmsApp/src/services/Api.ts`

#### Pattern B - generic RTK Query helper abstraction

- `FlmsApp/src/services/ApiServices.ts`
- `FlmsApp/src/services/SplitApiSetting.ts`

#### Pattern C - React Query + axios

- `FlmsApp/src/services/queries/useQueryApi.js`
- `FlmsApp/src/services/mutations/index.js`

This means the codebase has three ways to perform remote data access.

#### Concrete API inconsistencies

| File | Finding |
|---|---|
| `FlmsApp/App.tsx` | Registers `QueryClientProvider` while Redux also wires RTK Query middleware |
| `FlmsApp/src/services/ApiEndpoints.ts` | Hardcodes `BASE_URL = 'http://192.168.1.7:5002/v1/'` and commented alternatives |
| `FlmsApp/src/services/queries/useQueryApi.js` | Reads `SERVER_URL` from env instead of using the same base URL system |
| `FlmsApp/src/screens/verifyUser/index.tsx` | Bypasses shared API layer and uses direct `axios.post()` against external hardcoded URLs |
| `FlmsApp/src/services/_utils.ts` | Duplicates auth-header preparation already covered elsewhere and appears unused |
| `FlmsApp/src/hooks/dashboard/useStudentMeDashboard.ts` | Comment says one endpoint, implementation uses another endpoint and normalizes multiple shapes |
| `FlmsApp/src/hooks/dashboard/useStudentClasses.ts` | Dead hook still uses the older/more specific student classes endpoint |

### 3.4 Exact state-management inconsistencies in mobile

#### Mixed server-state systems

- `FlmsApp/App.tsx` -> Redux Provider + PersistGate + QueryClientProvider together
- `FlmsApp/src/redux/index.ts` -> typed Redux helpers exist
- Active files still use raw `useSelector` / `useDispatch`, for example:
  - `FlmsApp/src/screens/profile/index.tsx`
  - `FlmsApp/src/screens/editPersonalInformation/index.tsx`
  - `FlmsApp/src/screens/editGuardianInformation/index.tsx`
  - `FlmsApp/src/screens/requests/index.tsx`
  - `FlmsApp/src/hooks/classes/useClass.ts`

#### Manual refresh flags instead of formal invalidation

Examples:

- `FlmsApp/src/hooks/classes/useClass.ts`
- `FlmsApp/src/hooks/useAddClass.ts`
- `FlmsApp/src/hooks/classes/useInviteStudent.ts`
- `FlmsApp/src/hooks/students/useStudents.ts`
- `FlmsApp/src/hooks/students/useAddStudent.ts`

These use `globalThis.*RefreshNeeded` style flags.

### 3.5 Exact design inconsistencies in mobile

#### Setting rows

Two near-duplicate setting row components exist:

- `FlmsApp/src/components/SettingItem/index.tsx`
- `FlmsApp/src/components/SettingsItem.tsx`

#### Stats item components

Two similar components with different prop models:

- `FlmsApp/src/components/stats/StatsItemSection/index.tsx`
- `FlmsApp/src/components/sectionsList/statItem/index.tsx`

#### Search bars

Three search-input paths:

- `FlmsApp/src/components/SearchBar.tsx`
- `FlmsApp/src/components/SearchBarInput/index.tsx`
- `FlmsApp/src/components/sessions/SessionSearchBar.tsx`

#### Roles hooks

Two similarly named hooks:

- `FlmsApp/src/hooks/useRoles.ts`
- `FlmsApp/src/hooks/roles/useRoles.ts`

#### Theme system split

- `FlmsApp/src/theme/ThemeProvider.tsx`
- `FlmsApp/src/navigation/ThemeNavigationContainer.tsx`

The provider exposes theme APIs but hard-codes light mode and no-op setters, while navigation builds its own palette.

#### Input fragmentation

Multiple text-input patterns exist:

- `TxtInput` used broadly across the app
- `TextField` used in a smaller set of files
- additional raw input-related wrappers and screen-specific field components

### 3.6 Exact type-safety hotspots in mobile

Highest concentration of weak typing:

| File | Approx. `any` hits |
|---|---:|
| `FlmsApp/src/screens/createAssignment/index.tsx` | 36 |
| `FlmsApp/src/screens/classes/ClassView.tsx` | 32 |
| `FlmsApp/src/hooks/useCompleteProfile.ts` | 28 |
| `FlmsApp/src/components/studentForm/StudentFormStep4.tsx` | 22 |
| `FlmsApp/src/hooks/assignment/useAssignment.ts` | 18 |
| `FlmsApp/src/hooks/useAddClass.ts` | 14 |
| `FlmsApp/src/utils/profileNormalizer.ts` | 12 |
| `FlmsApp/src/services/Api.ts` | 11 |

Other concrete weak-typing examples:

- `FlmsApp/src/redux/slices/profileSlice.ts` uses `PayloadAction<any>`
- `FlmsApp/src/screens/editPersonalInformation/index.tsx` uses `state: any`, multiple `catch (error: any)`, and `as unknown as` casts
- `FlmsApp/src/hooks/timeSlot/useAddTimeSlot.ts` weak callback/error typing
- `FlmsApp/src/screens/addTimeSlot/index.tsx` weak typing in flow handlers
- `FlmsApp/src/hooks/auth/useOTPVerificationScreen.ts` weak error typing

---

## 4. Exact Fix Queues

### 4.1 Best low-risk cleanup queue

These are the easiest exact removals / consolidations with low product risk:

1. Delete dead mobile hooks and wrappers:
   - `FlmsApp/src/hooks/dashboard/useStudentClasses.ts`
   - `FlmsApp/src/hooks/staff/useStaffsList.ts`
   - `FlmsApp/src/hooks/timeSlot/useTimeSlotsList.ts`
   - `FlmsApp/src/hooks/useAddStaff.ts`
   - `FlmsApp/src/hooks/useStaffCrud.ts`
   - `FlmsApp/src/hooks/useAlert.ts`

2. Delete dead mobile screens / placeholders:
   - `FlmsApp/src/screens/allPackages/index.tsx`
   - `FlmsApp/src/screens/requestSubmitted/index.tsx`
   - `FlmsApp/src/screens/path/index.tsx`
   - wrapper-only history/progress screens if navigation no longer uses them

3. Remove dead backend code:
   - `furqan-lms-backend/src/services/client/student.service.ts` -> `calculateEnrollmentFee()`
   - dead `subscriptionLimitService` usage/comment block
   - unused `getStudentById()` call inside `updateStudentStatus()`
   - `furqan-lms-backend/src/services/cron/semesterStatusCron.service.ts` if the cron is intentionally retired

4. Merge exact duplicate service wrappers:
   - `attendanceOption.service.ts`
   - `semester.service.ts`
   - `formConfig.service.ts`

5. Replace app-specific provider wrappers with one shared adapter layer:
   - AuthProvider
   - LocaleProvider
   - NotificationProvider

### 4.2 Highest-value refactor queue

If you want the biggest maintenance payoff, these are the most valuable exact targets:

1. `furqan-lms-backend/src/services/client/student.service.ts`
2. `furqan-lms-backend/src/services/class.service.ts`
3. `FlmsApp/src/services/Api.ts`
4. `furqan-lms-frontend/apps/client/components/ServiceRequest/ClientServiceRequestForm.tsx`
5. `furqan-lms-frontend/apps/client/components/createAssignment/TableCellRenderer.tsx`
6. `FlmsApp/src/screens/createAssignment/index.tsx`
7. `FlmsApp/src/hooks/useAddClass.ts`

### 4.3 Best first typed cleanup queue

If the goal is to reduce bug risk without big architecture changes, start here:

- `FlmsApp/src/screens/createAssignment/index.tsx`
- `FlmsApp/src/screens/classes/ClassView.tsx`
- `FlmsApp/src/hooks/useCompleteProfile.ts`
- `FlmsApp/src/components/studentForm/StudentFormStep4.tsx`
- `furqan-lms-backend/src/services/class.service.ts`
- `furqan-lms-backend/src/services/client/student.service.ts`
- `furqan-lms-frontend/apps/client/utils/tableColumnUtils.tsx`
- `furqan-lms-frontend/apps/client/hooks/useAutoInitialization.ts`
- `furqan-lms-frontend/apps/client/utils/fieldValidationUtils.ts`

---

## 5. How To Use This File

Recommended workflow:

1. Start with `FULL_CODEBASE_AUDIT.md` for priority and roadmap.
2. Use this file to create concrete cleanup tickets.
3. Make separate tickets for:
   - dead code removal
   - provider/service consolidation
   - large-file splits
   - type-safety cleanup
   - security hardening
   - mobile API-layer unification

Suggested ticket style:

- Title: `Remove dead mobile hook useStudentClasses`
- Files: list exact paths from this document
- Risk: low / medium / high
- Verification: exact flows to smoke test after the change

