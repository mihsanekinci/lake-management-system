[🇹🇷 Türkçe](README.tr.md) | 🇬🇧 English

# Lake Management System

A full-stack web application for managing recreational fishing activities on Van Lake (Van Gölü, Turkey). Users can rent boats and equipment, view an interactive geospatial map of lake zones, participate in zone-based forums, and track activities. Admins get analytics dashboards covering revenue, user spending, and zone statistics.

---

## Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Frontend | React | ^19.2.0 |
| Frontend | Vite | ^7.2.4 |
| Frontend | Leaflet + React-Leaflet | ^1.9.4 / ^5.0.0 |
| Frontend | React Hot Toast | ^2.6.0 |
| Frontend | React Icons | ^5.5.0 |
| Backend | Node.js + Express | ^5.1.0 |
| Backend | pg (PostgreSQL driver) | ^8.16.3 |
| Backend | jsonwebtoken | ^9.0.3 |
| Backend | bcrypt | ^6.0.0 |
| Backend | Supabase JS client | ^2.86.0 |
| Database | PostgreSQL (Supabase cloud) | — |
| Database | PostGIS extension | — |

---

## Architecture Overview

```
Browser (React SPA)
    │
    │  HTTP + Bearer JWT
    ▼
Express API  (port 3000)
    │  Controller → Service → SQL
    ▼
Supabase PostgreSQL  (cloud)
    │
    └── PostGIS geometry columns
        (boat positions, lake zone polygons, hotspots)
```

**Request path:** React components call `src/api/api.js`, which attaches the JWT from `localStorage` and hits the Express REST API. Each route module delegates to a controller, which calls a service that runs parameterized SQL via `pg.Pool`. No ORM is used.

**Auth:** JWT is issued at login (7-day expiry, HS256). The `authMiddleware` validates it on protected routes; `adminMiddleware` additionally checks `role_id`.

**Geospatial:** Boat current positions and lake zone polygons are stored as PostGIS geometry (SRID 4326). The backend returns them as GeoJSON via `ST_AsGeoJSON`; the frontend renders them on a Leaflet map.

**Radar simulation:** `backend/services/radarSimulation.js` runs a background loop that writes random/simulated coordinates to `boats.current_geom`. This is not real GPS data.

---

## Features

### Users
- Register and log in (JWT-based sessions, passwords hashed with bcrypt)
- View personal rental history with filters
- View own spending stats

### Map
- Interactive Leaflet map of Van Lake
- Clickable zone polygons — selecting a zone filters the sidebar content
- Live boat markers pulled from the database (positions are simulated, not GPS)
- Hotspot markers per zone

### Boat Rental
- Browse available boats sorted by price
- Rent a boat for a specified duration (billed at `price_per_hour × hours`)
- View and complete active rentals
- Admin: create, edit, delete boats

### Equipment Rental
- Same rental flow as boats, applied to fishing equipment
- "Return all" endpoint to complete all active equipment rentals at once
- Admin: manage equipment inventory

### Activities / Events
- Time-categorized activity list per zone (past, current, upcoming)
- Admin: create, edit, delete activities

### Forum
- Zone-scoped posts with title, content, and photo URLs
- Comments and likes on posts
- Admin: view forum statistics per user

### Admin Analytics
- **AdminStatsPanel:** user spending rankings, forum post/comment counts, zone activity/post/duration stats, popular zones
- **AccountingPanel:** monthly revenue breakdown, boat vs. equipment revenue split, trend analysis over time
- **RentalHistoryPanel:** full rental history with status filters

---

## Folder Structure

```
balik-proje/
├── backend/
│   ├── config/db.js          # pg.Pool connected to Supabase
│   ├── controllers/          # 8 controllers (one per domain)
│   ├── middleware/           # auth, admin, error handler, request logger
│   ├── routes/               # 9 Express routers
│   ├── services/             # 11 services + radarSimulation.js
│   ├── utils/geojson.js
│   └── server.js
├── frontend/
│   └── src/
│       ├── api/api.js        # Single fetch wrapper (~430 lines)
│       ├── components/
│       │   ├── GameMap/      # Leaflet map + markers
│       │   ├── Sidebar/      # 5-tab sidebar (Info, Boat, Equip, Forum, Account)
│       │   ├── features/     # Domain-specific list/card components
│       │   ├── forms/        # Login, Register, Boat, Equipment, Activity forms
│       │   ├── modals/       # Confirm, ImageLightbox, PostCreate
│       │   ├── panels/       # Admin dashboards
│       │   └── ui/           # Button, Card, Input, Modal, Badge, etc.
│       ├── App.jsx
│       └── main.jsx
├── database/
│   ├── create_activities.sql
│   ├── create_equipment_rentals.sql
│   ├── insert_sample_activities.sql
│   ├── project_queries.sql       # 10 complex analytical queries
│   └── advanced_queries.sql
└── .env
```

---

## Setup & Running

### Prerequisites
- Node.js ≥ 18
- A Supabase project with the schema already applied (see `database/` SQL files)
- The `.env` file populated (see below)

### 1. Environment Variables

Create a `.env` file in the project root with the following:

```env
PORT=3000
JWT_SECRET=<replace-with-a-strong-random-secret>
JWT_EXPIRES_IN=7d
SUPABASE_URL=https://<your-project>.supabase.co
SUPABASE_ANON_KEY=<your-anon-key>
DATABASE_URL=postgresql://postgres.<your-project>:<password>@aws-0-eu-central-1.pooler.supabase.com:6543/postgres
```

### 2. Database

Run the SQL files in Supabase's SQL Editor (or via `psql`) in this order:

```
database/create_activities.sql
database/create_equipment_rentals.sql
database/insert_sample_activities.sql   # optional sample data
```

### 3. Backend

```bash
cd backend
npm install
node server.js
# API listens on http://localhost:3000
```

To seed an admin user:

```bash
node create_admin_user.js
```

### 4. Frontend

```bash
cd frontend
npm install
npm run dev
# Dev server at http://localhost:5173
```

Production build:

```bash
npm run build   # output in frontend/dist/
```

---

## Known Gaps & TODOs

- **Photo upload is not implemented.** The database has a `post_photos` table and the frontend renders photo URLs, but there is no file upload endpoint in the backend. Photos must be pre-hosted URLs.
- **Payment processing is not implemented.** A `payments` table exists in the schema, but no payment gateway is integrated. Rental amounts are recorded but no actual charge is made.
- **Radar simulation is fake.** Boat positions on the map are generated by `radarSimulation.js`, not real GPS devices.
- **No email verification.** Registration accepts any email string without sending a confirmation.
- **No rate limiting.** Auth endpoints (`/register`, `/login`) have no brute-force protection.
- **JWT secret in `.env` is a placeholder.** Change `super-secret-change-this` before any non-local deployment.
- **`SUPABASE_ANON_KEY` is committed to `.env`.** This file should not be in version control. Rotate the key if the repo is or was public.
- **Boat availability race condition.** The service uses `SELECT FOR UPDATE` to lock the boat row, but concurrent load behavior has not been verified.
