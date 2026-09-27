# SMARTED BATTALION — Prototype

> **Status: Phase 4 (Quartermaster & Equipment module) complete.** Gate & Visitor Security and Duty & Leave are not built yet — see [Build Phases](#build-phases-status) below.

## 1. Project Overview

SMARTED BATTALION is a functional prototype of a battalion administration system covering four modules — **Smart Mess & Ration**, **Quartermaster & Equipment**, **Battalion Gate & Visitor Security**, and **Duty Roster & Leave Management** — on one shared platform (single login, one RBAC model, one audit trail, one reporting layer).

This is a **prototype for functional demonstration and validation**, built to prove that the workflows documented in DOC-01 through DOC-05 work end-to-end. It is explicitly **not** a production deployment — see [Prototype vs Production](#18-prototype-vs-production-distinction).

## 2. Architecture

```
React (client)
  ↓ TanStack Query
Express REST API (server)
  ↓ Controllers → Services → Repositories
MongoDB (Mongoose)
```

Modular monolith — one backend process, one frontend app, organized into module folders (`mess`, `quartermaster`, `gate`, `duty`) plus shared platform services (`auth`, `users`, `roles`, `audit`, `reports`). No microservices, no GraphQL, no Kubernetes.

The frontend never talks to MongoDB directly — only the backend does.

## 3. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React, TypeScript, Vite, Tailwind CSS, React Router, TanStack Query, React Hook Form, Zod, Lucide React |
| Backend | Node.js, Express, TypeScript, REST |
| Database | MongoDB, Mongoose |

## 4. Prerequisites

- Node.js 18+ and npm
- MongoDB running locally (Community Edition) — see [Local MongoDB Setup](#5-local-mongodb-setup)
- No internet connection is required to run the app day-to-day (see [Offline Operation](#15-offline-operation))

## 5. Local MongoDB Setup

Install MongoDB Community Edition for your OS, then start it locally, e.g.:

```bash
# macOS (Homebrew)
brew tap mongodb/brew
brew install mongodb-community
brew services start mongodb-community

# Linux (systemd-based)
sudo systemctl start mongod

# Windows
# Install via the MongoDB installer, then it typically runs as a service automatically.
```

Confirm it's running on the default port:

```bash
mongosh --eval "db.runCommand({ ping: 1 })"
```

No cloud database (Atlas or otherwise) is used or required.

## 6. Backend Setup

```bash
cd server
cp .env.example .env    # adjust values if needed — defaults work for local dev
npm install
npm run dev
```

The API starts on `http://localhost:5000`. Confirm it's up:

```bash
curl http://localhost:5000/api/health
```

## 7. Frontend Setup

```bash
cd client
cp .env.example .env    # optional — defaults work for local dev
npm install
npm run dev
```

The app starts on `http://localhost:5173` and proxies `/api` calls to the backend automatically (see `client/vite.config.ts`).

> From the repo root, `npm run install:all` installs both, and `npm run dev:server` / `npm run dev:client` run each (in two terminals).

## 8. Environment Variables

**server/.env**

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | API port | `5000` |
| `NODE_ENV` | `development` \| `production` \| `test` | `development` |
| `MONGODB_URI` | Local MongoDB connection string | `mongodb://127.0.0.1:27017/smarted_battalion` |
| `JWT_SECRET` | Prototype auth signing secret — **replace before any non-local use** | *(placeholder)* |
| `JWT_EXPIRES_IN` | Session token lifetime | `8h` |
| `CLIENT_ORIGIN` | Allowed CORS origin | `http://localhost:5173` |

**client/.env**

| Variable | Purpose | Default |
|---|---|---|
| `VITE_API_BASE_URL` | API base path | `/api` |

No secrets are hardcoded anywhere in the codebase — everything above is read from environment variables.

## 9. Seed Command

```bash
npm run seed --prefix server
```

Populates:
- All 10 prototype roles with their permission sets (`server/src/common/permissions.ts`)
- One fictional demo user per role, plus 5 additional PERSONNEL accounts for demo richness (see [Demo Credentials](#10-demo-credentials))
- Mess module inventory items with real opening-stock transactions, plus 5 days of realistic meal-attendance history (exercises real auto-deduction and low-stock alerting)
- Quartermaster module: bulk and serialized equipment items, a couple of real issue transactions, one overdue and one upcoming maintenance schedule entry, and one pending damage report

Safe to re-run — existing records are updated or skipped, not duplicated.

> Full demo data for Gate & Visitor Security and Duty & Leave is added to this same script as each module is built (Phases 5–6).

## 10. Demo Credentials

All demo accounts share the same password: **`Demo@123`**

| Service No. | Role | Name |
|---|---|---|
| `100001` | ADMIN | Arvind Rathore |
| `100002` | COMMANDANT | Vikram Sehgal |
| `100003` | ADJUTANT | Rohan Malhotra |
| `100004` | MESS_SECRETARY | Karan Bisht |
| `100005` | STOREKEEPER | Suresh Thapa |
| `100006` | QUARTERMASTER | Deepak Rawat |
| `100007` | GATE_GUARD | Manoj Kumar |
| `100008` | SECURITY_OFFICER | Ajay Negi |
| `100009` | ROSTER_INCHARGE | Ramesh Chauhan |
| `100010`–`100015` | PERSONNEL | Vivek Joshi, Anil Kumar, Deepika Rana, Prakash Gurung, Sunil Bhandari, Meena Tamang |

All fictional — no real personnel data, per the branding lock.

## 11. Running the Application

Once both servers are running (§6, §7) and you've seeded the database (§9):

1. Open `http://localhost:5173` — you'll land on the SMARTED BATTALION login screen.
2. Log in with any demo account from §10.
3. You'll be redirected to `/dashboard`, showing real Mess and QM metrics (today's meal count, low-stock item count, overdue inspection count) for any role with the matching permission.
4. Open **Smart Mess & Ration** in the sidebar:
   - **My Mess** — log today's meal (if your role has `LOG_MEAL_ATTENDANCE`, e.g. `100010`), see your own attendance history and bills
   - **Inventory** — current stock levels, low-stock flags, add new items (`MANAGE_MESS_STOCK` roles, e.g. `100005`)
   - **Stock Ledger** — full transaction history, record new IN/OUT transactions
   - **Billing** — generate monthly bills for all personnel at once, update payment status (`MANAGE_MESS_BILLING` roles, e.g. `100004`)
   - **Reports** — low-stock, attendance, and monthly-bill summary views
5. Open **Quartermaster & Equipment** in the sidebar:
   - **My Equipment** — see equipment currently issued to you, report damage (`REPORT_QM_DAMAGE` roles, e.g. `100011`–`100015`)
   - **Equipment** — full catalog (bulk and serialized), issue/return actions (`ISSUE_QM_ITEM` roles, e.g. `100006`)
   - **Maintenance** — inspection schedules, overdue flagging, recording outcomes (`MANAGE_QM_MAINTENANCE` roles)
   - **Damage & Scrap** — review reported damage, record Board-of-Officers decisions and finalize write-offs (`MANAGE_QM_DAMAGE_SCRAP` roles)
   - **Reports** — maintenance-due and damage/scrap audit views
6. Log in as `100010` (PERSONNEL) in a different browser/incognito window — notice the sidebar and each module's sub-nav show far fewer options, and navigating directly to `/mess/billing` or `/quartermaster/damage-scrap` by URL redirects you away rather than showing a broken page.

## 12. Testing

Backend unit tests cover the critical behavior built so far:
- **Auth**: login (success, wrong password, unknown user, inactive account), JWT issuance/verification
- **RBAC**: permission enforcement (granted, denied + audit event, unauthenticated), role/permission definition sanity
- **Mess**: billing grand-total calculation, the manual-stock-entry-blocks-vs-auto-deduction-never-blocks asymmetry, duplicate meal-attendance prevention
- **Quartermaster**: bulk vs. serialized issue/return handling (over-issue blocking, already-issued blocking, return-exceeds-balance blocking, written-off items refusing issue), Board-decision-before-write-off gating, double-finalization prevention, and that a bulk-item write-off correctly does *not* touch `availableStock` (a documented schema gap, not a bug)

**36 tests, all passing.**

```bash
npm test --prefix server
```

These tests mock the database layer deliberately, so they run without a live MongoDB connection — useful for CI or a quick sanity check before starting the real server.

### Manual end-to-end demo checklist (Phase 4 scope)

- [ ] `npm run seed --prefix server` completes and logs demo credentials
- [ ] Log in as `100006` (QUARTERMASTER), open **Equipment** — confirm bulk items (Combat Boots, Uniforms...) and serialized items (rifles, radios with serial numbers) both appear, with correct status badges
- [ ] Issue a serialized item that's already issued (per seed data) to a different person — confirm it's rejected with a clear "already issued to..." message
- [ ] Issue a bulk item in a quantity larger than available stock — confirm it's rejected
- [ ] Return an item and confirm the balance updates immediately on both the QM's Equipment view and the individual's **My Equipment** view
- [ ] Open **Maintenance** — confirm one entry shows "Overdue" and one shows "Upcoming" (from seed data); record an inspection outcome and confirm the due date updates
- [ ] Open **Damage & Scrap** — confirm the seeded pending report appears; record a Board decision of "Approved Write-Off" on a *serialized* item and confirm it disappears from the issuable list in **Equipment** (status becomes "Written Off")
- [ ] Attempt to record a second decision on the same now-resolved entry — confirm it's rejected
- [ ] Log in as `100013` (PERSONNEL), open **My Equipment** — confirm you can report damage but cannot see the Board-decision action
- [ ] `GET /api/audit?module=QM` (via curl with an ADMIN token) shows `ISSUE_ITEM`, `RETURN_ITEM`, and `RECORD_DAMAGE_DECISION` events from the steps above

## 13. Build

```bash
npm run build   # from repo root — builds both server and client
```

Backend compiles to `server/dist` (test files are excluded from this build — see `server/tsconfig.build.json`); frontend builds to `client/dist` (static files, servable by any static file server or wrapped later — see §17).

## 14. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Backend fails to start with an env validation error | Missing/invalid `server/.env` — copy from `.env.example` and check `JWT_SECRET` length |
| `ECONNREFUSED` connecting to MongoDB | MongoDB isn't running locally — see §5 |
| Frontend shows network errors calling `/api/*` | Backend isn't running, or is on a different port than `vite.config.ts` expects |
| CORS errors in the browser console | `CLIENT_ORIGIN` in `server/.env` doesn't match the URL the frontend is actually served from |

## 15. Offline Operation

Once dependencies are installed (`npm install`, which requires internet the first time) and MongoDB is running locally, the application requires **no internet connection** for normal operation. The frontend, backend, and database all run on `localhost`. This matches DOC-01 §4's air-gapped/local-LAN architecture requirement.

## 16. Future LAN Deployment

The current prototype runs frontend + backend + database on one machine. DOC-01 §4 describes a production direction where a **central server** hosts the application and database, reached by client terminals over the Battalion's **local LAN** — no code redesign is required for this; it is a deployment/infrastructure change (server placement, network configuration, multi-client access), not an application architecture change. This prototype's layered structure (frontend never touches the DB directly; only the backend does) is what makes that transition possible later without a rewrite.

**Not implemented in this prototype:** real network/VLAN configuration, Active Directory integration, multi-terminal hardware setup. See DOC-01 §4 and §20 for what remains `[TO BE CONFIRMED]` before a real LAN deployment.

## 17. Future EXE Packaging

An EXE is **application packaging only** — it does not replace the central database/server needed for a multi-user LAN deployment. The codebase is structured (clear frontend/backend separation, environment-based configuration) so that a desktop wrapper (e.g. Electron) could later package the frontend + a bundled backend for single-machine offline use. This has not been implemented in the current prototype and is explicitly P2 (not required) per the 4-day priority list.

## 18. Prototype vs Production Distinction

This prototype demonstrates that the documented workflows (DOC-01 through DOC-05) function correctly. It does **not** by itself satisfy production requirements:

- Authentication uses a placeholder JWT secret — must be replaced and hardened for any real deployment.
- Sessions are stateless JWTs with no server-side revocation list — logging out clears the client's token but a captured token remains valid until it expires. A production system would need a revocation/refresh-token strategy.
- No formal security validation, penetration testing, or encryption-at-rest has been performed (DOC-01 §12 leaves exact encryption scope `[TO BE CONFIRMED]`).
- No backup/recovery infrastructure is implemented (DOC-01 §15).
- No real LAN/network/AD configuration exists (§16 above).
- Several business rules across all four modules are seed-configurable **placeholders** where the source documents left them `[TO BE CONFIRMED]` (e.g. Mess ration-scale formula, QM inspection intervals, Duty leave approval chain) — see `PHASE_0_IMPLEMENTATION_AUDIT.md` §0.5 for the full register. These are clearly commented in code and must be replaced with Battalion-confirmed values before production use.
- All demo accounts (§10) share one publicly-documented password — obviously never acceptable outside a local prototype.

---

## Build Phases — Status

| Phase | Scope | Status |
|---|---|---|
| 0 | Implementation audit | ✅ Complete — see `PHASE_0_IMPLEMENTATION_AUDIT.md` |
| 1 | Foundation | ✅ Complete |
| 2 | Auth + RBAC | ✅ Complete |
| 3 | Mess module | ✅ Complete |
| 4 | Quartermaster module (this delivery) | ✅ Complete |
| 5 | Gate module | ⏳ Next |
| 6 | Duty module | Not started |
| 7 | Integration (audit, reports, seed data) | Not started |
| 8 | Finalization (testing, polish, docs) | Not started |

### What Phase 4 delivered

- **Full Quartermaster & Equipment module** (DOC-03) end to end: equipment catalog (both bulk and individually-serialized items), issue/return with required digital signature, personnel-wise balance, inspection/maintenance scheduling, and damage/scrap reporting through to Board-of-Officers decision and write-off finalization
- **A genuine architectural decision, not just an implementation detail**: DOC-03's schema left serialized-vs-bulk item tracking unresolved (flagged in `PHASE_0_IMPLEMENTATION_AUDIT.md` from day one). This phase resolves it: a serialized item's "current status" (in store / issued / to whom) is *derived* from its latest issue/return ledger entry rather than stored redundantly, while bulk items keep a real `availableStock` counter — documented at the point of definition in `equipmentItem.model.ts`, not left implicit
- **A new permission added mid-project, not silently**: `REPORT_QM_DAMAGE`, because DOC-03 explicitly names the Individual as able to report damage (report-only), distinct from the QM/Admin-only full review-and-write-off lifecycle (`MANAGE_QM_DAMAGE_SCRAP`) — the addition and its rationale are documented in `common/permissions.ts` itself
- **A documented schema gap, not a silently-wrong feature**: a bulk item's write-off is recorded in the audit trail but does not auto-adjust `availableStock`, since DOC-03's own schema has no write-off-quantity field or third transaction type for this — noted in code and in the test suite rather than quietly guessed at
- **12 new backend unit tests** covering the bulk/serialized asymmetry and the board-decision/write-off gating — **36 total, all passing**
- **Frontend**: five new screens (My Equipment, Equipment, Maintenance, Damage & Scrap, Reports) reusing every shared component built in Phase 3 — no new reusable components were needed, which is itself a signal the Phase 3 component set was built at the right level of generality
- Dashboard now also shows a real QM metric (overdue inspection count) for any role with `VIEW_QM`

### What Phase 4 deliberately did not build

Automatic procurement-request generation on low bulk stock (explicitly listed as a DOC-03 future enhancement, not current scope), barcode/QR scanning for issue/return (same), and a dedicated Board-of-Officers login (DOC-03's own TBC register leaves this open; the prototype's documented decision is that QM/Admin records the Board's decision on its behalf — see `PHASE_0_IMPLEMENTATION_AUDIT.md` §0.1).
