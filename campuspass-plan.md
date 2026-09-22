# CampusPass — Development Plan

## Overview

Build **CampusPass**, a responsive, lightweight college event ticketing web app.

- **Stack**: React (Vite) + Node.js/Express + PostgreSQL
- **Auth**: JWT, .edu email domain validation, two roles: `student` and `organizer`
- **Payment**: Mock UI-only confirmation — no payment data persisted
- **QR Codes**: Client-side via `qrcode.react`
- **Notifications**: Mock email service (console.log only) + in-app notification records

### Scope

| In Scope | Out of Scope |
|---|---|
| Student + organizer auth | Third-party SSO (Google, etc.) |
| Event CRUD for organizers | Real payment processing |
| Ticket reservation + QR display | File/image uploads for events |
| In-app notifications | Real email delivery |
| Responsive UI | Native mobile app |

---

## Data Model

### `users`
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| email | VARCHAR UNIQUE | Must end in `.edu` |
| password_hash | VARCHAR | bcrypt |
| role | ENUM student/organizer | Self-designated at registration |
| name | VARCHAR | |
| created_at | TIMESTAMP | |

### `events`
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| organizer_id | UUID FK → users | |
| title | VARCHAR | |
| description | TEXT | |
| location | VARCHAR | |
| event_date | TIMESTAMP | |
| capacity | INT | |
| tickets_remaining | INT | Decremented on reservation |
| price | DECIMAL | For display only |
| status | ENUM draft/published/cancelled | |
| created_at | TIMESTAMP | |

### `tickets`
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| user_id | UUID FK → users | |
| event_id | UUID FK → events | |
| qr_payload | VARCHAR | JSON string encoded as QR |
| status | ENUM reserved/cancelled | |
| reserved_at | TIMESTAMP | |

### `notifications`
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| user_id | UUID FK → users | |
| type | VARCHAR | e.g. ticket_confirmed, event_cancelled |
| message | TEXT | |
| read | BOOLEAN | Default false |
| created_at | TIMESTAMP | |

---

## Sub-Tasks

---

### Sub-Task 1 — Project Scaffolding & Monorepo Setup
**Status**: [ ] pending

**Intent**  
Set up the project layout so both frontend and backend can be developed and run independently with a consistent structure. This is the foundation every other sub-task depends on.

**Expected Outcomes**
- `/client` — Vite + React app with Tailwind CSS configured
- `/server` — Node.js + Express app with a working health-check endpoint
- `/server/db` — PostgreSQL connection module (using `pg` or `drizzle-orm`)
- Root `package.json` with `dev` scripts for both client and server (e.g. via `concurrently`)
- `.env.example` documenting all required environment variables
- `README.md` with local setup instructions

**Todo List**
- [ ] Initialise monorepo root with a root `package.json`
- [ ] Scaffold `/client` using `npm create vite@latest` (React + TypeScript template)
- [ ] Install and configure Tailwind CSS in `/client`
- [ ] Scaffold `/server` with Express, `dotenv`, `cors`, `helmet`
- [ ] Add PostgreSQL client (`pg`) and connection module at `/server/db/index.js`
- [ ] Add a `GET /health` endpoint to verify the server is running
- [ ] Create `.env.example` with `DATABASE_URL`, `JWT_SECRET`, `PORT`, `CLIENT_URL`
- [ ] Add root-level `dev` script using `concurrently`
- [ ] Write `README.md` with setup and run instructions

**Relevant Context**
- No existing codebase — greenfield setup
- Use `pg` (node-postgres) directly; no ORM needed for this scope
- Tailwind v3 with Vite requires `postcss` + `autoprefixer`

---

### Sub-Task 2 — Database Schema & Migrations
**Status**: [ ] pending

**Intent**  
Define and apply the full PostgreSQL schema as raw SQL migration files so the database structure is reproducible and version-controlled.

**Expected Outcomes**
- `/server/db/migrations/001_initial_schema.sql` with all four tables
- A `migrate.js` script that runs all pending migrations in order
- `npm run migrate` works from the `/server` directory
- Tables verified via `psql` or a schema dump

**Todo List**
- [ ] Create `/server/db/migrations/` directory
- [ ] Write `001_initial_schema.sql` with `CREATE TABLE` for `users`, `events`, `tickets`, `notifications`
- [ ] Add ENUM types for `role` (student/organizer), `event status` (draft/published/cancelled), `ticket status` (reserved/cancelled)
- [ ] Add foreign-key constraints and indexes on `organizer_id`, `user_id`, `event_id`
- [ ] Write `/server/db/migrate.js` to execute migration files in order using `pg`
- [ ] Add `migrate` script to `/server/package.json`
- [ ] Test migration runs cleanly on a fresh database

**Relevant Context**
- Schema defined in the Data Model section above
- Use `IF NOT EXISTS` guards so migrations are idempotent

---

### Sub-Task 3 — Authentication API & Middleware
**Status**: [ ] pending

**Intent**  
Implement the backend auth layer: registration with `.edu` validation, password hashing, JWT issuance, and a reusable role-guard middleware.

**Expected Outcomes**
- `POST /api/auth/register` — validates `.edu` email, hashes password, stores user, returns JWT
- `POST /api/auth/login` — validates credentials, returns JWT
- `GET /api/auth/me` — returns current user from JWT (protected)
- Reusable `authMiddleware` that verifies JWT and attaches `req.user`
- Reusable `requireRole(role)` middleware factory for role-gated routes

**Todo List**
- [ ] Install `bcryptjs` and `jsonwebtoken`
- [ ] Create `/server/routes/auth.js` with register and login handlers
- [ ] Validate that email ends with `.edu` in the register handler (reject otherwise with 400)
- [ ] Hash passwords with `bcryptjs` before storing
- [ ] Issue a signed JWT containing `{ id, email, role }` with an expiry of 7 days
- [ ] Create `/server/middleware/auth.js` — verifies JWT from `Authorization: Bearer` header
- [ ] Create `/server/middleware/requireRole.js` — checks `req.user.role` matches expected role
- [ ] Add `GET /api/auth/me` protected route
- [ ] Register auth routes in `server.js` under `/api/auth`

**Relevant Context**
- JWT secret from `process.env.JWT_SECRET`
- `.edu` check: `email.endsWith('.edu')` is sufficient for this scope

---

### Sub-Task 4 — Events API (Organizer CRUD)
**Status**: [ ] pending

**Intent**  
Build the events REST API so organizers can create, update, publish, and cancel events, and students can browse published events.

**Expected Outcomes**
- `POST /api/events` — organizer only; creates event in `draft` status
- `GET /api/events` — public (auth required); returns published events
- `GET /api/events/:id` — returns single event
- `PATCH /api/events/:id` — organizer only; update fields or change status
- `DELETE /api/events/:id` — organizer only; soft-cancel (set status = cancelled)
- Only the owning organizer can modify their own events

**Todo List**
- [ ] Create `/server/routes/events.js`
- [ ] Implement `POST /api/events` — insert with `status = draft`, `tickets_remaining = capacity`
- [ ] Implement `GET /api/events` — filter by `status = published`, order by `event_date ASC`
- [ ] Implement `GET /api/events/:id`
- [ ] Implement `PATCH /api/events/:id` — check `organizer_id = req.user.id` before update
- [ ] Implement `DELETE /api/events/:id` — set `status = cancelled`
- [ ] Apply `authMiddleware` to all routes; apply `requireRole('organizer')` to write routes
- [ ] Register event routes in `server.js` under `/api/events`

**Relevant Context**
- `tickets_remaining` is decremented in Sub-Task 5 (ticket reservation), not here
- Organizer ownership check must be enforced server-side, not just client-side

---

### Sub-Task 5 — Ticket Reservation API & Notification Trigger
**Status**: [ ] pending

**Intent**  
Implement instant ticket reservation: atomically decrement `tickets_remaining` and create a ticket record, then trigger a mock notification.

**Expected Outcomes**
- `POST /api/tickets` — authenticated students reserve a ticket for an event
- Returns 409 if `tickets_remaining = 0`
- Returns 409 if student already holds a ticket for the event
- Creates a `notifications` row for the student (type: `ticket_confirmed`)
- Logs a mock email to the console
- `GET /api/tickets/mine` — returns the authenticated student's tickets

**Todo List**
- [ ] Create `/server/routes/tickets.js`
- [ ] Implement `POST /api/tickets` inside a PostgreSQL transaction:
  - Lock the event row (`SELECT ... FOR UPDATE`)
  - Check `tickets_remaining > 0`, else return 409
  - Check no existing ticket for this user+event, else return 409
  - Decrement `tickets_remaining` on the event
  - Insert ticket row with `qr_payload = JSON.stringify({ ticketId, eventId, userId, reservedAt })`
- [ ] Create `/server/services/notification.js` — inserts a notification row and `console.log`s a mock email
- [ ] Call the notification service after successful reservation
- [ ] Implement `GET /api/tickets/mine` — returns tickets with joined event details
- [ ] Apply `authMiddleware` and `requireRole('student')` to write routes
- [ ] Register ticket routes in `server.js` under `/api/tickets`

**Relevant Context**
- The transaction is critical to prevent overselling under concurrent requests
- `qr_payload` is a JSON string; the QR code is rendered client-side from this string

---

### Sub-Task 6 — Notifications API
**Status**: [ ] pending

**Intent**  
Expose the in-app notification records so the frontend can display an unread count and a notification list.

**Expected Outcomes**
- `GET /api/notifications` — returns the authenticated user's notifications, newest first
- `PATCH /api/notifications/:id/read` — marks a single notification as read
- `PATCH /api/notifications/read-all` — marks all of the user's notifications as read

**Todo List**
- [ ] Create `/server/routes/notifications.js`
- [ ] Implement `GET /api/notifications` — filter by `user_id = req.user.id`, order by `created_at DESC`
- [ ] Implement `PATCH /api/notifications/:id/read`
- [ ] Implement `PATCH /api/notifications/read-all`
- [ ] Apply `authMiddleware` to all routes
- [ ] Register notification routes in `server.js` under `/api/notifications`

**Relevant Context**
- Notifications are created by the service in Sub-Task 5
- No WebSocket/polling required — frontend fetches on page load and after reservation

---

### Sub-Task 7 — Frontend: Auth Pages & API Client
**Status**: [ ] pending

**Intent**  
Build the login and registration pages, a central API client, and the global auth context so all subsequent frontend sub-tasks have a working auth foundation.

**Expected Outcomes**
- `/login` and `/register` pages with form validation
- Register form includes a role selector (student / organizer)
- Client rejects non-.edu emails before submitting
- A global `AuthContext` providing `user`, `token`, `login()`, `logout()`
- A central `apiClient.js` that attaches the JWT `Authorization` header to all requests
- Protected route wrapper that redirects to `/login` if unauthenticated

**Todo List**
- [ ] Install `react-router-dom` and `axios` (or `fetch` wrapper) in `/client`
- [ ] Create `/client/src/api/client.js` — Axios instance with base URL and JWT interceptor
- [ ] Create `/client/src/context/AuthContext.jsx` — stores token in `localStorage`, exposes user + helpers
- [ ] Build `RegisterPage` with fields: name, email (.edu hint), password, role selector
- [ ] Build `LoginPage` with email + password fields
- [ ] Add client-side `.edu` email validation on the register form
- [ ] Create `ProtectedRoute` component that checks auth and redirects
- [ ] Create `RoleRoute` component that checks role and redirects (student vs organizer)
- [ ] Set up `react-router-dom` routes in `App.jsx`

**Relevant Context**
- Use `localStorage` for JWT persistence (sufficient for this scope)
- Error messages from the API (e.g. non-.edu, wrong password) should surface in the form

---

### Sub-Task 8 — Frontend: Student Event Browser & Checkout Flow
**Status**: [ ] pending

**Intent**  
Build the student-facing pages: a list of published events, an event detail page, the checkout flow with mock payment, and the confirmation screen.

**Expected Outcomes**
- `/events` — card grid of published events with title, date, location, price, tickets remaining
- `/events/:id` — detail page with a "Reserve Ticket" button
- Clicking Reserve triggers `POST /api/tickets`; on success, transitions to mock payment screen
- Mock payment screen shows a "Payment Confirmed" message (no form, no data sent)
- After confirmation, redirects to `/my-tickets`

**Todo List**
- [ ] Build `EventListPage` — fetches `GET /api/events`, renders `EventCard` components
- [ ] Build `EventCard` component — displays title, date, location, price, remaining tickets badge
- [ ] Build `EventDetailPage` — fetches single event, shows full description and Reserve button
- [ ] Disable Reserve button and show "Sold Out" when `tickets_remaining = 0`
- [ ] On Reserve click, call `POST /api/tickets`; show loading state
- [ ] On success, render `MockPaymentScreen` — static "Payment Confirmed ✓" with event summary
- [ ] Add a "Go to My Tickets" button on the confirmation screen
- [ ] Handle 409 errors (already reserved, sold out) with user-friendly messages

**Relevant Context**
- Mock payment is a UI state transition only — no API call, no data persisted
- `EventCard` and `EventDetailPage` are shared by the organizer preview in Sub-Task 9

---

### Sub-Task 9 — Frontend: Organizer Dashboard
**Status**: [ ] pending

**Intent**  
Build the organizer-only dashboard for creating, editing, and managing events.

**Expected Outcomes**
- `/organizer` — dashboard listing the organizer's own events with status badges
- `/organizer/events/new` — form to create a new event
- `/organizer/events/:id/edit` — pre-filled edit form
- Publish and Cancel action buttons on each event row
- Ticket count shown per event (capacity vs remaining)

**Todo List**
- [ ] Build `OrganizerDashboardPage` — fetches organizer's events (filter client-side by `organizer_id = user.id` or add a `/api/events/mine` endpoint)
- [ ] Build `EventFormPage` — shared create/edit form with fields matching the events schema
- [ ] On create: call `POST /api/events`; on edit: call `PATCH /api/events/:id`
- [ ] Add Publish button — calls `PATCH /api/events/:id` with `{ status: 'published' }`
- [ ] Add Cancel button — calls `DELETE /api/events/:id`; confirm before sending
- [ ] Show `tickets_remaining / capacity` on the dashboard table
- [ ] Gate all organizer routes behind `RoleRoute` (organizer only)

**Relevant Context**
- `RoleRoute` is built in Sub-Task 7
- Consider adding `GET /api/events/mine` on the backend if filtering client-side feels fragile

---

### Sub-Task 10 — Frontend: My Tickets & QR Code Display
**Status**: [ ] pending

**Intent**  
Build the student's ticket wallet page where they can view their reserved tickets and see a QR code for each.

**Expected Outcomes**
- `/my-tickets` — list of the student's tickets with event name, date, and status
- Each ticket renders a QR code generated from `qr_payload`
- QR code is displayed using `qrcode.react`
- Cancelled tickets are visually dimmed

**Todo List**
- [ ] Install `qrcode.react` in `/client`
- [ ] Build `MyTicketsPage` — fetches `GET /api/tickets/mine`
- [ ] Build `TicketCard` component — shows event title, date, location, status badge
- [ ] Render `<QRCodeSVG value={ticket.qr_payload} />` inside each `TicketCard`
- [ ] Visually differentiate `reserved` vs `cancelled` tickets (opacity, strikethrough)

**Relevant Context**
- `qr_payload` is a JSON string set at reservation time (Sub-Task 5)
- `qrcode.react` exports `QRCodeSVG` and `QRCodeCanvas` — SVG is preferred for sharpness

---

### Sub-Task 11 — Frontend: Notification Bell & Panel
**Status**: [ ] pending

**Intent**  
Add a notification bell to the nav bar showing the unread count, with a dropdown panel listing notifications.

**Expected Outcomes**
- Bell icon in the nav bar with a red badge showing unread count
- Clicking the bell opens a dropdown listing notifications (newest first)
- Clicking a notification or "Mark all read" calls the API and clears the badge
- Notifications are fetched on app load and after ticket reservation

**Todo List**
- [ ] Build `NotificationBell` component — fetches `GET /api/notifications`, shows unread count badge
- [ ] Build `NotificationPanel` dropdown — lists notification messages and timestamps
- [ ] Add "Mark all read" button — calls `PATCH /api/notifications/read-all`
- [ ] Re-fetch notifications after `POST /api/tickets` succeeds
- [ ] Add `NotificationBell` to the shared `NavBar` component
- [ ] Ensure the bell is visible for both student and organizer roles

**Relevant Context**
- No real-time updates needed — polling or refetch-on-action is sufficient
- The `notifications` rows are created server-side in Sub-Task 5

---

### Sub-Task 12 — Responsive Styling & Polish
**Status**: [ ] pending

**Intent**  
Ensure the full application is mobile-responsive and visually consistent using Tailwind utility classes.

**Expected Outcomes**
- All pages render correctly on mobile (375px), tablet (768px), and desktop (1280px)
- Consistent color palette, typography scale, and spacing applied across all components
- Navigation collapses to a hamburger menu on mobile
- Empty states (no events, no tickets, no notifications) have placeholder UI
- Loading and error states are handled in all data-fetching components

**Todo List**
- [ ] Define a Tailwind theme extension in `tailwind.config.js` (primary color, font)
- [ ] Build a responsive `NavBar` with hamburger toggle for mobile
- [ ] Audit each page for responsive breakpoints (`sm:`, `md:`, `lg:`)
- [ ] Add empty-state components for event list, ticket list, notification panel
- [ ] Add skeleton loaders or spinners for all API fetch states
- [ ] Add error boundary or per-component error messages for failed fetches

**Relevant Context**
- Tailwind is configured in Sub-Task 1
- This sub-task is best done as a final pass after all feature pages exist

---

## Implementation Notes

- Sub-Tasks 1–6 are backend-focused; Sub-Tasks 7–12 are frontend-focused
- Sub-Tasks can be worked in parallel within their layer (e.g. Sub-Task 3 and Sub-Task 4 are independent)
- The frontend sub-tasks (7–11) have a soft dependency order: Sub-Task 7 (auth context) must come first
- Sub-Task 12 is a final polish pass and should be last
