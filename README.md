# RFID Registration System

RFID Registration System is a full-stack application for managing multi-portal RFID driven registrations. It combines a React/Vite front-end for portal operators and administrators with an Express/PostgreSQL backend that orchestrates registrations, RFID tag assignment, and automatic tag release based on live reader logs.

## Key Features
- Multi-portal workflow covering portal selection, data capture for individual or batch registrations, and RFID tag assignment.
- Registration UI with searchable school and university lists, district filtering per province, language tagging, and batch counting.
- Admin views that surface registration stats, tag inventory, and filterable participant details across portals.
- Express API backed by PostgreSQL with automatic card availability syncing, transactional tag linking, and background watchers that release tags when EXIT events are detected.
- Health checks and environment-driven configuration for flexible deployment across staging and production environments.

## Repository Layout
- `Backend/` - Node.js Express service exposing registration and RFID APIs (`src/server.js`, `src/routes/tags.js`).
- `frontend/` - React + Vite single-page app for portal operators (`src/`, `data/`, styling assets).
- `Database/schema.sql` - PostgreSQL schema defining `logs`, `registration`, `members`, and `rfid_cards` tables plus supporting indexes.
- `.github/` - Optional CI/CD workflows for automation.

## Prerequisites
- Node.js 18 LTS or newer (required by Vite 7 and modern React tooling).
- npm 9 or newer (bundled with Node 18).
- PostgreSQL 13 or newer.
- RFID reader(s) or a process capable of posting tap events to the `/api/tags/rfidRead` endpoint.

## Configuration
### Backend environment (`Backend/.env`)
| Variable | Required | Example | Notes |
| --- | --- | --- | --- |
| `PORT` | optional | `4000` | Port the Express API listens on. |
| `DATABASE_URL` | required | `postgres://user:pass@localhost:5432/rfid_reg` | PostgreSQL DSN with credentials and database name. |
| `PG_SSL` | optional | `false` | Set to `true` (case-insensitive) when connecting to managed Postgres that requires TLS. |

### Frontend environment (`frontend/.env`)
| Variable | Required | Example | Notes |
| --- | --- | --- | --- |
| `VITE_API_BASE` | required | `http://localhost:4000` | Base URL of the backend API consumed by the React app. |

## Setup
### 1. Prepare the database
1. Create a PostgreSQL database and user with access to it.
2. Run the schema to provision tables and indexes:
   ```bash
   psql -d rfid_reg -f Database/schema.sql
   ```
3. Ensure the credentials and database name match the `DATABASE_URL` in `Backend/.env`.

### 2. Install backend dependencies
```bash
cd Backend
npm install
```

### 3. Install frontend dependencies
```bash
cd ../frontend
npm install
```

## Local Development Workflow
1. Start PostgreSQL and verify connectivity using the credentials in `Backend/.env`.
2. Run the backend (from `Backend/`):
   ```bash
   npm run dev
   ```
   This boots Express with `nodemon`, exposes `GET /health`, and serves all `/api/tags/*` routes.
3. Run the frontend (from `frontend/`):
   ```bash
   npm run dev
   ```
   Vite prints the local development URL (default `http://localhost:5173`).
4. Open the Vite URL in a browser. The app stores the selected portal in `localStorage`, polls backend health every 15 seconds, and guides operators through registration and tag assignment.

## API Surface
All endpoints are prefixed with `VITE_API_BASE` (default `http://localhost:4000`).

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Heartbeat returning `{ ok, ts }` for automated or manual checks. |
| `GET` | `/api/tags/admin/registrations` | Returns every registration record for admin analytics. |
| `POST` | `/api/tags/register` | Creates a new registration (individual or batch leader) and returns the registration `id`. |
| `POST` | `/api/tags/updateCount` | Updates the latest registration `group_size` after a batch headcount. |
| `POST` | `/api/tags/link` | Links the most recent REGISTERED card to a leader or member (transactional). |
| `GET` | `/api/tags/list-cards` | Lists known RFID cards and their assignment status. |
| `POST` | `/api/tags/rfidRead` | Records a raw RFID tap event from readers into the `logs` table. |

The backend keeps the `rfid_cards` table synchronized with new REGISTER events and periodically releases cards when EXIT or EXITOUT logs arrive.

## Frontend Flows
- **Portal Selection** - Operators pick a portal (`portal1`, `portal2`, `portal3`), persisted in `localStorage` for quick reloads.
- **Registration Flow** - Guides operators through individual or batch registration:
  - Province to district filtering.
  - Searchable school and university selectors backed by JSON datasets found in `frontend/data/`.
  - Language tagging (up to two languages) and batch size capture via RFID tap counts.
- **Tag Assignment** - Initiates leader RFID linking, followed by member tag assignments where applicable.
- **Admin Portal** - Summaries for total registrations, people counts by type, available versus assigned tags, and a filterable table of records.

## Data Sources
- Static JSON lookups (`frontend/data/`) provide provinces, districts, school lists (grouped by province and district), and universities. Update these files when administrative datasets change.
- RFID readers should POST their events to `/api/tags/rfidRead`. The backend automatically:
  - Adds unseen cards into `rfid_cards` via `syncRfidCardsFromLogs()`.
  - Runs `checkAndReleaseOnNewExitout()` every 3 seconds to free cards whose holders exit a portal.

## Development Notes
- Backend uses `nodemon` for hot reload during development; production should rely on `npm start`.
- Frontend linting is available via `npm run lint` inside `frontend/`.
- All code uses ES modules and modern JavaScript; stick to Node 18 or newer to avoid runtime issues.

## Troubleshooting
- **Health check fails:** Confirm the backend is running and that `PORT` and `VITE_API_BASE` match. The frontend header shows current status.
- **Card not linking:** Ensure an RFID tap with label `REGISTER` is logged for the portal before calling `/api/tags/link`; otherwise the backend returns informative errors (for example "No card tapped for registration").
- **Tags never release:** Verify readers send EXIT or EXITOUT events and timestamps are current. The watcher only releases tags for events within the last three minutes.
- **Static data missing:** Ensure the `frontend/data` directory is served (Vite copies everything under `public/`; if deploying, include the data files or adjust fetch paths).
