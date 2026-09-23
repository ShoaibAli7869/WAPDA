# WAPDA Troubleshooting Report System

A full-stack troubleshooting/workflow system for WAPDA operations teams. The repository contains:
- **Backend/**: Node.js + Express API with Passport session auth and MongoDB persistence.
- **Frontend/**: React + Vite single-page app for Shift Engineer, Department, OE, Resident Engineer, and Admin workflows.

## Table of Contents
- [1. Repository Overview](#1-repository-overview)
- [2. Roles and Departments](#2-roles-and-departments)
- [3. Architecture](#3-architecture)
- [4. Report Lifecycle (Status + Stage)](#4-report-lifecycle-status--stage)
- [5. Repository Structure](#5-repository-structure)
- [6. Technology Stack](#6-technology-stack)
- [7. Local Setup](#7-local-setup)
- [8. Environment Variables](#8-environment-variables)
- [9. Run, Build, Lint, Deploy](#9-run-build-lint-deploy)
- [10. API Reference](#10-api-reference)
- [11. Data Models](#11-data-models)
- [12. Frontend Routes, Pages, and Utilities](#12-frontend-routes-pages-and-utilities)
- [13. Security, Session, and CORS Notes](#13-security-session-and-cors-notes)
- [14. Operational Notes](#14-operational-notes)
- [15. Testing Status](#15-testing-status)
- [16. Known Inconsistencies / Maintenance Notes](#16-known-inconsistencies--maintenance-notes)

## 1. Repository Overview
This project implements a role-based troubleshooting report pipeline:
1. Shift Engineer creates reports.
2. Department users take departmental action and/or refer reports.
3. OE verifies, sends forward, or returns for revision.
4. Resident Engineer performs final close/reject/revision decisions.
5. Admin manages reports and users.

The frontend and backend are structured for separate deployment targets (frontend app + backend API), with backend Vercel config included in `Backend/vercel.json` and frontend production API defaulting to `https://wapda.vercel.app`.

## 2. Roles and Departments

### Roles (as implemented)
- `shift_engineer`
- `department` (departmental users)
- `oe`
- `resident_engineer`
- `admin`

### Department values represented in enums/lists
- `EME (P)`
- `EME (SY)`
- `P&IE`
- `MME (P)`
- `OE`
- `MME (A)`
- `XEN (EW)`
- `XEN (BARAL)`
- `SOS`
- `ITRE`
- `Admin`

## 3. Architecture

```text
[React SPA (Vite, BrowserRouter)]
          |
          | fetch(..., credentials: 'include')
          v
[Express API (Backend/server.js)]
          |
          | express-session + connect-mongo
          | Passport strategies:
          |   - engineer-local (email)
          |   - departmental-local (employeeId + department)
          v
[MongoDB via Mongoose]
   - User
   - DepartmentalUser
   - Report (+ embedded remarks)
```

Deployment notes from repo evidence:
- **Backend**: `Backend/vercel.json` builds `server.js` with `@vercel/node` and routes all paths to it.
- **Frontend**: built with Vite; API base comes from `VITE_API_URL` or defaults (`https://wapda.vercel.app` in production).

## 4. Report Lifecycle (Status + Stage)

### Status enum (`Report.status`)
- `Pending`
- `Under Review`
- `Needs Revision`
- `Closed`
- `Rejected`

### Stage enum (`Report.currentStage`)
- `Department`
- `OE Department`
- `Resident Engineer`
- `Completed`

### Typical flow (implemented behavior)
1. **Create**: Shift Engineer posts `/createReport` → `Pending` + `Department`.
2. **Department action**: `/reports/:id/department-action` typically moves report to review stage (see known duplicate route note).
3. **OE review**: `/reports/:id/oe-action`
   - `approve` → `Under Review` + `Resident Engineer`
   - `reject` (revision request) → `Needs Revision` + `Department`
   - `refer` → `Pending` + `Department` with updated `referTo`
4. **Resident Engineer action**: `/reports/:id/resident-action`
   - `close` → `Closed` + `Completed`
   - `reject` → `Rejected` + `Completed`
   - `revision` → `Needs Revision` + `Department`
5. **Resident Engineer forward**: `/reports/:id/forward-to-oe` sends report to `OE Department` when allowed.

Remarks history is appended across transitions (`/reports/:id/remarks`, `/reports/:id/oe-remark`, and action routes).

## 5. Repository Structure

```text
WAPDA/
├─ .gitignore
├─ Backend/
│  ├─ db.js
│  ├─ package.json
│  ├─ package-lock.json
│  ├─ server.js
│  ├─ vercel.json
│  ├─ models/
│  │  ├─ Users.js
│  │  ├─ Emp.js
│  │  └─ Report.js
│  ├─ routes/
│  │  └─ index.js
│  └─ scripts/
│     └─ migrateIndexes.js
└─ Frontend/
   ├─ README.md
   ├─ eslint.config.js
   ├─ index.html
   ├─ package.json
   ├─ package-lock.json
   ├─ vite.config.js
   ├─ public/
   │  └─ vite.svg
   ├─ utils/
   │  ├─ api.js
   │  ├─ checkAuth.js
   │  └─ logout.js
   └─ src/
      ├─ main.jsx
      ├─ App.jsx
      ├─ App.css
      ├─ index.css
      ├─ assets/
      │  └─ react.svg
      ├─ components/
      │  ├─ header.jsx
      │  ├─ reportCard.jsx
      │  └─ reportDetail.jsx
      ├─ extras/
      │  └─ landingpage.jsx
      ├─ pages/
      │  ├─ selectionPortal.jsx
      │  ├─ login.jsx
      │  ├─ depLogin.jsx
      │  ├─ SEDashboard.jsx
      │  ├─ departmendDashboard.jsx
      │  ├─ OEDashboard.jsx
      │  ├─ REDashboard.jsx
      │  └─ adminDashboard.jsx
      └─ stores/
         └─ useDataStore.jsx
```

## 6. Technology Stack

### Backend
| Area | Stack |
|---|---|
| Runtime | Node.js (CommonJS) |
| API | Express 5 |
| Database ODM | Mongoose 8 |
| Auth | Passport 0.7, passport-local, passport-local-mongoose |
| Session | express-session + connect-mongo |
| Utilities | cookie-parser, cors, dotenv, multer, cloudinary, morgan |
| Dev | nodemon |

### Frontend
| Area | Stack |
|---|---|
| UI | React 19 |
| Build | Vite 7 + `@vitejs/plugin-react-swc` |
| Styling | Tailwind CSS 4 via `@tailwindcss/vite` |
| Routing | react-router-dom 7 |
| State | Zustand 5 (legacy store file exists) |
| Motion/UI libs | Framer Motion, lucide-react |
| Media utility | browser-image-compression |
| Linting | ESLint 9 |

### Database/Auth/Deployment
| Area | Implementation |
|---|---|
| Database | MongoDB (via `MONGO_URL`) |
| Authentication mode | Session cookie auth (not JWT) |
| Session store | MongoDB-backed sessions |
| Backend deploy target | Vercel serverless Node (`Backend/vercel.json`) |
| Frontend deploy pattern | Separate SPA deployment (repo metadata points to `https://trouble-reporting-frontend.vercel.app`) |

## 7. Local Setup

### Prerequisites
- Node.js 18+
- npm
- MongoDB connection string

### Clone
```bash
git clone https://github.com/ShoaibAli7869/WAPDA.git
cd WAPDA
```

### Install dependencies
```bash
cd Backend
npm install

cd ../Frontend
npm install
```

## 8. Environment Variables

> Use placeholders only. Do not commit real secrets.

### Backend (`Backend/.env`)
```env
MONGO_URL=<your_mongodb_connection_string>
SESSION_SECRET=<your_strong_session_secret>
FRONTEND_URL=<frontend_origin_url>
NODE_ENV=<development_or_production>
PORT=<optional_port_default_8000>
```

### Frontend (`Frontend/.env`)
```env
VITE_API_URL=<backend_base_url>
```

Frontend API URL resolution in `Frontend/utils/api.js`:
1. `VITE_API_URL`, else
2. production default `https://wapda.vercel.app`, else
3. local default `http://localhost:8000`.

## 9. Run, Build, Lint, Deploy

### Backend
```bash
cd Backend
npm run dev     # nodemon server.js
npm start       # node server.js
```

### Frontend
```bash
cd Frontend
npm run dev
npm run build
npm run lint
npm run preview
```

## 10. API Reference

Authentication: routes marked protected require a valid session cookie (`isLoggedIn`).

### Health / utility
| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/` | No | API welcome/health JSON |
| GET | `/test-cors` | No | CORS/session debug payload |
| GET | `/checkAuth` | No | Returns `{ isLoggedIn, user? }` |
| GET | `/userData` | Yes | Returns authenticated user details |
| GET | `/users` | Yes | Combined engineer + departmental list |

### Auth
| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/engineer/register` | No | Engineer registration |
| POST | `/engineer/login` | No | Engineer login via email/password |
| POST | `/department/register` | No | Departmental registration (`employeeId + department`) |
| POST | `/department/login` | No | Departmental login using employeeId/password/department |
| GET | `/logout` | Yes | Logout + session destroy + cookie clear |

### Reports (general / department)
| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/createReport` | Yes | Create report |
| GET | `/reports` | Yes | Role-filtered listing |
| GET | `/reports/:id` | Yes | Report detail |
| PUT | `/reports/:id/department-action` | Yes | Department action submission/resubmission |
| PUT | `/reports/:id/department-refer` | Yes | Refer to other departments |
| POST | `/reports/:id/remarks` | Yes | Generic remark |
| GET | `/reports/stats/summary` | Yes | Status summary counts |

### OE / Resident Engineer
| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/reports/oe/pending` | Yes (OE role expected) | OE queue |
| PUT | `/reports/:id/oe-action` | Yes (OE role) | `approve` / `reject` / `refer` |
| POST | `/reports/:id/oe-remark` | Yes (OE role) | OE-only remarks |
| PUT | `/reports/:id/resident-action` | Yes (resident_engineer role) | `close` / `reject` / `revision` |
| PUT | `/reports/:id/forward-to-oe` | Yes (resident_engineer role) | Forward back to OE |

### Admin
| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/admin/users` | Yes + admin | List users |
| POST | `/admin/users` | Yes + admin | Create engineer/departmental user |
| DELETE | `/admin/users/:id` | Yes + admin | Delete user |
| PUT | `/admin/users/:id/status` | Yes + admin | Toggle active/disabled |
| GET | `/admin/reports` | Yes + admin | Full report list |
| PUT | `/admin/reports/:id` | Yes + admin | Update report fields |
| DELETE | `/admin/reports/:id` | Yes + admin | Delete report |

## 11. Data Models

### `User` (`Backend/models/Users.js`)
- Core fields: `name`, unique `email`, `department`, `role`, `status`, `phoneNumber`
- Role enum: `shift_engineer`, `oe`, `resident_engineer`, `admin`
- Passport plugin with `usernameField: "email"`
- Timestamps enabled

### `DepartmentalUser` (`Backend/models/Emp.js`)
- Core fields: `employeeId` (uppercased), `name`, optional `email`, `department`, `role`, `status`, `phoneNumber`
- Compound unique index: `{ employeeId: 1, department: 1 }`
- Passport plugin with `usernameField: "employeeId"`, `usernameUnique: false`
- Timestamps enabled

### `Report` (`Backend/models/Report.js`)
- Identity/content: `serialNo` (unique uppercase), `date`, `time`, `apparatus`, `description`
- Action fields: `recommendation`, `operationAction`, `departmentAction`
- Routing/context: `notifiedBy`, `referTo` (array of department enum values), `means`
- Workflow fields: `status`, `currentStage`
- Ownership: `createdBy` (ref `User`)
- Tracking: `remarks[]`, `priority`, `estimatedCompletionDate`, `actualCompletionDate`
- Timestamps enabled

### Embedded remark shape
- `user` (string)
- `text` (string)
- `timestamp` (string)

## 12. Frontend Routes, Pages, and Utilities

### SPA routes (`Frontend/src/App.jsx`)
- `/` → role selection portal
- `/landing` → alternate landing page component
- `/login` → engineer login/register
- `/depLogin` → departmental login/register
- `/adminDashboard`
- `/shiftDashboard`
- `/depDashboard`
- `/oeDashboard`
- `/reDashboard`

### Page responsibilities
- `selectionPortal.jsx`: entry role-selection UI/navigation.
- `landingpage.jsx`: separate promotional-style landing page component.
- `login.jsx`: engineer auth flow; role-based navigation after login.
- `depLogin.jsx`: departmental auth flow using employeeId + department.
- `SEDashboard.jsx`: report creation (`TR-YYYY-NNN` serial generation), list/filter/search, submit to `/createReport`.
- `departmendDashboard.jsx`: department queue; loads `/userData` + `/reports`, 30s polling, status/search filtering, remarks, department action/resubmission, referral actions, browser audio notifications.
- `OEDashboard.jsx`: OE pending queue (`/reports/oe/pending`), 30s polling, muteable audio notifications, remarks, approve/revision/refer actions via `/reports/:id/oe-action`.
- `REDashboard.jsx`: resident engineer view over `/reports`, category filters (pending review, ready for OE, at OE, closed, rejected), 30s polling, remarks, forward-to-OE and final resident actions.
- `adminDashboard.jsx`: report + user administration (`/admin/*`), filtering/search, edit/delete reports, add remarks, create/delete/enable/disable users.

### Components/utilities
- `components/header.jsx`: shared header UI.
- `components/reportCard.jsx`: report list card UI.
- `components/reportDetail.jsx`: detailed report modal/panel UI.
- `utils/api.js`: API base URL resolution + authenticated fetch wrapper.
- `utils/checkAuth.js`: calls `/checkAuth`.
- `utils/logout.js`: calls `/logout`, then verifies auth/check and redirects.
- `stores/useDataStore.jsx`: local/mock initial report state; not primary path for main server-backed dashboards.

## 13. Security, Session, and CORS Notes
- Session auth is cookie-based (`express-session`) with Mongo-backed session storage.
- Cookie name: `wapda_session`.
- Cookie flags include `httpOnly`, `secure` in production, and `sameSite` (`none` in production, `lax` otherwise).
- CORS is configured with explicit allowed origins and `credentials: true`.
- Server trusts proxy (`app.set("trust proxy", 1)`) for deployment behind proxy/CDN.

## 14. Operational Notes
- Backend entrypoint is `Backend/server.js`.
- Server listens on `PORT` (default `8000`) **only when not in production**; production usage expects platform handler export (`module.exports = app`).
- Index migration script exists: `Backend/scripts/migrateIndexes.js` for departmental user compound index migration.
- Dashboards use periodic polling (typically 30 seconds) and browser-side audio notifications; no WebSocket implementation is present.

## 15. Testing Status
No test script or dedicated test suite is defined in the inspected `Backend/package.json` or `Frontend/package.json`. This repository currently relies on manual/runtime verification rather than automated tests.

## 16. Known Inconsistencies / Maintenance Notes
1. **Duplicate backend route definitions in `Backend/routes/index.js`:**
   - `PUT /reports/:id/department-action` appears more than once.
   - `PUT /reports/:id/forward-to-oe` appears more than once.
   Express matches routes in registration order, so earlier handlers can shadow later logic.

2. **Frontend route naming mismatch:**
   - In `login.jsx`, departmental role mapping points to `/departmentDashboard`.
   - In `App.jsx`, implemented route is `/depDashboard`.

3. **Legacy/alternate frontend state/content paths exist:**
   - `stores/useDataStore.jsx` contains local mock data and is not the main dashboard data source.
   - `extras/landingpage.jsx` appears unrelated to troubleshooting workflow and coexists with operational portal pages.

4. **Enum/string consistency caveats:**
   - Department strings vary slightly across files (spacing/format variants), so centralizing shared constants would reduce mismatch risk.

