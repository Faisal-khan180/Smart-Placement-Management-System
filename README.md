# Smart Placement Management System

A centralized web platform for colleges/universities to run campus placement
activities end-to-end: student registration and profiles, company-created
placement drives, an automatic eligibility engine, applications, interview
scheduling, selection, and placement analytics — for three roles: **Student**,
**Company/Recruiter**, and **TPO/Admin**.

> **Status: Phase 5 of 15 complete.** This is being built incrementally,
> phase by phase (see [Roadmap](#roadmap)). Companies can now manage
> their own profile; the TPO/admin can list companies and approve or
> revoke them. Posting actual placement drives (the eligibility engine,
> applications) is Phase 6/7 — a company must be approved here first
> before that becomes meaningful.

## Tech stack

- **Frontend:** HTML5, CSS3, JavaScript, Bootstrap 5, Fetch API, Chart.js
- **Backend:** Node.js, Express.js, RESTful APIs
- **Database:** MySQL
- **Auth/Security:** JWT, bcrypt, role-based authorization

## Project structure

```text
smart-placement-management-system/
├── backend/
│   ├── config/db.js            MySQL connection pool
│   ├── controllers/            authController.js, studentController.js,
│   │                            companyController.js
│   ├── database/
│   │   ├── schema.sql          13-table schema (Phase 2 + 4)
│   │   ├── migrate.js          runs schema.sql
│   │   └── seed.js             demo users/students/companies/drives
│   ├── routes/                 authRoutes.js, studentRoutes.js,
│   │                            companyRoutes.js
│   ├── middleware/              errorMiddleware, authMiddleware,
│   │                            asyncHandler, uploadMiddleware
│   ├── services/                (empty — filled in per phase)
│   ├── utils/                   response, validation, jwt helpers
│   ├── uploads/resumes/         uploaded resume files (Phase 4)
│   ├── app.js
│   ├── server.js
│   ├── .env.example
│   └── package.json
└── frontend/
    ├── index.html
    ├── student/ company/ admin/   (empty — filled in per phase)
    ├── css/style.css
    └── js/api.js
```

## Getting started

```bash
cd backend
npm install
cp .env.example .env      # then fill in your local MySQL credentials
npm run dev                # nodemon, restarts on file changes
# or: npm start
```

The API boots even if MySQL isn't reachable yet — it logs a connection
warning instead of crashing, so you can verify the server itself is fine
before wiring up the database (Phase 2).

Check it's running:

```bash
curl http://localhost:5000/api/health
```

```json
{
  "success": true,
  "message": "Smart Placement Management System API is running",
  "data": {}
}
```

## Database setup (Phase 2)

Once `.env` has your local MySQL credentials:

```bash
npm run db:migrate   # creates the `placement_management` DB + all 11 tables
npm run db:seed       # inserts demo users/students/companies/drives
```

Both are safe to re-run — `db:migrate` uses `CREATE TABLE IF NOT EXISTS`,
and `db:seed` checks for existing rows first and skips instead of
duplicating or erroring.

**Schema (11 tables):** `users` (shared auth for all 3 roles) →
`students` / `companies` (1:1 role profiles) → `placement_drives`
(posted by a company) → `drive_eligible_branches`, `applications`,
`interview_rounds`, `interview_results`, `selections`, plus
`student_skills` and `notifications`.

## Demo credentials

Seeded by `npm run db:seed`. Every account uses the same demo password:
`Password@123` (login itself isn't wired up until Phase 3 — for now
these just exist in the `users` table).

| Role    | Email                              |
|---------|-------------------------------------|
| Admin   | admin@spms.edu                      |
| Student | aarav.sharma@student.spms.edu       |
| Student | priya.verma@student.spms.edu        |
| Student | rohan.gupta@student.spms.edu        |
| Company | hr@technova.example.com (approved)  |
| Company | hr@buildright.example.com (pending) |

## Environment variables

See `backend/.env.example` and `frontend/.env.example` for separate local
configuration. Do not commit populated `.env` files. For Railway, set the
variables in each service's **Variables** settings; Railway does not load
local `.env` files from the repository.

```text
PORT=5000
NODE_ENV=development
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=
DB_NAME=placement_management
JWT_SECRET=
JWT_EXPIRES_IN=7d
```

## Railway deployment

Deploy this repository as two Railway services so the frontend and API can
have independent environment variables:

1. Create a MySQL service and a backend service with **Root Directory** set
  to `/backend`. Use `npm start` as the start command.
2. Add the backend variables `NODE_ENV=production`, `DB_HOST`, `DB_PORT`,
  `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `JWT_SECRET`, `JWT_EXPIRES_IN`, and
  `FRONTEND_ORIGIN`. Map the five DB variables to your Railway MySQL service
  values (`MYSQLHOST`, `MYSQLPORT`, `MYSQLUSER`, `MYSQLPASSWORD`, and
  `MYSQLDATABASE`). Set `FRONTEND_ORIGIN` to the frontend service's public
  origin, without a trailing slash. Generate a unique long random
  `JWT_SECRET`; do not reuse the example value.
3. Before first use, run `npm run db:migrate` from the backend service
  directory with its Railway variables available. This creates/updates the
  schema. Seed demo data only when desired with `npm run db:seed`.
4. Create a frontend service with **Root Directory** set to `/frontend` and
  start command `npm start`. Set `API_BASE_URL` to the backend public URL
  followed by `/api`, for example `https://your-api.up.railway.app/api`.
  The frontend server publishes this value through `/config.json`; the
  browser never reads a `.env` file directly.
5. Add a Railway Volume to the backend and mount it at the backend's
  `uploads` directory (commonly `/app/uploads`) if uploaded resumes must
  survive redeployments. Without persistent storage, Railway's container
  filesystem is temporary.

Railway supplies `PORT` for both services. The frontend configuration is
public by design and must contain only the backend URL, never credentials or
secrets. The frontend server defaults to `http://localhost:5000/api` on local hosts
and to this project's Railway API on public hosts. Set `API_BASE_URL` to your
backend URL (ending in `/api`) when deploying a different backend.

## API conventions

Every endpoint returns one of these two shapes:

```json
{ "success": true, "message": "...", "data": {} }
```

```json
{ "success": false, "message": "..." }
```

## Auth API (Phase 3)

All three routes are mounted at `/api/auth`. `register` and `login`
are public; `me` requires a Bearer token.

**Register (student):**
```bash
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Test Student", "email": "test.student@spms.edu",
        "password": "TestPass123", "role": "student",
        "roll_number": "CS2021099", "branch": "Computer Science", "batch_year": 2025
      }'
```

**Register (company):**
```bash
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Test HR", "email": "test.hr@example.com",
        "password": "TestPass123", "role": "company",
        "company_name": "Test Corp"
      }'
```

**Login** (works with the seed accounts too — see [Demo credentials](#demo-credentials)):
```bash
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "aarav.sharma@student.spms.edu", "password": "Password@123" }'
```
Copy the `token` from the response.

**Me** (protected — proves the token/middleware works end to end):
```bash
curl http://localhost:5000/api/auth/me \
  -H "Authorization: Bearer PASTE_TOKEN_HERE"
```

## Student API (Phase 4)

All routes mounted at `/api/students`. `/me/*` routes require a student
token; `/` and `/:id` require an admin token.

| Method | Path                                    | Who     | Purpose                          |
|--------|------------------------------------------|---------|-----------------------------------|
| GET    | `/api/students/me`                       | student | Own profile + skills/projects/certifications |
| PUT    | `/api/students/me`                       | student | Update branch, degree, batch_year, cgpa, backlog_count, gender, dob, address |
| POST   | `/api/students/me/skills`                | student | Add a skill (`{ "skill_name": "..." }`) |
| DELETE | `/api/students/me/skills/:skillId`       | student | Remove a skill |
| POST   | `/api/students/me/projects`              | student | Add a project (`title` required; `description`, `tech_stack`, `project_url` optional) |
| DELETE | `/api/students/me/projects/:projectId`   | student | Remove a project |
| POST   | `/api/students/me/certifications`        | student | Add a certification (`title` required; `issuing_organization`, `issue_date`, `credential_url` optional) |
| DELETE | `/api/students/me/certifications/:certificationId` | student | Remove a certification |
| POST   | `/api/students/me/resume`                | student | Upload resume — multipart form field `resume`, PDF/DOC/DOCX, max 5MB |
| GET    | `/api/students`                          | admin   | List all students, filters: `?branch=&batch_year=&min_cgpa=` |
| GET    | `/api/students/:id`                      | admin   | Full profile of one student |

Uploaded resumes are served back from `/uploads/resumes/<filename>`.

**Example — update profile:**
```bash
curl -X PUT http://localhost:5000/api/students/me \
  -H "Authorization: Bearer STUDENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "cgpa": 8.9, "address": "New address here" }'
```

**Example — add a skill:**
```bash
curl -X POST http://localhost:5000/api/students/me/skills \
  -H "Authorization: Bearer STUDENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "skill_name": "TypeScript" }'
```

## Company API (Phase 5)

All routes mounted at `/api/companies`. `/me` requires a company token;
everything else requires an admin token.

| Method | Path                              | Who     | Purpose                          |
|--------|------------------------------------|---------|-----------------------------------|
| GET    | `/api/companies/me`                | company | Own profile                       |
| PUT    | `/api/companies/me`                | company | Update company_name, industry, website, description, hr_contact_name, hr_contact_phone |
| GET    | `/api/companies`                   | admin   | List all companies, filter: `?is_approved=0` or `1` |
| GET    | `/api/companies/:id`                | admin   | Full profile of one company       |
| PATCH  | `/api/companies/:id/approve`        | admin   | Approve a pending company (idempotent) |
| PATCH  | `/api/companies/:id/revoke`         | admin   | Revoke an approved company (idempotent) |

Approving/revoking also drops a row into `notifications` for that
company's user, so Phase 11 (notifications feature) has real data to
show once it's built.

**Example — approve the pending seed company (BuildRight):**
```bash
curl -X PATCH http://localhost:5000/api/companies/2/approve \
  -H "Authorization: Bearer ADMIN_TOKEN"
```

## A note on `npm audit`

`npm audit` will show a couple of vulnerabilities that trace back to
`bcrypt`'s own native-build tooling (`node-pre-gyp` → `tar`), not to
runtime code your API executes. This is a well-known, low-risk situation
across most Node projects that use `bcrypt`; it's mentioned here rather
than silently ignored.

## Roadmap

- [x] **Phase 1** — Project setup, Express skeleton, MySQL pool, error handling
- [x] **Phase 2** — Database schema + seed data
- [x] **Phase 3** — Authentication, JWT, role-based access
- [x] **Phase 4** — Student module (profile, skills, projects, certifications, resume)
- [x] **Phase 5** — Company module
- [ ] **Phase 6** — Placement drive module + approval workflow
- [ ] **Phase 7** — Eligibility engine + skill matching
- [ ] **Phase 8** — Application module
- [ ] **Phase 9** — Interview module
- [ ] **Phase 10** — Selection + placement tracking
- [ ] **Phase 11** — Notifications + announcements
- [ ] **Phase 12** — Dashboards + analytics (Chart.js)
- [ ] **Phase 13** — Frontend polishing + responsive design
- [ ] **Phase 14** — Testing + bug fixing
- [ ] **Phase 15** — README finalization + deployment notes

