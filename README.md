# Kenya Microfinance MVP

A practical microfinance management system starter for Kenyan MFIs / Microfinance Banks.

## Features (MVP)

### Clients Module
- Client list with search (name, phone, National ID, client number) and status filter
- Register new client with full KYC fields (Kenyan counties, next of kin, etc.)
- Client detail page with edit, status management, KYC flag, and loan history
- Auto-generated client numbers (CL-0001, CL-0002, ...)
- API: GET/POST /api/clients, GET/PUT/DELETE /api/clients/[id]

### Coming next
- Loan products
- Loan application + schedule generation (Flat / Declining balance)
- Loan approval & disbursement workflow
- Repayment recording (manual + M-Pesa reference)
- Dashboard with live PAR / collections
- Authentication & roles

## Tech Stack

- Next.js 16 (App Router) + TypeScript + Tailwind CSS
- Prisma + SQLite (switch to PostgreSQL for production)
- Zod validation
- Loan calculator utility (Flat + Declining Balance)

## Quick Start

```bash
cd microfinance-mvp

# Install dependencies
npm install

# Setup database + seed sample data
npm run db:setup

# Run development server
npm run dev
```

Open http://localhost:3000

### Demo credentials (after seeding)
- Admin: admin@mfi.ke / admin123
- Loan Officer: officer@mfi.ke / officer123

### Useful commands
```bash
npm run db:studio   # Browse data in Prisma Studio
npm run db:push     # Push schema changes
npm run db:seed     # Re-seed sample data
```

## Project Structure

```
src/
  app/
    api/clients/          # Client REST API
    clients/              # Client list, new, detail pages
    loans/                # (next)
    page.tsx              # Dashboard
  lib/
    prisma.ts
    loan-calculator.ts
prisma/
  schema.prisma           # Full domain model
  seed.ts
```

## Kenya-specific notes

- National ID uniqueness check
- County dropdown (all 47 counties)
- Phone format ready for +254
- Next of kin fields (common requirement)
- Group lending support in schema
- M-Pesa reference field prepared on repayments
- Soft-delete preferred (cannot hard-delete clients with loans)

## License

Demo / educational use. Adapt freely for your institution.

## Loan Application Module

### Features
- Loan list with status filter and pagination
- New loan application: select client + product, live schedule preview
- Flat rate and Declining balance interest calculation
- Auto loan numbers: LN-2026-0001
- Status workflow: PENDING_APPROVAL → APPROVED → ACTIVE (disburse)
- Reject path
- Full repayment schedule stored at application time
- Loan detail with schedule table and client info

### API
- GET/POST /api/loans
- GET/PUT /api/loans/[id]
- GET /api/products (active products for the form)

### Pages
- /loans — list
- /loans/new — application form (supports ?clientId= for preselect)
- /loans/[id] — detail + approve / reject / disburse actions

## M-Pesa STK Push

### Setup
1. Create an app on [Safaricom Daraja](https://developer.safaricom.co.ke)
2. Copy Consumer Key & Secret into `.env`
3. Use sandbox Lipa Na M-Pesa Online shortcode `174379` and the sandbox passkey from Daraja docs
4. Expose your local server with ngrok: `ngrok http 3000`
5. Set `MPESA_CALLBACK_URL=https://xxxx.ngrok-free.app/api/mpesa/callback`

### Flow
1. Open an **ACTIVE** loan → Record payment → enter amount
2. Click **Send M-Pesa STK Push** (uses client phone by default)
3. Client enters PIN on phone
4. Daraja calls `/api/mpesa/callback` → repayment is recorded automatically
5. UI polls status and refreshes when SUCCESS

### Endpoints
- `POST /api/mpesa/stk-push` — `{ loanId, amount, phone? }`
- `GET /api/mpesa/stk-push?checkoutRequestId=...` — poll status
- `POST /api/mpesa/callback` — Safaricom callback (public)

## M-Pesa B2C Disbursement

Sends loan principal to the client’s M-Pesa wallet after approval.

### Env vars
```
MPESA_B2C_SHORTCODE=          # B2C shortcode (sandbox e.g. from Daraja)
MPESA_B2C_INITIATOR=testapi
MPESA_B2C_SECURITY_CREDENTIAL= # encrypted initiator password from Daraja
MPESA_B2C_RESULT_URL=https://your-domain/api/mpesa/b2c-result
MPESA_B2C_TIMEOUT_URL=https://your-domain/api/mpesa/b2c-timeout
```

### Flow
1. Loan **APPROVED** (or pending — B2C will auto-approve)
2. Click **Disburse via M-Pesa (B2C)** on loan detail
3. Daraja pays client → result callback marks loan **ACTIVE** + `disbursementDate`
4. UI polls until SUCCESS / FAILED / TIMEOUT

### Endpoints
- `POST /api/mpesa/b2c` — `{ loanId, amount?, phone? }`
- `GET /api/mpesa/b2c?conversationId=...` — poll status
- `POST /api/mpesa/b2c-result` — Safaricom result URL
- `POST /api/mpesa/b2c-timeout` — queue timeout URL

**Manual alternative:** **Disburse cash / manual** still sets ACTIVE without M-Pesa.

## Generate B2C SecurityCredential

```bash
# 1. Download SandboxCertificate.cer from Daraja portal → put in ./certs/
# 2. Run (use your Initiator password from Test Credentials):
npm run mpesa:credential -- "YourInitiatorPassword" ./certs/SandboxCertificate.cer

# 3. Paste output into .env:
# MPESA_B2C_SECURITY_CREDENTIAL=<long base64 string>
# MPESA_B2C_INITIATOR=testapi   # plain username, not encrypted
```

Uses RSA PKCS#1 v1.5 (required by Safaricom). Sandbox cert for sandbox, production cert for live.

## M-Pesa C2B validation (Paybill)

Lets customers pay via **Paybill** using the **loan number** as account reference.

### Env
```env
MPESA_C2B_SHORTCODE=174379
MPESA_C2B_VALIDATION_URL=https://YOUR-PUBLIC-URL/api/mpesa/c2b/validation
MPESA_C2B_CONFIRMATION_URL=https://YOUR-PUBLIC-URL/api/mpesa/c2b/confirmation
MPESA_C2B_RESPONSE_TYPE=Completed
```

### One-time register
```bash
curl -X POST http://localhost:3000/api/mpesa/c2b/register
```

### Behaviour
1. **Validation** — checks loan exists, is open, amount not wildly over outstanding → accept/reject
2. **Confirmation** — posts repayment, updates schedule, stores TransID as mpesaRef (idempotent)

### Customer flow
Paybill → Business number (shortcode) → Account = `LN-2026-0001` → Amount → PIN

## Authentication

Simple cookie session (HMAC-signed). Middleware protects pages and APIs.

- Login: `/login` — demo `admin@mfi.ke` / `admin123`
- `POST /api/auth/login` · `POST /api/auth/logout` · `GET /api/auth/me`
- Set `SESSION_SECRET` in `.env` for production

M-Pesa callbacks remain public (no session).

## Live dashboard

`GET /api/dashboard` powers the home page with real counts: clients, active loans, portfolio, PAR 30+, collections today, recent activity.

## PAR buckets & arrears aging

- **Page:** `/reports/par`
- **API:** `GET /api/reports/par`
- **Shared logic:** `src/lib/par.ts`

### PAR (cumulative)
| Metric | Meaning |
|--------|---------|
| PAR 1 | Loans with any installment overdue **> 1** day |
| PAR 30 | Overdue **> 30** days |
| PAR 60 | Overdue **> 60** days |
| PAR 90 | Overdue **> 90** days |
| PAR 180 | Overdue **> 180** days |

Numerator = full **outstanding balance** of those loans. Denominator = total portfolio outstanding (ACTIVE / DISBURSED / DEFAULTED).

### Aging (exclusive buckets)
Current → 1–30 → 31–60 → 61–90 → 91–180 → 180+ days past due (by oldest unpaid installment).

Also reports **amount in arrears** (unpaid portion of past-due installments only) vs full outstanding.

## GitHub setup

```bash
# 1. Clone or use this folder
cd microfinance-mvp

# 2. Install & compile
npm install
npm run db:setup          # prisma generate + db push + seed
npm run build             # production compile (prisma generate + next build)

# 3. Run locally
npm run dev               # http://localhost:3000
# login: admin@mfi.ke / admin123

# 4. Push to GitHub
git init
git add .
git commit -m "Initial commit: Kenya Microfinance MVP"
gh repo create kenya-microfinance-mvp --private --source=. --push
# or:
# git remote add origin https://github.com/YOUR_USER/kenya-microfinance-mvp.git
# git branch -M main
# git push -u origin main
```

**Do not commit:** `.env`, `node_modules`, `prisma/dev.db`, M-Pesa `.cer` files.  
**Do commit:** `.env.example`, source, Prisma schema, notebooks.

## Production build

```bash
npm run build
npm start
```

Set `DATABASE_URL` to PostgreSQL for production, and a strong `SESSION_SECRET`.
