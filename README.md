# Vibe-Coding
# Restaurant Reservation Management System

A full-stack reservation system with role-based access (Customer / Admin), built with
React, Node.js/Express, and MongoDB.

## Tech Stack

- **Frontend:** React 18 + Vite, React Router, Axios
- **Backend:** Node.js, Express
- **Database:** MongoDB with Mongoose
- **Auth:** JWT (JSON Web Tokens), bcrypt password hashing

## Project Structure

```
restaurant-reservation-system/
├── backend/
│   ├── config/db.js              # MongoDB connection
│   ├── models/                   # User, Table, Reservation schemas
│   ├── middleware/                # auth (JWT + role guard), centralized error handler
│   ├── controllers/               # business logic (auth, tables, reservations)
│   ├── routes/                    # Express route definitions
│   ├── utils/timeUtils.js         # time-slot parsing & overlap detection
│   ├── utils/seed.js              # seeds an admin user + starter tables
│   └── server.js                  # app entry point
└── frontend/
    └── src/
        ├── api/axios.js           # pre-configured API client (attaches JWT)
        ├── context/AuthContext.jsx
        ├── components/            # Navbar, ProtectedRoute
        └── pages/                 # Login, Register, Customer & Admin views
```

## Setup Instructions

### Prerequisites
- Node.js 18+
- A MongoDB instance (local `mongod`, or a free MongoDB Atlas cluster)

### 1. Backend

```bash
cd backend
npm install
cp .env.example .env
# edit .env: set MONGO_URI to your MongoDB connection string, and set a real JWT_SECRET
npm run seed     # creates an admin user + 6 sample tables
npm run dev      # starts the API on http://localhost:5000
```

Seeded admin credentials: `admin@restaurant.com` / `admin123` (change this after first login in any real deployment).

### 2. Frontend

```bash
cd frontend
npm install
cp .env.example .env
# edit .env if your backend isn't on http://localhost:5000/api
npm run dev      # starts the app on http://localhost:5173
```

Customers can register directly from the UI. Admin accounts are not self-service (see
Assumptions below) — use the seeded admin account, or promote a user's `role` field to
`"admin"` directly in the database.

## Assumptions Made

- **Single restaurant, fixed tables:** the system assumes one restaurant location. Tables
  are seeded via `npm run seed` (6 tables, capacities 2–8), but admins can add/edit/deactivate
  tables at runtime through the "Manage Tables" screen.
- **Time slots are fixed strings** in `"HH:MM-HH:MM"` format (e.g. `18:00-19:30`) rather than
  a free-form start time + duration. This keeps slot comparison simple and predictable for
  a restaurant with a known seating rhythm. The frontend offers a preset list of slots; the
  API itself will accept any valid `HH:MM-HH:MM` string, so this can be extended later.
- **No self-service admin signup:** allowing anyone to register as an admin would defeat
  role-based access control. Admins are provisioned via the seed script or direct DB update.
- **Bookings can't be made for past dates**, compared by calendar date (not time-of-day),
  which keeps the validation simple and avoids timezone edge cases for this assignment's scope.
- **Table deactivation is a soft delete** (`isActive: false`) rather than a hard delete, so
  historical reservations still resolve to a real table document.
- **One reservation = one table.** Combining multiple tables for a single large party is out
  of scope, consistent with the assignment notes on scope.

## Reservation & Availability Logic

This is the core evaluation area, so here's exactly how it works:

1. **Time representation:** each time slot (`"18:00-19:30"`) is parsed into `startMinutes`
   and `endMinutes` (minutes since midnight) and stored alongside the human-readable string.
   This makes overlap comparisons cheap and unambiguous.

2. **Conflict detection:** two time ranges overlap if `startA < endB && startB < endA`. When
   creating or editing a reservation, the API pulls every **confirmed** reservation for the
   same table and date, and checks this condition against each one. Cancelled reservations
   are excluded, so a freed-up slot becomes bookable again immediately.

3. **Capacity validation:** a table is only offered/accepted if its `capacity >= guests`.
   This is checked both when auto-assigning a table and when a customer explicitly picks one.

4. **Availability endpoint** (`GET /api/reservations/availability`): given a date, time slot,
   and party size, the backend returns every active table that (a) can seat the party and
   (b) has no overlapping confirmed reservation — sorted smallest-capacity-first, so small
   parties don't unnecessarily occupy large tables. The frontend calls this before letting
   the customer confirm a booking.

5. **Booking:** `POST /api/reservations` either uses a `tableId` the customer selected from
   the availability check (re-validated server-side to avoid race conditions/stale data) or,
   if omitted, auto-assigns the smallest available suitable table.

6. **Admin edits:** `PATCH /api/reservations/:id` re-runs the same capacity + overlap checks
   whenever date, time, guest count, or table is changed, excluding the reservation being
   edited itself from the conflict check.

All validation failures return meaningful HTTP status codes (`400` for bad input, `404` for
missing resources, `409` for conflicts) with a human-readable `message`.

## Role-Based Access (Customer vs Admin)

- Every protected route requires a valid JWT (`Authorization: Bearer <token>`), verified by
  the `protect` middleware, which loads the user and attaches it to `req.user`.
- The `authorize(...roles)` middleware then restricts specific routes by role:
  - Customers can only create reservations, view **their own** reservations (`/my`), and
    cancel their own reservations. Attempting to cancel someone else's reservation returns
    `403 Forbidden`.
  - Admins have access to `GET /api/reservations` (all reservations, optionally filtered by
    `?date=`), `PATCH /api/reservations/:id` (edit or cancel any reservation), and table
    management endpoints.
- On the frontend, `ProtectedRoute` redirects unauthenticated users to `/login`, and can also
  gate a route to a specific role (e.g. `/admin/tables` is admin-only). The Navbar and home
  route also render different views based on `user.role`.

## Known Limitations

- Time slots are a fixed preset list in the UI (though the API accepts any valid slot string).
- No email/SMS notifications or reminders (explicitly out of scope per the assignment).
- No pagination on the admin "all reservations" list — fine at small scale, would need
  pagination/virtualization for a high-volume restaurant.
- No password reset / email verification flow.
- Concurrent booking race conditions are mitigated by re-validating on the server at write
  time, but under very high concurrency a database-level unique constraint or transaction
  would be a stronger guarantee than the current check-then-write pattern.

## Areas for Improvement With Additional Time

- Add optimistic locking or a MongoDB transaction around the availability check + reservation
  insert to fully close the race-condition window under concurrent load.
- Support configurable/admin-editable time slots instead of a fixed frontend list.
- Add automated tests (unit tests for the overlap/availability logic, integration tests for
  the API routes).
- Add pagination and search/filter (by customer, table, status) to the admin dashboard.
- Support multi-table bookings for large parties.
- Add a "waitlist" feature for fully booked slots.
