# Patient Case-Taking API (server/)

Express + SQLite + JWT backend for the Patient Case-Taking Software
(SIH26047).

## Setup

```bash
cp .env.example .env
npm install
npm run dev      # auto-restarts on file changes (node --watch)
# or: npm start  # plain node, for production
```

The server listens on the port from `.env` (`PORT`, default `4000`). A
SQLite database file is created automatically at the path given by
`DB_PATH` (default `./data/app.db`) and seeded with demo data on first boot.

To force a full wipe-and-reseed on every restart (useful while developing),
set `RESEED_ON_START=true` in `.env`. To reseed once manually without
restarting the server:

```bash
npm run seed          # seeds only if tables are empty
npm run seed -- --wipe  # wipes existing data first, then reseeds
```

## Environment variables (`.env`)

| Variable | Default | Description |
|---|---|---|
| `PORT` | `4000` | Port the API listens on |
| `JWT_SECRET` | *(insecure default)* | Secret used to sign JWTs — **change this** |
| `JWT_EXPIRES_IN` | `8h` | Session token lifetime |
| `CORS_ORIGIN` | `http://localhost:3000` | Comma-separated list of allowed frontend origins |
| `DB_PATH` | `./data/app.db` | SQLite database file path |
| `RESEED_ON_START` | `false` | If `true`, wipes and reseeds the DB on every server start |

## Authentication

All endpoints except `/api/health`, `/api/auth/login`, and
`/api/auth/register` require an `Authorization: Bearer <token>` header. The
token is returned by `/api/auth/login` or `/api/auth/register`.

## API Reference

### Auth

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/login` | — | `{ identifier, password }` → `{ token, user }` |
| POST | `/api/auth/register` | — | `{ name, username, email, password, role, department? }` → `{ token, user }` |
| GET | `/api/auth/me` | ✅ | Returns the current user for the given token |
| POST | `/api/auth/logout` | ✅ | No-op (JWTs are stateless); clears client-side token |

### Users (`administrator` role required for list/create/delete)

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/users` | admin | List all users |
| POST | `/api/users` | admin | Create a user |
| PUT | `/api/users/:id` | self or admin | Update a user (only admins may change `role`) |
| DELETE | `/api/users/:id` | admin | Delete a user (cannot delete self) |

### Patients

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/patients?search=` | ✅ | List patients, optionally filtered by name/code/phone |
| GET | `/api/patients/:id` | ✅ | Get one patient |
| POST | `/api/patients` | ✅ | Create a patient |
| PUT | `/api/patients/:id` | ✅ | Update a patient |
| DELETE | `/api/patients/:id` | ✅ | Delete a patient (cascades to their cases) |

### Cases

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/cases?patient_id=` | ✅ | List cases, optionally filtered by patient |
| GET | `/api/cases/:id` | ✅ | Get one case |
| POST | `/api/cases` | ✅ | Create a case record |
| PUT | `/api/cases/:id` | ✅ | Update a case record |
| DELETE | `/api/cases/:id` | ✅ | Delete a case record |

### Admin utilities

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/admin/reset` | admin | Wipe and reseed the database with demo data |
| GET | `/api/admin/stats` | ✅ | `{ patientCount, caseCount, userCount }` |

### Health

| Method | Path | Description |
|---|---|---|
| GET | `/api/health` | `{ status: "ok", time }` — no auth required |

## Data model

Mirrors `src/types.ts` in the frontend exactly, so responses can be consumed
without any type conversion:

```ts
User        { id, username, name, email, role, department? }
Patient     { id, patient_code, name, age, gender, phone, address,
              emergency_contact, emergency_contact_phone?, blood_group?,
              created_at, updated_at }
CaseRecord  { id, patient_id, chief_complaint, present_illness, past_history,
              allergies, medications, examination_notes, vitals?,
              additional_notes, recorded_by_name, recorded_by_role,
              created_at, updated_at }
Vitals      { blood_pressure?, pulse_rate?, temperature?,
              respiratory_rate?, spo2?, weight? }
```

## Tech choices

- **better-sqlite3** — synchronous, fast, zero-config embedded database.
  Swap for Postgres/MySQL later by replacing `src/db/connection.js` and the
  `db.prepare(...)` calls in each route with your driver/ORM of choice; the
  route logic and API contracts won't need to change.
- **bcryptjs** — pure-JS bcrypt, no native build step required.
- **jsonwebtoken** — stateless session tokens.
- **zod** — request payload validation.
- **helmet / cors / express-rate-limit** — baseline security hardening.
