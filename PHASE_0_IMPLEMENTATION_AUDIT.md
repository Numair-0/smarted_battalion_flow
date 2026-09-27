# SMARTED BATTALION — Phase 0: Implementation Audit

**Status:** Analysis complete — no application code generated yet, per instruction.
**Source of truth:** DOC-01 (Master Architecture), DOC-02 (Mess), DOC-03 (Quartermaster), DOC-04 (Gate & Visitor), DOC-05 (Duty & Leave).
**Purpose:** Map every documented feature to a concrete prototype component before Phase 1 begins, surface every `[TO BE CONFIRMED]` item, and flag any place the five documents need a prototype-level decision to move forward.

---

## 0.1 — Decisions Already Locked (confirmed in this conversation)

These override the "open" status shown in the documents' own Section 19 checklists:

| # | Decision | Resolves |
|---|---|---|
| 1 | **Duty allocation is manual.** Roster-in-Charge/Adjutant sets the roster by hand (seniority/rotation, as in standard defence-services practice); the system records, publishes, checks leave conflicts, and reports — it does not auto-generate assignments. | DOC-05 §2/§4 central scope question, DOC-05 TBC #1 |
| 2 | **Gate OTP is a labelled DEMO mechanism only** (per this prototype brief) — no SMS/internet-dependent OTP. A local, clearly-marked "Demo Verification" step stands in for the undecided production mechanism. | DOC-04 §4/§16 OTP tension, DOC-04 TBC #3 |
| 3 | **Board of Officers is not a separate login** for the prototype — QM/Admin records the Board's decision on a damage/scrap entry (per DOC-03's "conservative reading" already used when drafting DOC-03). | DOC-03 TBC #1 |
| 4 | **Security Officer** is treated as a **separate role from Adjutant** for the prototype (both can view gate reports), since nothing suggests otherwise and it costs nothing to keep them distinct. | DOC-04 TBC #1 |

Everything else marked `[TO BE CONFIRMED]` in the five documents remains **unconfirmed** and is handled per §0.5 below (configurable, demo-labelled, or simply out of scope) — never silently treated as final.

---

## 0.2 — Consolidated Prototype Role List

The four module documents each name roles independently (Diner/Storekeeper/Secretary for Mess, QM for Quartermaster, Gate Guard/Security Officer for Gate, Roster-in-Charge for Duty), plus Battalion-wide oversight roles (Commandant, Adjutant) and platform Admin. DOC-01 §6 already anticipated this reconciliation. For the prototype, one unified role table is used, backed by granular permissions (not hardcoded role checks) so DOC-01's RBAC model (User → Role → Permission → Module → Action) is implemented literally.

| Role (enum) | Primary Module | Notes |
|---|---|---|
| `ADMIN` | All | Platform-level, least-privilege exempt for user/role management only |
| `COMMANDANT` | All (view/approve) | View + approve leave (final), view all reports, no data entry |
| `ADJUTANT` | Gate, Duty, All (view) | Gate reports, Duty roster/leave management, cross-module view |
| `MESS_SECRETARY` | Mess | Full Mess module access incl. billing/config |
| `STOREKEEPER` | Mess | Mess stock entry only |
| `QUARTERMASTER` | Quartermaster | Full QM module access |
| `GATE_GUARD` | Gate | Entry/exit logging only |
| `SECURITY_OFFICER` | Gate | Report viewing only |
| `ROSTER_INCHARGE` | Duty | Create/publish roster, recommend leave decisions |
| `PERSONNEL` | Self-service | Base role: view own mess account, own duty, own equipment balance, apply for leave, act as gate visit host |

**Permission naming convention:** `<ACTION>_<MODULE>` (e.g. `MANAGE_MESS_STOCK`, `ISSUE_QM_ITEM`, `CREATE_GATE_ENTRY`, `MANAGE_DUTY_ROSTER`, `APPROVE_LEAVE`, `VIEW_AUDIT`, `VIEW_REPORTS`). Full matrix to be implemented as a seed-time permission set, not hardcoded per-route role checks — this keeps the door open for the Battalion to later change who holds which permission without a code change, consistent with DOC-01 §6.

> **Prototype assumption, not a documented requirement:** the exact role list above is a reconciliation Claude is making across four documents that never had to agree with each other on naming. It is logged here explicitly so it can be corrected in one place if the Battalion's actual appointment structure differs.

---

## 0.3 — Shared Platform Entities (DOC-01 §7, §10)

| Entity | Owner | Key Fields |
|---|---|---|
| `User` | Platform | serviceNo (unique), rank, fullName, unitCompany, role, status (ACTIVE/POSTED_OUT/LEAVE), passwordHash, rfidTagId (optional) |
| `Role` / `Permission` | Platform | seed-defined; role→permission mapping table |
| `Session` | Platform | JWT-based; not persisted beyond token expiry, per prototype auth flow |
| `AuditLog` | Platform | user, role, module, action, entity, entityId, timestamp, result, metadata |
| `Notification` (in-app, not push) | Platform | recipientRole/User, type, message, read, createdAt — used for low-stock, maintenance-due, leave-decision alerts |

These live outside any one module's folder (`server/src/modules/{users,roles,audit}` + a small `common/notifications`), consumed by all four feature modules — mirrors DOC-01 §7's "common platform services, not duplicated per module" principle.

---

## 0.4 — Module-by-Module Feature Map

### Module 1 — Smart Mess & Ration (DOC-02)

| Documented Feature | Doc Ref | Mongoose Model(s) | API Group | Frontend Route |
|---|---|---|---|---|
| Personnel/auth (shared) | §4 FR-1 | `User` (shared) | `/api/auth` | `/login` |
| Meal attendance (B/L/T/D, guest, off-mess) | §4 FR-2, §11 | `MealLog` | `/api/mess/meal-logs` | `/mess/attendance` |
| Ration stock in/out | §4 FR-3, §11 | `InventoryItem`, `StockLedger` | `/api/mess/inventory`, `/api/mess/stock-ledger` | `/mess/inventory`, `/mess/stock-ledger` |
| Low-stock alert | §4 FR-3.4 | (derived from `InventoryItem.reorderLevel`) | — triggers `Notification` | Dashboard widget |
| Auto ration deduction on attendance | §4 FR-3.3 (TBC scaling) | `StockLedger` (auto entries) | internal service call | — |
| Monthly bill generation | §4 FR-4, §11 | `MonthlyBill` | `/api/mess/bills` | `/mess/billing` |
| Reports (bill/audit, stock, consumption) | §14 | — (aggregation) | `/api/mess/reports/*` | `/mess/reports` |

**Prototype-only handling:** ration-scale quantity per meal type (DOC-02 TBC #2) and mess-cut formula (TBC #3) ship as **seed-configurable values** (e.g. `config/mess-ration-scale.json`), clearly commented as placeholder, not hardcoded into business logic — so a Battalion-supplied real SOP can replace them without a code rewrite.

### Module 2 — Quartermaster & Equipment (DOC-03)

| Documented Feature | Doc Ref | Mongoose Model(s) | API Group | Frontend Route |
|---|---|---|---|---|
| Equipment master (serialized/bulk) | §4 FR-1, §11 | `EquipmentItem` | `/api/qm/equipment` | `/qm/equipment` |
| Issue / return + digital signature | §4 FR-2, §11 | `IssueReturnLog` | `/api/qm/issue-return` | `/qm/issue-return` |
| Personnel-wise balance | §4 FR-2, §10 | (derived from `IssueReturnLog`) | `/api/qm/balances/:serviceNo` | `/qm/balances` |
| Inspection/maintenance due tracking | §4 FR-3, §11 | `MaintenanceSchedule` | `/api/qm/maintenance-schedule` | `/qm/maintenance` |
| Damage/scrap logging + Board decision | §4 FR-4, §11 | `DamageScrapLog` | `/api/qm/damage-scrap` | `/qm/damage-scrap` |
| Reports | §14 | — (aggregation) | `/api/qm/reports/*` | `/qm/reports` |

**Prototype-only handling:** inspection interval per category (DOC-03 TBC #7) is seed-configurable, not hardcoded. Board of Officers decision is a status field + free-text `boardDecision`, recorded by QM/Admin (per §0.1 decision #3).

### Module 3 — Battalion Gate & Visitor Security (DOC-04)

| Documented Feature | Doc Ref | Mongoose Model(s) | API Group | Frontend Route |
|---|---|---|---|---|
| Visitor entry (category, ID, photo, purpose) | §4 FR-1, §11 | `VisitorLog` | `/api/gate/visitor-log` | `/gate/entry` |
| Vehicle & driver capture | §4 FR-2, §11 | `VehicleLog` | `/api/gate/vehicle-log` | (part of entry form) |
| Demo ID/OTP verification | §4 FR-1.3 (TBC mechanism) | `VisitorLog.otpVerified` | `/api/gate/verify-demo` | (part of entry form) |
| Badge/pass issuance | §4 FR-3, §11 | `VisitorLog.badgeRef` | `/api/gate/visitor-log/:id/badge` | (part of entry form) |
| Exit logging | §4 FR-4 | `VisitorLog.exitTimestamp` | `/api/gate/visitor-log/:id/exit` | `/gate/active` |
| Daily gate activity report | §4 FR-5, §14 | — (aggregation) | `/api/gate/reports/daily-activity` | `/gate/reports` |
| Visitor search/history | §8 UC-07 | — (query) | `/api/gate/search` | `/gate/history` |

**Prototype-only handling:** verification step is explicitly labelled **"Demo Verification"** in the UI, with a code-comment and README note that this is not the production mechanism (per prototype brief's explicit instruction — no SMS OTP, no internet dependency).

### Module 4 — Duty Roster & Leave Management (DOC-05)

| Documented Feature | Doc Ref | Mongoose Model(s) | API Group | Frontend Route |
|---|---|---|---|---|
| Manual roster creation/edit (draft) | §4 FR-1, §11 — **manual, confirmed §0.1** | `RosterEntry` | `/api/duty/roster` | `/duty/roster` |
| Roster publish | §4 FR-1.3 | `RosterEntry.status` | `/api/duty/roster/publish` | `/duty/roster` |
| Leave-conflict check on assignment | §4 FR-1.4/FR-3.1 | cross-query `RosterEntry` + `LeaveRecord` | `/api/duty/roster/conflict-check` | inline warning in roster UI |
| Leave application | §4 FR-2.1, §11 | `LeaveRecord` | `/api/duty/leave` | `/duty/leave/apply` |
| Leave approval workflow | §4 FR-2.2 (TBC chain) | `LeaveRecord.status/approvedBy` | `/api/duty/leave/:id/decision` | `/duty/leave/approvals` |
| Real-time leave status dashboard | §4 FR-2.3 | — (query) | `/api/duty/leave/dashboard` | `/duty/dashboard` |
| Own-duty view | §13 | — (query) | `/api/duty/roster?self=true` | `/duty/my-duty` |

**Prototype-only handling:** leave approval chain (DOC-05 TBC #2) implemented as a single-step approval by `ADJUTANT` or `COMMANDANT` (either can decide) for the prototype, clearly commented as a simplification pending the real SOP; `leaveType` enum ships with a placeholder set (Casual/Earned/Medical) marked TBC per DOC-05 TBC #9.

---

## 0.5 — Consolidated `[TO BE CONFIRMED]` Register (all 5 docs, prototype treatment)

Every item below is either (a) shipped as a seed-configurable value, (b) shipped as a clearly-labelled demo/placeholder in the UI, or (c) excluded entirely as P2/future. None are implemented as if they were settled requirements.

| Source | Item | Prototype Treatment |
|---|---|---|
| DOC-02 | Ration-scale per meal type | Seed-config, placeholder values, commented |
| DOC-02 | Mess-cut formula | Seed-config, placeholder values, commented |
| DOC-02 | Guest meal rate | Seed-config, defaults to same rate as regular meal |
| DOC-03 | Inspection interval per category | Seed-config, placeholder values, commented |
| DOC-03 | Board of Officers login | Excluded — decision recorded by QM/Admin (§0.1 #3) |
| DOC-03 | Serialized vs bulk mapping | `EquipmentItem.isSerialized` flag, seed decides per demo item |
| DOC-04 | OTP mechanism | Demo-labelled verification only (§0.1 #2) |
| DOC-04 | Badge/pass printer hardware | Excluded (P2) — badge is a digital reference + printable view only |
| DOC-04 | Data retention period (visitor photos/ID) | Not enforced in prototype; noted in README as a production decision |
| DOC-05 | Duty distribution model | **Resolved** — manual (§0.1 #1) |
| DOC-05 | Leave approval chain | Simplified single-step approval, commented as placeholder |
| DOC-05 | Leave-balance enforcement | Not enforced in prototype — applications recorded without balance deduction |
| DOC-05 | Roster post-publication edit rules | Allowed in prototype, audit-logged; no separate notification mechanism built |
| All docs | Encryption scope, key management | Out of scope for prototype (DOC-01 §12) — noted in README as production requirement |
| All docs | LAN/network topology, backup schedule | Out of scope for prototype (DOC-01 §4/§15) — README documents the future direction only |

---

## 0.6 — Cross-Document Tension Identified

**Gate OTP vs. offline mandate:** DOC-04 §4 FR-1.3 (proposal-sourced) calls for "OTP/ID verification," while DOC-01 §4 fixes the whole system as air-gapped/no-internet. A real SMS OTP would break that constraint. This was already flagged in DOC-04 itself (§4, §16, TBC #3) rather than silently resolved — the prototype brief's instruction to use a "clearly labelled DEMO verification mechanism" is the correct resolution and is what will be built (§0.1 #2). No other genuine contradiction was found between the five documents; the remaining gaps are absences (things no document specifies), not conflicts (two documents disagreeing).

---

## 0.7 — Proposed Repository Structure

```
smarted-battalion/
├── client/                 # React + TS + Vite
│   └── src/
│       ├── api/             # typed API client functions (TanStack Query hooks)
│       ├── components/      # DataTable, StatusBadge, StatCard, EmptyState, etc.
│       ├── layouts/
│       ├── modules/
│       │   ├── mess/
│       │   ├── quartermaster/
│       │   ├── gate/
│       │   └── duty/
│       ├── pages/           # login, dashboard, admin
│       ├── hooks/ lib/ routes/ types/ utils/
├── server/
│   └── src/
│       ├── config/          # env loading
│       ├── database/        # mongoose connection
│       ├── middleware/      # auth, error handler, RBAC guard
│       ├── common/           # audit, notifications
│       ├── modules/
│       │   ├── auth/ users/ roles/ audit/ reports/
│       │   ├── mess/ quartermaster/ gate/ duty/
│       ├── seed/
│       ├── app.ts server.ts
├── README.md
```

Each module folder = `routes → controllers → services → repositories → models → validators → types`, per the brief's mandated layering.

---

## 0.8 — What Phase 1 Will Build

Foundation only: monorepo scaffold, Vite+TS+Tailwind client shell, Express+TS server shell, MongoDB connection via env var, base layout (SMARTED BATTALION branding, locked color palette, logo), common API/error-handling scaffolding, ESLint/Prettier. No feature logic yet — that starts Phase 2 (Auth+RBAC) onward.

---

**Ready to proceed to Phase 1 on your confirmation — or let me know if any role-consolidation or TBC-treatment decision above should change first.**
