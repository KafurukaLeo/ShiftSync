# ShiftSync

**Workforce scheduling, time tracking, leave management, branch
approvals, and payroll-ready exports.**

> Product and implementation blueprint based on the supplied *ShiftSync
> --- Build specification v1.0* (November 8, 2026). This README
> describes the target system and a recommended maintainable PERN
> architecture. It does not claim that every listed feature is already
> implemented.

------------------------------------------------------------------------

## 1. Project overview

ShiftSync helps a multi-branch company manage staff schedules and
attendance from one system.

The core pay-period workflow is:

**Schedule → Clock in/out → Review and correct → Approve each branch →
Export payroll**

ShiftSync prepares accurate, auditable hours by pay type. **It does not
calculate final wages, taxes, or deductions**; payroll is calculated in
QuickBooks by the accountant.

### Main capabilities

-   Multi-company (multi-tenant) setup with branches and branch
    geofences.
-   Users, roles, branch memberships, staff groups, and shift templates.
-   Weekly scheduling with cross-branch double-booking prevention.
-   Mobile clock-in, breaks, clock-out, and GPS checks.
-   Automatic 30-minute break handling.
-   Manager timesheet review and corrections.
-   Vacation and sick leave requests, decisions, and revocation.
-   Alberta statutory-holiday rules.
-   Twice-monthly pay periods and per-branch approval/version history.
-   Payroll-ready spreadsheet, copy, and print views.
-   Employee view of current/previous period hours and estimated gross
    pay.
-   Read-only audit history.
-   Android update checks and iOS TestFlight/App Store distribution.

### Explicitly out of scope

-   Final payroll, tax, and deduction calculation.
-   Weekly overtime (44-hour threshold) in the current rules.
-   Per-branch pay rates.
-   Public Google Play or public App Store listing in the initial
    distribution plan.

------------------------------------------------------------------------

## 2. Product roles and access

Authorization must be enforced by the API on every request. Hiding a
frontend control is not authorization.

  ----------------------------------------------------------------------------
  Capability        Company admin        Manager       Employee     Accountant
  ---------------- -------------- -------------- -------------- --------------
  Company                     All            ---            ---            ---
  settings,                                                     
  branches, pay                                                 
  schedule,                                                     
  holidays,                                                     
  adjustment types                                              

  Add/edit people             All  Own branches,            ---            ---
                                       excluding                
                                          admins                

  Reset password              Any   Staff at own            ---            ---
                                        branches                

  Schedule shifts    All branches   Own branches            ---            ---

  Clock in/out and       Optional           Self           Self            ---
  breaks                                                        

  View timesheets             All   Own branches    Own (phone)            All

  Correct draft               All   Own branches            ---            ---
  timesheets                                                    

  Remove                      Any           Only            ---            ---
  adjustment                         adjustments                
                                    they created                

  Request leave               ---           Self           Self            ---

  Decide leave                All    Staff whose            ---      View only
  requests                        primary branch                
                                    they manage;                
                                       never own                

  Approve/reopen              All   Own branches            ---            ---
  branch period                                                 

  Payroll                     All   Own branches            ---            All
  sheet/export                                                  

  Audit log                   All   Own branches            ---            All
  ----------------------------------------------------------------------------

**Tenant boundary:** every business record belongs to a company. A user
must never access another company's data. Branch-scoped roles must also
be constrained to their assigned branches.

Recommended authorization context for each request:

1.  Authenticated user.
2.  Active company/tenant.
3.  Role.
4.  Branch memberships and primary branch.
5.  Resource ownership and period/branch status.

Never trust `companyId`, `branchId`, `userId`, role, or approval scope
supplied by the client without checking it against the authenticated
user's server-side context.

------------------------------------------------------------------------

## 3. Recommended technology stack

This is a PERN-based architecture with a dedicated mobile client.

  -----------------------------------------------------------------------
  Layer                   Technology              Responsibility
  ----------------------- ----------------------- -----------------------
  Database                PostgreSQL 16+          Relational source of
                                                  truth, constraints,
                                                  transactions

  ORM/data access         Prisma                  Schema, migrations,
                                                  typed database access

  API                     Node.js + Express +     Business rules,
                          TypeScript              validation,
                                                  authorization, audit,
                                                  exports

  Web                     React + Vite +          Admin, manager, and
                          TypeScript              accountant workspace



  API contracts           REST, versioned under   Stable web/mobile
                          `/api/v1`               integration

  Validation              Zod (recommended)       Request and environment
                                                  validation

  API documentation       OpenAPI / Swagger       Discoverable API
                          (recommended)           contract

  Testing                 Vitest/Jest +           Unit, API integration,
                          Supertest +             web/mobile flows
                          Playwright/Detox        
                          (recommended)           

  Deployment              Docker Compose + Caddy  Containers, HTTPS,
                                                  reverse proxy

  Web serving             Nginx or static hosting React production assets
                          behind Caddy            

  -----------------------------------------------------------------------

Use TypeScript across the API, web, and mobile applications to reduce
contract drift. Keep the backend as a **modular monolith** initially:
one deployable API, clearly separated business modules, and one
PostgreSQL database. This is simpler to operate than premature
microservices while preserving boundaries for future extraction.

### Golden rule

**The server computes every authoritative number.** Hours, breaks,
overtime, holiday eligibility, pay-type totals, and payroll exports must
come from shared backend domain services. Web and mobile clients display
server results; they do not independently calculate payroll hours.

------------------------------------------------------------------------

## 4. Architecture

### 4.1 High-level system

``` mermaid
flowchart TB
    Admin[Admin / Manager / Accountant]
    Employee[Employee / Manager mobile app]
    Web[React + Vite web client]
    Mobile[Expo / React Native app]
    Proxy[Caddy HTTPS reverse proxy]
    API[Express + TypeScript API]
    Auth[Authentication & authorization]
    Domain[Domain services / business rules]
    DB[(PostgreSQL)]
    Files[Private file storage: company logos / exports]
    Jobs[Scheduled jobs: pay periods / reminders / cleanup]

    Admin --> Web
    Employee --> Mobile
    Web --> Proxy
  
    Proxy --> API
    API --> Auth
    API --> Domain
    Domain --> DB
    API --> Files
    Jobs --> Domain
    Domain --> DB
```

### 4.2 Backend layers

Keep HTTP concerns separate from business rules and persistence.

``` text
HTTP route
  └── middleware (authentication, tenant context, validation, authorization)
        └── controller (HTTP input/output only)
              └── application service (use case orchestration)
                    ├── domain service (business rules/calculations)
                    └── repository (Prisma/database access)
```

**Responsibilities**

-   **Routes:** map HTTP methods and paths to controllers.
-   **Middleware:** authenticate, establish tenant context, validate
    payloads, apply rate limits.
-   **Controllers:** translate HTTP requests into service calls and
    consistent responses.
-   **Application services:** coordinate use cases and transactions.
-   **Domain services:** implement pay-period, break, overtime, holiday,
    leave, and approval rules.
-   **Repositories:** encapsulate database queries; always apply
    tenant/branch scope.
-   **Audit service:** record sensitive actions in the same transaction
    as the action where possible.
-   **Export service:** produce payroll outputs from approved
    server-calculated data.

Do not put business calculations in React components, route handlers, or
Prisma model hooks.

### 4.3 Suggested repository layout

A monorepo keeps shared API types and validation contracts aligned.

``` text
shiftsync/
├── apps/
│   ├── api/
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seed.ts
│   │   ├── src/
│   │   │   ├── config/
│   │   │   ├── middleware/
│   │   │   ├── modules/
│   │   │   │   ├── auth/
│   │   │   │   ├── companies/
│   │   │   │   ├── branches/
│   │   │   │   ├── users/
│   │   │   │   ├── memberships/
│   │   │   │   ├── scheduling/
│   │   │   │   ├── time-tracking/
│   │   │   │   ├── timesheets/
│   │   │   │   ├── leave/
│   │   │   │   ├── holidays/
│   │   │   │   ├── pay-periods/
│   │   │   │   ├── approvals/
│   │   │   │   ├── payroll/
│   │   │   │   └── audit/
│   │   │   ├── shared/
│   │   │   │   ├── errors/
│   │   │   │   ├── http/
│   │   │   │   ├── logging/
│   │   │   │   └── utils/
│   │   │   ├── app.ts
│   │   │   └── server.ts
│   │   └── tests/
│   │       ├── unit/
│   │       └── integration/
│   ├── web/
│   │   ├── src/
│   │   │   ├── app/
│   │   │   ├── components/
│   │   │   ├── features/
│   │   │   │   ├── auth/
│   │   │   │   ├── dashboard/
│   │   │   │   ├── branches/
│   │   │   │   ├── people/
│   │   │   │   ├── scheduling/
│   │   │   │   ├── timesheets/
│   │   │   │   ├── leave/
│   │   │   │   ├── approvals/
│   │   │   │   ├── payroll/
│   │   │   │   └── audit/
│   │   │   ├── hooks/
│   │   │   ├── lib/
│   │   │   └── routes/
│       └── tests/
│   
├── packages/
│   ├── contracts/       # Shared DTOs, schemas, API types
│   ├── config/          # Shared lint/TypeScript configuration
│   └── domain-types/    # Shared enums and non-secret constants
├── infrastructure/
│   ├── docker/
│   ├── caddy/
│   └── scripts/
├── docs/
│   ├── architecture/
│   ├── api/
│   ├── business-rules/
│   ├── decisions/
│   └── operations/
├── .env.example
├── docker-compose.yml
├── package.json
├── pnpm-workspace.yaml
└── README.md
```

Each backend module should use a predictable internal structure:

``` text
modules/scheduling/
├── scheduling.routes.ts
├── scheduling.controller.ts
├── scheduling.service.ts
├── scheduling.repository.ts
├── scheduling.schemas.ts
├── scheduling.types.ts
└── scheduling.test.ts
```

For complex domains, split application use cases and domain services
into subfolders rather than creating oversized files.

------------------------------------------------------------------------

## 5. Core domain model

The following is a conceptual model. Exact field names, indexes,
deletion policies, and relation cardinalities should be finalized in
Prisma schema design.

  -----------------------------------------------------------------------
  Entity                              Purpose / important relationships
  ----------------------------------- -----------------------------------
  Company                             Tenant; name, short name, logo,
                                      time zone

  Branch                              Belongs to company; address,
                                      coordinates, geofence radius

  User                                Company user; email, password hash,
                                      role, start date, optional hourly
                                      rate, primary branch

  BranchMembership                    User-to-branch relationship and
                                      membership type

  StaffGroup                          Branch-specific group of users

  ShiftTemplate                       Branch-specific reusable start/end
                                      time

  Shift                               Scheduled user, branch, start/end,
                                      optional template

  TimeEntry                           Actual clock-in/out, GPS readings,
                                      flags, optional shift

  Break                               Time entry, start/end,
                                      automatic/manual, given-back state

  PayPeriod                           Start/end dates and lifecycle

  BranchPeriod                        Pay-period status for an individual
                                      branch

  AdjustmentType                      Built-in/custom adjustment category
                                      and note policy

  TimesheetAdjustment                 User, branch, date, signed minutes,
                                      note, optional shift/leave link

  StatutoryHoliday                    Company holiday date and name

  LeaveRequest                        User, type, status, reason,
                                      decision metadata

  LeaveRequestDay                     One requested date and requested
                                      hours

  ApprovalVersion                     Immutable approval/reversal history
                                      and accepted warnings

  AuditEvent                          Actor, action, target, details,
                                      timestamp, company
  -----------------------------------------------------------------------

### Important data design rules

-   Put `companyId` on tenant-owned records where practical and enforce
    tenant consistency through service validation and database
    constraints.
-   Use foreign keys and unique constraints for integrity; do not rely
    only on frontend validation.
-   Store timestamps consistently (UTC instants) and interpret business
    dates in the company's configured IANA time zone.
-   Use integer minutes for durations and adjustments to avoid
    floating-point rounding.
-   Store the GPS result captured at clock-in/out only. Do not
    continuously track employees.
-   Preserve approved versions and audit events; never delete historical
    approval evidence.
-   Model branch approval separately from the pay period because each
    branch is approved independently.
-   Keep leave-generated adjustments linked to their leave request/day
    so revocation can be safe and traceable.

### Suggested database constraints and indexes

At minimum, evaluate:

-   Unique company/user email (or unique normalized email within tenant,
    according to account policy).
-   Unique active leave request per `(userId, date)` using an
    appropriate partial unique index or equivalent transactional
    enforcement.
-   Indexes on `(companyId, branchId, date/startAt)`,
    `(companyId, userId)`, pay-period dates, leave status, and audit
    timestamp.
-   Foreign keys that prevent cross-company relationships.
-   Database-level protection against duplicate period records.
-   A robust overlap-prevention strategy for shifts across all branches.
    A PostgreSQL exclusion constraint can be considered with a range
    column and carefully designed active-row semantics; otherwise use a
    transaction plus locking and a final conflict query.

------------------------------------------------------------------------

## 6. Business rules --- source of truth

Implement these rules in backend domain services and cover them with
automated tests. The company time zone in the supplied specification is
`America/Edmonton`.

### 6.1 Pay periods

-   Twice monthly: day 1--15 and day 16--last day of month.
-   The current and next period must always exist.
-   Period records are created automatically.
-   Approval status is per branch: `Draft → Approved → Exported`.
-   Reopening returns the branch to `Draft`.
-   Do not offer approval before the period has ended.
-   Overnight shifts belong to the calendar day on which the shift
    started.

### 6.2 Breaks

-   A manually started break ends automatically after 30 minutes if not
    ended sooner.
-   If no break was recorded by clock-out, deduct an automatic 30
    minutes, capped at the shift length.
-   If a recorded break exists, use it; do not also apply the automatic
    deduction.
-   Managers may give back an automatic break. Record who performed the
    action and when.
-   Every shift qualifies; there is no minimum shift length for the
    automatic-break rule.
-   Persist break records and their origin/status; do not calculate
    historical breaks differently on every read.

### 6.3 Paid hours and daily overtime

-   Paid shift minutes = clock-out − clock-in − break minutes,
    plus/minus linked adjustments.
-   Daily total includes shifts across all branches plus non-shift
    adjustments such as Travel.
-   First 8.00 paid hours per employee per day are Regular.
-   Paid time above 8.00 hours is Overtime.
-   Attribute daily overtime to the employee's last shift of that day.
-   Sick and Vacation are not worked time and do not count toward the
    8-hour threshold.
-   Weekly overtime is disabled in the current specification.
-   The 8-hour daily overtime rule is fixed and read-only in settings.

### 6.4 Statutory holidays

-   Preload Alberta statutory holidays for a selected year;
    administrators may edit the list.
-   Eligibility: active on the holiday date and either in a staff group
    or ever scheduled.
-   Eligible employee who does not work: 8.00 hours of `Stat not worked`
    at their primary branch, including part-time employees.
-   If the employee works: all paid hours are `Stat worked` at 1.5×. Do
    not also add Regular, Overtime, or the extra 8 hours.
-   Sick/Vacation requests on statutory holidays are rejected.
-   Approved versions retain the holiday set used at approval. Holiday
    changes after approval require reopening to apply.

### 6.5 Out of Office / leave

-   Types: Vacation and Sick.
-   Sick: up to 14 days in the past, and any future date.
-   Vacation: today onward, or past dates only when their period is
    still Draft.
-   Statutory holidays and dates with an active request cannot be
    selected.
-   Hours per day: 0.5--12 hours in 0.5-hour increments; default 8.
-   Company admin or a manager of the requester's primary branch may
    decide. No one may approve their own request.
-   Decline requires a reason.
-   Approval creates one locked Sick/Vacation adjustment per day at the
    primary branch, with note `Out of Office request`.
-   Revocation requires a reason and is allowed only while the period is
    Draft; remove the linked adjustments transactionally.
-   Period approval warns about pending requests.
-   The server revalidates every date and rule on submission; if any day
    fails, reject the entire request.

### 6.6 Branch approval and history

-   Block approval if anyone has an open shift.
-   Show warnings for flagged location, missed breaks, holiday changes,
    pending leave, and overtime dependent on another branch still in
    Draft.
-   Permit an authorized approver to accept warnings explicitly; persist
    the accepted warning list in the approval version.
-   After approval, timesheet edits are refused until the branch is
    reopened.
-   Reopen requires a reason. Mark the previous version `Reversed`,
    preserve it, and create the next approval as version `N+1`.
-   If cross-branch hour allocation changes after approval, flag that
    the split changed since approval.

### 6.7 Payroll export

One row per staff member, with columns in this exact order:

1.  Employee
2.  Regular (1.0×)
3.  Overtime (1.5×)
4.  Stat worked (1.5×)
5.  Stat not worked (1.0×)
6.  Vacation
7.  Total Pay Hours
8.  Sick

`Total Pay Hours` is the sum of Regular, Overtime, Stat worked, Stat not
worked, and Vacation. Sick is displayed but excluded from Total Pay
Hours under the current specification.

Export requirements:

-   XLSX title: `<short name> – Hours <period>`.
-   Include company logo and live formulas for totals.
-   Include a TOTAL row.
-   Prevent spreadsheet formula injection for untrusted text beginning
    with `=`, `+`, `-`, or `@`.
-   Support tab-separated copy and print.
-   Show a red `Not final — <branch> still in Draft` banner until every
    branch is approved.
-   Log every download and copy.
-   Set export responses to `Cache-Control: no-store`.

### 6.8 Clocking and location

-   Refuse a second clock-in when the employee already has an open
    shift; return the open shift details.
-   If location permission is denied or GPS cannot obtain a fix, record
    the clock event and flag `no location`.
-   If outside the branch radius, record the event and flag
    `outside site`; do not silently reject it.
-   Collect location only at clock-in and clock-out, never continuously.
-   Schedule local reminders for shift start, shift end, and approaching
    break end. The phone handles reminders locally; cap scheduled
    reminders at 60.

------------------------------------------------------------------------

## 7. API conventions

Use a versioned REST API. The following is a recommended endpoint map,
not a claim that every endpoint currently exists.

### General conventions

-   Base path: `/api/v1`
-   JSON request/response bodies.
-   ISO-8601 timestamps for instants; `YYYY-MM-DD` for business dates.
-   Validate all request bodies, params, and query strings.
-   Use consistent pagination (`page`, `limit`) or cursor pagination for
    large audit/time-entry lists.
-   Return stable machine-readable error codes.
-   Derive company and permitted branch scope from the authenticated
    user; never accept client-supplied scope as authoritative.
-   Use idempotency keys for retry-sensitive mobile clock and leave
    submission operations where appropriate.

### Endpoint groups

  -----------------------------------------------------------------------------------------------
  Module                              Example endpoints
  ----------------------------------- -----------------------------------------------------------
  Auth                                `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`,
                                      `POST /auth/change-password`

  Companies                           `GET /company`, `PATCH /company`, `POST /company/logo`

  Branches                            `GET /branches`, `POST /branches`, `GET /branches/:id`,
                                      `PATCH /branches/:id`

  People                              `GET /users`, `POST /users`, `GET /users/:id`,
                                      `PATCH /users/:id`, `POST /users/:id/deactivate`,
                                      `POST /users/:id/reset-password`

  Memberships/groups                  `/memberships`, `/staff-groups`, `/shift-templates`

  Scheduling                          `GET /schedule`, `POST /shifts`, `PATCH /shifts/:id`,
                                      `DELETE /shifts/:id`

  Time tracking                       `POST /time/clock-in`, `POST /time/:entryId/breaks/start`,
                                      `POST /time/:entryId/breaks/end`,
                                      `POST /time/:entryId/clock-out`

  Timesheets                          `GET /timesheets`, `GET /timesheets/:userId`,
                                      `POST /timesheet-adjustments`,
                                      `DELETE /timesheet-adjustments/:id`

  Leave                               `GET /leave-requests`, `POST /leave-requests`,
                                      `POST /leave-requests/:id/approve`,
                                      `POST /leave-requests/:id/decline`,
                                      `POST /leave-requests/:id/revoke`

  Holidays                            `GET /holidays`, `POST /holidays/import`, `POST /holidays`,
                                      `PATCH /holidays/:id`, `DELETE /holidays/:id`

  Pay periods                         `GET /pay-periods`, `GET /pay-periods/:id/branches`

  Approvals                           `POST /pay-periods/:periodId/branches/:branchId/approve`,
                                      `POST /pay-periods/:periodId/branches/:branchId/reopen`

  Payroll                             `GET /pay-periods/:periodId/payroll`,
                                      `GET /pay-periods/:periodId/payroll.xlsx`,
                                      `GET /pay-periods/:periodId/payroll.tsv`,
                                      `GET /pay-periods/:periodId/payroll/print`

  Audit                               `GET /audit-events`

  App version                         `GET /app-version/android`
  -----------------------------------------------------------------------------------------------

### Response shape

A consistent envelope makes clients easier to maintain:

``` json
{
  "data": {},
  "meta": {
    "requestId": "request-id"
  }
}
```

Example error:

``` json
{
  "error": {
    "code": "PERIOD_APPROVED",
    "message": "Approved — reopen to edit.",
    "details": {}
  },
  "meta": {
    "requestId": "request-id"
  }
}
```

Do not expose stack traces, SQL details, password hashes, or internal
secrets in API responses.

------------------------------------------------------------------------

## 8. Frontend application structure

### Web application

The web app serves company admins, managers, and accountants.

Recommended navigation:

-   Dashboard
-   Branch switcher (only branches the user may access)
-   People
-   Staff groups and shift templates
-   Schedule
-   Timesheets / pay periods
-   Out of Office
-   Payroll
-   Audit log
-   Company settings (admin only)
-   My account

Key screens:

-   **Branches:** list, create/edit, map point, geofence radius.
-   **People:** role, branch membership, primary branch, start date,
    optional rate, deactivate/reset password.
-   **Schedule:** weekly branch view, assign individuals or groups, show
    overlap errors.
-   **Timesheets:** branch status, approval actions, payroll summary,
    detail toggle, history, exports.
-   **Person timesheet:** paper-style daily rows and manager correction
    controls.
-   **Out of Office:** pending/approved/declined requests and
    reason-required actions.
-   **Audit:** read-only, filterable event history.

### Mobile application

The Expo app serves employees and managers.

Main screens:

-   Sign in.
-   Dashboard with today's shift and large Clock in / Clock out
    controls.
-   Start/End break and break timer.
-   Week schedule (Monday--Sunday).
-   Out of Office request flow: type → dates → hours → summary →
    confirm.
-   Money view for current and previous period.
-   More: reminders, account, sign out, app version, Android update
    check.

Mobile should display authoritative hours and pay estimates returned by
the API. GPS permission should be requested with a clear explanation and
only used for clock events.

------------------------------------------------------------------------

## 9. Security, privacy, and audit

### Authentication and sessions

-   Hash passwords with Argon2id or bcrypt using a current secure
    configuration.
-   Apply password strength validation and rate-limit sign-in attempts.
-   Session lifetime: 8 hours, then require sign-in again.
-   Prefer secure, HTTP-only, same-site cookies for the web session
    where deployment architecture permits. For mobile, use a carefully
    managed token/session strategy and secure device storage.
-   Provide logout and password-change flows; revoke relevant sessions
    after password reset.

### Authorization and tenant isolation

-   Enforce role, company, and branch scope server-side for every
    request.
-   Never trust client-provided tenant or branch IDs.
-   Managers may only act within their assigned branches and cannot
    decide their own leave requests.
-   Accountants are read-only.
-   Platform/company administration must not accidentally grant
    cross-tenant access unless explicitly intended and audited.

### Operational security

-   HTTPS everywhere, HSTS enabled, and no plain HTTP access through the
    server IP.
-   Keep secrets, signing keys, keystores, and `.env` files out of
    source control.
-   Validate and sanitize inputs; use Prisma parameterized queries.
-   Apply upload type/size limits to logos (PNG/JPG, maximum 1 MB per
    specification).
-   Protect exports from formula injection.
-   Avoid caching payroll/export responses.
-   Log security-relevant events without logging passwords, tokens, or
    unnecessary sensitive location data.

### Audit events

At minimum, audit:

-   Password resets.
-   Added/removed timesheet adjustments.
-   Break give-back actions.
-   Branch approvals and reopens.
-   Leave approvals, declines, and revocations.
-   Payroll downloads and copy actions.
-   Holiday changes.
-   Company settings and logo changes.

Audit events are read-only and must not be deleted through normal
application workflows. Include actor, company, action, target,
timestamp, and structured details. For critical mutations, write the
business change and audit event in one database transaction.

### Location privacy

Capture GPS only at clock-in and clock-out. Do not implement background
or continuous employee tracking. Retain only the location data needed
for the stated attendance and audit purposes, subject to the company's
retention policy.

------------------------------------------------------------------------

## 10. Testing strategy

Test domain rules independently of HTTP and UI. Then test API
authorization, persistence, and full user workflows.

### Backend unit tests

-   Pay-period boundary calculations.
-   Overnight shift business-date assignment.
-   Automatic/manual break calculation and give-back.
-   Daily regular/overtime split across branches.
-   Travel and other adjustment effects.
-   Holiday eligibility and pay classification.
-   Leave date validation and hour increments.
-   Payroll total inclusion/exclusion.
-   Approval warning generation.

### API integration tests

-   Tenant isolation.
-   Manager branch isolation.
-   Accountant read-only enforcement.
-   Admin and manager permission matrix.
-   Double clock-in prevention.
-   Cross-branch schedule overlap prevention.
-   Leave approval self-approval rejection.
-   Atomic leave approval/revocation.
-   Approval blocked by open shifts.
-   Approved timesheet edits rejected.
-   Reopen creates a new version and preserves the old one.
-   Export audit and no-store headers.

### Frontend and mobile tests

-   Web role-based navigation and guarded actions.
-   Weekly schedule interactions and overlap feedback.
-   Timesheet review, approval, reopen, and export.
-   Mobile clock and break flows.
-   GPS denied/no-fix/outside-geofence states.
-   Leave request calendar restrictions.
-   Offline/network failure messaging and retry safety.

### Required acceptance examples

These cases come directly from the build specification and should become
regression tests:

  --------------------------------------------------------------------------
                            \# Input                 Expected result
  ---------------------------- --------------------- -----------------------
                             1 09:00--17:00, no      Automatic 0:30 break;
                               break recorded        7.50 total; 7.50
                                                     Regular

                             2 09:00--18:30,         9.00 total; 8.00
                               recorded 0:30 break   Regular; 1.00 Overtime

                             3 South 8.00 h +        8.00 Regular; 3.00
                               Westside 3.00 h on    Overtime on last
                               same day              (Westside) shift

                             4 7.50 h worked + 1.00  8.50 day total; 8.00
                               h Travel              Regular; 0.50 Overtime

                             5 Holiday, works 6.00 h 6.00 Stat worked; no
                                                     Regular/Overtime/Stat
                                                     not worked

                             6 Holiday, works 10.00  10.00 Stat worked; no
                               h                     Overtime

                             7 Eligible employee     8.00 Stat not worked at
                               does not work on      primary branch
                               holiday               

                             8 Hired after holiday   No holiday hours

                             9 Vacation Nov 10--13,  Nov 11 disabled;
                               8 h/day, Nov 11 is a  approval creates 24
                               holiday               Vacation + 8 Stat not
                                                     worked; Total Pay Hours
                                                     32

                            10 Sick request for a    Rejected
                               date 20 days ago      

                            11 Manager approves own  Rejected
                               leave                 

                            12 Approve branch while  Blocked
                               employee is clocked   
                               in                    

                            13 Sick 8 h and Regular  Total Pay Hours 72;
                               72 h                  Sick excluded

                            14 Reopen, edit, and     Version 1 Reversed;
                               approve again         version 2 Final; both
                                                     and audit entries
                                                     retained
  --------------------------------------------------------------------------

------------------------------------------------------------------------

## 11. Local development setup

The commands below describe the recommended workspace workflow. Adjust
package scripts to the actual implementation.

### Prerequisites

-   Node.js LTS.
-   pnpm (or another single package manager used consistently).
-   Docker and Docker Compose.
-   PostgreSQL 16+ (Docker is recommended for local development).

### Clone and install

``` bash
git clone <repository-url>
cd shiftsync
pnpm install
```

### Configure environment

Create local environment files from the examples:

``` bash
cp .env.example apps/api/.env
```

Example API environment variables:

``` dotenv
NODE_ENV=development
PORT=4000
DATABASE_URL=postgresql://shiftsync:shiftsync@localhost:5432/shiftsync?schema=public
APP_ORIGIN=http://localhost:5173
API_BASE_URL=http://localhost:4000/api/v1
SESSION_SECRET=replace-with-a-long-random-secret
COMPANY_DEFAULT_TIME_ZONE=America/Edmonton
LOG_LEVEL=debug
```

Use strong random secrets in real environments. Never commit `.env`
files. The exact environment variable names should match the
implementation.

### Start infrastructure and database

``` bash
docker compose up -d postgres
pnpm --filter api prisma:generate
pnpm --filter api prisma:migrate
pnpm --filter api db:seed
```

### Run applications

``` bash
# API
pnpm --filter api dev

# Web
pnpm --filter web dev


Typical local URLs (if configured as above):

-   Web: `http://localhost:5173`
-   API: `http://localhost:4000`
-   API health: `http://localhost:4000/health`
-   OpenAPI docs: `http://localhost:4000/api-docs`

### Recommended root scripts

``` json
{
  "scripts": {
    "dev": "turbo run dev --parallel",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "format": "prettier --write ."
  }
}
```

These are suggested scripts; add the corresponding package scripts and
task runner configuration before relying on them.

------------------------------------------------------------------------

## 12. Deployment and operations

### Production topology

-   Linux host.
-   Docker Compose services for API, PostgreSQL, web/static server, and
    Caddy.
-   Caddy terminates HTTPS and routes traffic to internal services.
-   PostgreSQL is not publicly exposed.
-   Persistent volumes for database data and required uploaded assets.
-   Private, off-server database backups.

### Deployment checklist

1.  Provision host, DNS, firewall, and HTTPS.
2.  Configure production secrets outside the repository.
3.  Build immutable API and web images.
4.  Take a database backup before migrations.
5.  Run reviewed Prisma migrations.
6.  Deploy API and web containers.
7.  Run health checks and smoke tests.
8.  Verify scheduled pay-period creation.
9.  Verify backup completion and restore procedure.
10. Monitor logs, errors, disk space, and database health.

### Backups and migrations

-   Take a database dump before every migration.
-   Daily off-server backups are recommended.
-   Test restore procedures periodically.
-   Do not edit an already-applied migration; create a new migration.
-   Preserve approval versions and audit history through migrations.
-   Document any data backfill and rollback limitations.


------------------------------------------------------------------------

## 13. Configuration and decisions still requiring confirmation

The build specification identifies the following as configurable or
awaiting accountant decisions. Keep these isolated in validated company
configuration or explicit constants; do not scatter them across UI code.

  -----------------------------------------------------------------------
  Setting / question      Current behavior        Potential change
  ----------------------- ----------------------- -----------------------
  Pay type labels         Regular, Overtime, Stat Rename to match
                          worked, Stat not        QuickBooks
                          worked, Vacation, Sick  

  Total Pay Hours columns Sick excluded; other    Add/remove Sick
                          five pay columns        
                          included                

  Overtime on statutory   No overtime             Higher rate for hours
  holiday                 classification; all     over 8
                          worked hours are Stat   
                          worked at 1.5×          

  Stat not worked         Separate column         Merge into Regular

  Leave columns           Vacation and Sick       Hide or rename

  Weekly overtime         Off                     Enable 44 h/week in
                                                  addition to daily

  Public company sign-up  Open                    Disable while only EZ
                                                  uses it
  -----------------------------------------------------------------------

Additional implementation decisions to confirm before production:

-   Whether the system will support multiple time zones or remain fixed
    to `America/Edmonton` for the initial company.
-   Data retention periods for GPS, time entries, audit events, and
    inactive users.
-   Exact holiday calendar source and the accountant's definition of
    holiday eligibility.
-   Whether managers can export only their own branches (the permission
    matrix indicates own-branch payroll access).
-   Exact approval behavior when cross-branch calculations depend on
    another branch that remains Draft.
-   Logo/file storage provider and retention.
-   Whether public company registration is enabled in the first release.

Record significant technical choices as Architecture Decision Records in
`docs/decisions/` (for example, `ADR-001-modular-monolith.md`).

------------------------------------------------------------------------

## 14. Recommended implementation phases

Build in vertical, testable increments so each phase produces working
behavior.

1.  **Foundation:** monorepo, TypeScript, lint/format, PostgreSQL,
    Prisma, migrations, CI, health checks, logging, error handling.
2.  **Identity and tenancy:** company, branches, users, memberships,
    authentication, session lifecycle, authorization tests.
3.  **Scheduling:** staff groups, shift templates, weekly schedule,
    overlap prevention, branch-scoped management.
4.  **Time capture:** mobile authentication, clock-in/out, GPS flags,
    break lifecycle, reminders.
5.  **Timesheets:** server-side calculations, adjustment types, manager
    corrections, paper-style views.
6.  **Leave and holidays:** holiday calendar, leave requests, decisions,
    locked adjustments, revocation.
7.  **Pay periods and approvals:** automatic period generation, branch
    approval, warnings, immutable versions, reopen.
8.  **Payroll and audit:** pay-type aggregation, XLSX/TSV/print, export
    audit, audit log UI.
9.  **Hardening and release:** security review, end-to-end regression,
    backup/restore drill, monitoring, iOS/Android distribution.

A phase is complete only when its authorization, business rules, audit
behavior (where applicable), tests, and documentation are complete---not
merely when its UI is present.

------------------------------------------------------------------------

## 15. Definition of done

A feature is ready when:

-   Its behavior matches an approved requirement or a recorded decision.
-   Request schemas and response contracts are documented.
-   Authentication, tenant scope, role, and branch permissions are
    tested.
-   Business rules live in backend services and have unit tests.
-   Database writes use appropriate transactions and constraints.
-   Sensitive mutations produce audit events.
-   Errors are actionable and do not leak secrets.
-   Web/mobile clients handle loading, empty, success, and failure
    states.
-   Relevant acceptance and regression tests pass.
-   Operational or configuration changes are documented.

------------------------------------------------------------------------

## 16. Documentation map

Keep this README as the project entry point and split detailed material
into focused documents as the system grows:

``` text
docs/
├── architecture/
│   ├── system-overview.md
│   ├── backend-modules.md
│   └── data-model.md
├── api/
│   ├── conventions.md
│   └── openapi.yaml
├── business-rules/
│   ├── time-and-breaks.md
│   ├── overtime-and-holidays.md
│   ├── leave.md
│   └── approvals-and-payroll.md
├── decisions/
│   └── ADR-001-modular-monolith.md
└── operations/
    ├── local-development.md
    ├── deployment.md
    ├── backups-and-restore.md
    └── mobile-releases.md
```

## 17. Product reference

This README is based on:

-   **Source:** *ShiftSync --- Build specification*
-   **Version:** 1.0
-   **Date:** November 8, 2026
-   **Product context:** EZ Waste Management
-   **Specification author:** Anatoli Rubatemba

Where this README labels an item as "recommended," it is an architecture
recommendation for maintainability, not a requirement explicitly stated
in the product specification. Where the specification marks behavior as
configurable or unresolved, confirm the decision with the
business/accountant before changing the current rules.
