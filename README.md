# MediPay — Hospital Billing SaaS

MediPay is a multi-tenant healthcare finance and operations portal built with the MERN stack. It gives hospitals and clinics one role-based workspace for staff administration, patient encounters, departmental charges, invoices, payments, documents, appointments, reports, contracts, claims, notifications, and audit history.

The main billing workflow is:

```text
Patient → Encounter → Department service charge → Invoice → Payment → Receipt / Refund / Reconciliation
```

## Repository and deployment

- Source repository: [github.com/Tousifrahmananto/MediPay](https://github.com/Tousifrahmananto/MediPay)
- Frontend target: Vercel, with `client` as the root directory
- Backend target: Render, configured by [`render.yaml`](render.yaml)
- Database target: MongoDB Atlas

Deployment URLs are environment-specific. Set the current frontend origin in Render's `CLIENT_ORIGIN` and the current Render API URL in Vercel's `VITE_API_URL`. A free Render service may take several seconds to wake after inactivity.

## Core features

### Authentication and authorization

- Organization registration and owner-account creation
- Email/password login with bcrypt password hashes and JWT sessions
- Role- and permission-based API access
- Temporary-password change flow for newly created staff
- Forgot-password and single-use password-reset links through Resend
- Login and reset rate limiting

### Organization, departments, and staff

- Organization profile and approval/status management
- Department and billable-service catalogues
- Staff creation, department assignment, role selection, permissions, and account status
- Department-scoped access for operational staff
- Centralized administration view across organizations

### Patients and clinical billing

- Patient registration and duplicate-patient checks
- OPD, emergency, and inpatient encounters
- Department-level service charge capture
- Server-calculated quantity, VAT, and totals using decimal arithmetic
- Duplicate open-encounter prevention

### Invoices and payments

- Guided invoice builder with configurable services, quantities, and hospital-stay days
- Consolidated invoices generated from uninvoiced encounter charges
- Patient name, email, and phone snapshots on each invoice
- PDF invoices with a public **Pay Now** link
- Invoice delivery through Resend with `NOT_SENT`, `SENT`, or `FAILED` status tracking
- Cash and bank payment recording
- SSLCOMMERZ hosted checkout in academic sandbox mode
- Optional Bangla QR entry when enabled for the merchant account
- Transaction validation, partial payments, PDF receipts, refunds, and reconciliation

### Operations portal

- Date-filtered dashboard with monthly billing, collections, outstanding amounts, charts, and operational summaries
- Document upload, download, rename, and deletion
- Appointment availability, calendar, booking, rescheduling, cancellation, decisions, and email reminders
- Support conversations and notifications
- Contract upload, review, electronic signature, and signed PDF generation
- Report attachments, search, filters, and downloads
- Claims workflow and CSV export
- Audit-log filtering and CSV export

## Roles

| Role | Main responsibility |
| --- | --- |
| `PROVIDER_OWNER` | Owns the organization and manages its configuration and staff |
| `ADMIN` | Manages organization-wide operational and administrative workflows |
| `SUPER_ADMIN` | Uses the centralized console to review organizations and platform resources |
| `PROVIDER_STAFF` / `DEPARTMENT_STAFF` | Works within an assigned department and granted permissions |
| `DOCTOR` | Uses permitted patient, encounter, appointment, and document workflows |
| `BILLING_ADMIN` | Creates invoices and supervises billing and payment activity |
| `CASHIER` | Records authorized payments at the billing counter |

The frontend hides unavailable sections, but authorization is enforced again by backend middleware on every protected API route.

## Technology stack

| Layer | Technology |
| --- | --- |
| Frontend | React 19, Vite 7, TypeScript |
| Backend | Node.js 22+, Express 5, TypeScript |
| Database | MongoDB Atlas or MongoDB 8, Mongoose |
| Authentication | JWT, bcryptjs |
| Validation | Zod and Mongoose schema validation |
| Money calculations | decimal.js and MongoDB `Decimal128` |
| Email | Resend HTTP API |
| Documents | PDFKit and MongoDB-backed file storage |
| Payments | SSLCOMMERZ sandbox |
| Deployment | Vercel frontend, Render backend, MongoDB Atlas |
| Testing | Vitest, React Testing Library, Supertest |

## Architecture and data integrity

The React client calls the Express REST API. The API validates each request, authenticates its JWT, checks roles or permissions, applies the authenticated `hospitalId` tenant scope, and then reads or writes through Mongoose.

Important safeguards include:

- Hospital-scoped records for multi-tenant data separation
- Compound unique indexes for department codes, service codes, employee numbers, patient numbers, encounter numbers, invoice numbers, and claim numbers
- Atomic per-hospital counters for readable identifiers
- Immutable invoice item and recipient snapshots
- MongoDB transactions when consolidating charges into an invoice and when settling a payment
- Idempotent gateway settlement and invoice-email requests
- `Decimal128` storage for financial values
- Audit records containing the actor, target, before/after state, correlation ID, IP address, and user agent
- Hashed, expiring, single-use password-reset tokens

## Project structure

```text
client/                     React/Vite portal
  src/App.tsx               Authentication and registration flows
  src/PortalDashboard.tsx   Role-based portal modules
server/
  src/config/               Environment and MongoDB configuration
  src/lib/                  Email, PDF, payment, audit, and error helpers
  src/middleware/           Authentication, authorization, and request context
  src/models/               Mongoose schemas and indexes
  src/routes/               REST endpoints and business workflows
  src/scripts/seed.ts       Idempotent local demo data
documentation/              Project documentation
compose.yaml                Local MongoDB replica set
render.yaml                 Render web-service Blueprint
```

Design artifacts are also included:

- [`Hospital_Billing_ERD.drawio`](Hospital_Billing_ERD.drawio)
- [`Hospital_Billing_ERD.pdf`](Hospital_Billing_ERD.pdf)
- [`Hospital_Billing_Wireframes.html`](Hospital_Billing_Wireframes.html)
- [`Hospital_Billing_Wireframes_Preview.png`](Hospital_Billing_Wireframes_Preview.png)
- [`Project_Execution_Architecture.drawio`](Project_Execution_Architecture.drawio)
- [`PROJECT_FEATURE_AUDIT.md`](PROJECT_FEATURE_AUDIT.md)

## Local setup

### Requirements

- Node.js 22 or newer
- npm
- Docker Desktop for the supplied local MongoDB replica set, or a MongoDB Atlas connection string

### 1. Install dependencies

```powershell
npm.cmd install
```

### 2. Configure the backend

```powershell
Copy-Item server/.env.example server/.env
```

At minimum, set a private development JWT secret and a demo password of at least 12 characters:

```env
JWT_SECRET=replace-with-at-least-32-random-characters
DEMO_USER_PASSWORD=replace-with-a-local-password-of-at-least-12-characters
```

The checked-in example uses the local replica-set connection:

```env
MONGODB_URI=mongodb://127.0.0.1:27017/hospital_billing?replicaSet=rs0
```

To use Atlas instead, replace `MONGODB_URI` with the Atlas driver connection string and allow the development machine's IP in Atlas Network Access.

### 3. Start MongoDB

Skip this step when using Atlas.

```powershell
docker compose up -d
```

`compose.yaml` starts MongoDB 8 and initializes the `rs0` replica set required by the application's transaction-based financial workflows.

### 4. Seed optional demo data

```powershell
npm.cmd run seed --workspace server
```

The seed is disabled in production and is safe to run repeatedly. It creates or updates:

- `admin@shurokkha.test` — demo super administrator
- `opd@shurokkha.test` — department-scoped OPD staff
- Administration, OPD, Laboratory, and Cash Counter departments
- Sample OPD and laboratory services

Both demo users receive the password configured in `DEMO_USER_PASSWORD`.

### 5. Run the application

Start the API and client in separate terminals:

```powershell
npm.cmd run dev:server
```

```powershell
npm.cmd run dev:client
```

Open [http://localhost:5173](http://localhost:5173). Vite proxies local `/api` requests to the backend at `http://localhost:4000`.

## Environment variables

### Backend (`server/.env` or Render)

| Variable | Required | Purpose |
| --- | --- | --- |
| `NODE_ENV` | Production | `development`, `test`, or `production` |
| `PORT` | No | API port; defaults to `4000` locally and Render supplies its own port |
| `MONGODB_URI` | Production | MongoDB connection string |
| `DNS_SERVERS` | No | DNS override for environments that cannot resolve Atlas SRV records |
| `JWT_SECRET` | Production | JWT signing secret; minimum 32 characters |
| `CLIENT_ORIGIN` | Yes | Exact frontend origin used by CORS and reset links |
| `SERVER_URL` | Yes | Public backend URL used for payment callbacks and invoice links |
| `RESEND_API_KEY` | Email features | Resend API key |
| `PASSWORD_RESET_FROM` | Password reset | Verified sender in `Name <email@domain>` format |
| `INVOICE_FROM` | Invoice/reminder email | Verified sender; falls back to `PASSWORD_RESET_FROM` |
| `PASSWORD_RESET_TTL_MINUTES` | No | Reset-link lifetime; defaults to `20` |
| `SSLCOMMERZ_STORE_ID` | Online payments | Sandbox or live merchant store ID |
| `SSLCOMMERZ_STORE_PASSWORD` | Online payments | Sandbox or live merchant password |
| `SSLCOMMERZ_SANDBOX` | No | Keep `true` for this academic deployment |
| `DEMO_USER_PASSWORD` | Local seed only | Password assigned to local demo users |

Example production values without secrets:

```env
NODE_ENV=production
MONGODB_URI=mongodb+srv://...
JWT_SECRET=replace-with-a-random-secret-of-at-least-32-characters
CLIENT_ORIGIN=https://your-vercel-project.vercel.app
SERVER_URL=https://your-render-service.onrender.com
RESEND_API_KEY=re_...
PASSWORD_RESET_FROM="MediPay <support@your-verified-domain.com>"
INVOICE_FROM="MediPay Billing <billing@your-verified-domain.com>"
PASSWORD_RESET_TTL_MINUTES=20
SSLCOMMERZ_STORE_ID=your-sandbox-store-id
SSLCOMMERZ_STORE_PASSWORD=your-sandbox-store-password
SSLCOMMERZ_SANDBOX=true
```

Resend's default `onboarding@resend.dev` sender can only send test messages to the Resend account owner's address. Verify a domain and use a sender on that domain to deliver invoices, reminders, and password resets to arbitrary patient or staff email addresses.

### Frontend (`client/.env` or Vercel)

```env
VITE_API_URL=https://your-render-service.onrender.com
```

Leave `VITE_API_URL` blank for local development. Never place `MONGODB_URI`, `JWT_SECRET`, `RESEND_API_KEY`, or payment credentials in Vercel frontend variables.

## Invoice email and payment flow

1. Billing staff creates an invoice or consolidates posted encounter charges.
2. The backend validates the recipient and calculates all totals.
3. The invoice is saved before delivery is attempted.
4. PDFKit creates an itemized A4 invoice with the backend's public checkout link.
5. Resend receives the email with the PDF attached.
6. MongoDB records `emailDeliveryStatus`, `emailSentAt`, the Resend message ID, or the delivery error.
7. The payer follows the PDF link to SSLCOMMERZ hosted checkout.
8. The backend validates the gateway response before settling the payment and invoice together in a transaction.

An email failure does not remove the invoice. Updating an unpaid or overdue invoice's recipient information triggers another delivery attempt.

## Available commands

| Command | Purpose |
| --- | --- |
| `npm.cmd run dev:server` | Start the Express API with TypeScript watch mode |
| `npm.cmd run dev:client` | Start the Vite development server |
| `npm.cmd run seed --workspace server` | Create/update local demo data |
| `npm.cmd run typecheck` | Type-check both workspaces |
| `npm.cmd test` | Run client and server tests |
| `npm.cmd run build` | Create production builds for both workspaces |
| `npm.cmd start --workspace server` | Run the compiled API |

## API overview

All protected endpoints use `Authorization: Bearer <JWT>`.

| Area | Base route |
| --- | --- |
| Health | `GET /api/health` |
| Authentication | `/api/auth` |
| Dashboard | `/api/dashboard` |
| Organization and administration | `/api/organizations` |
| Staff and audit | `/api/staff`, `/api/audit` |
| Patients and encounters | `/api/patients`, `/api/encounters` |
| Departments and services | `/api/catalog` |
| Charges and invoices | `/api/charges`, `/api/invoices` |
| Payments, refunds, reconciliation | `/api/payments`, `/api/refunds`, `/api/reconciliation` |
| Documents and reports | `/api/documents`, `/api/reports` |
| Appointments and messages | `/api/appointments`, `/api/messages` |
| Contracts and claims | `/api/contracts`, `/api/claims` |
| Notifications | `/api/notifications` |

SSLCOMMERZ callbacks are generated from `SERVER_URL`. The IPN endpoint is:

```text
POST /api/payments/sslcommerz/ipn
```

## Quality checks

Run the complete verification suite before deployment:

```powershell
npm.cmd run typecheck
npm.cmd test
npm.cmd run build
```

## Academic scope and limitations

- SSLCOMMERZ is configured for academic sandbox use; production merchant onboarding is outside this project.
- Bangla QR is available only if the configured merchant account returns a QR checkout URL.
- Uploaded files are stored in MongoDB and limited to 5 MB; production-scale deployments should use object storage.
- Appointment reminders are sent explicitly from the portal rather than by an always-running background scheduler.
- Invoice email sending occurs in the API request; a production-scale version should use a background queue, retries, and delivery webhooks.
- Optional MFA is represented in the data model but is not implemented as a user workflow.

## Security notes

- Do not commit `.env` files, database credentials, Resend keys, JWT secrets, or payment credentials.
- Rotate any credential that has been pasted into a public repository, screenshot, chat, or deployment log.
- Use verified Resend domains and separate sandbox/live payment credentials.
- Keep MongoDB Atlas Network Access and database-user permissions as restrictive as practical.

## Author

**Tousif Rahman Anto**

CSE391 — Summer 2026, BRAC University
