# CampusPass 🎫

> A responsive, zero-dependency single-page web application for college event ticketing — built with HTML5, Tailwind CSS, and vanilla JavaScript.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Summary](#2-architecture-summary)
   - [Technology Stack](#technology-stack)
   - [Application Layers](#application-layers)
   - [Component Map](#component-map)
   - [Data Flow](#data-flow)
3. [Features](#3-features)
4. [Deploying via GitHub Pages](#4-deploying-via-github-pages)
   - [Prerequisites](#prerequisites)
   - [Step-by-Step Deployment](#step-by-step-deployment)
   - [Enabling a Custom Domain (optional)](#enabling-a-custom-domain-optional)
   - [Updating the Live Site](#updating-the-live-site)
5. [Maintenance Guide](#5-maintenance-guide)
   - [Event Data Model Reference](#event-data-model-reference)
   - [Adding a New Event Field](#adding-a-new-event-field)
   - [Adding a New Event Category](#adding-a-new-event-category)
   - [Editing Seed Events](#editing-seed-events)
   - [Changing the .edu Validation Rule](#changing-the-edu-validation-rule)
   - [Extending the Ticket Model](#extending-the-ticket-model)
   - [Theming & Branding](#theming--branding)
6. [Known Limitations](#6-known-limitations)
7. [Roadmap](#7-roadmap)
8. [License](#license)

---

## 1. Project Overview

**CampusPass** is a lightweight, front-end-only event ticketing platform designed for college campuses. Student organizations can publish events; students can discover, reserve, and manage tickets — all from a single HTML file with no build tooling, no backend server, and no database required. This platform was planned and built using IBM BOB during a 60 minute webinar for a group of college students. CampusPass is for DEMO purposes only.

### Who it's for

| Role | What they can do |
|---|---|
| **Student** | Register with a `.edu` email, browse published events, reserve up to 2 tickets per event, view tickets with QR codes |
| **Event Organizer** | Register as an organizer, publish new events with title, category, capacity, price, date, and location |

### Design goals

- **No build step** — open `campuspass.html` in any browser, or serve it from GitHub Pages.
- **No backend required** — all state lives in JavaScript memory for the session.
- **Accessible & responsive** — works on phones, tablets, and desktops.
- **Extensible** — the data model and render functions are structured so new fields and features can be added without refactoring.

---

## 2. Architecture Summary

### Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| Markup | HTML5 | Single file: `campuspass.html` |
| Styling | [Tailwind CSS v3](https://tailwindcss.com) (CDN) | No PostCSS build; loaded from `cdn.tailwindcss.com` |
| Logic | Vanilla JavaScript (ES2020) | No frameworks, no bundler |
| QR Codes | Pure SVG, generated in-browser | Deterministic hash-based matrix — visual placeholder |
| Icons | Inline SVG (Heroicons-style) | No icon font dependency |

### Application Layers

```
campuspass.html
│
├── <style>          — CSS custom classes (modal backdrop, drawer animation,
│                      category colour tokens, toast transition)
│
├── <script> block 1 — GLOBAL STATE & PURE LOGIC
│   ├── Helpers       uuid(), esc(), fmtDate(), fmtTime()
│   ├── State{}       currentUser, events[], tickets[], notifications[]
│   ├── Auth          isEduEmail(), login(), logout()
│   ├── Tickets       reserveTickets()
│   ├── Notifications addNotification(), markAllRead()
│   ├── Events        publishEvent(), filteredEvents()
│   ├── Toast         showToast()
│   └── QR            buildQrSvg()
│
├── HTML Sections
│   ├── #toast              — Slide-down success/error banner
│   ├── <header>            — Sticky nav bar, notification bell, auth area
│   ├── <section> hero      — Hero banner + live stats
│   ├── #organizer-panel    — Organizer-only event creation form
│   ├── <main> #events      — Filter pills, search, event card grid
│   ├── #auth-modal         — Login / Register modal
│   ├── #purchase-modal     — Ticket purchase flow (select → confirm)
│   ├── #tickets-drawer     — Slide-over "My Tickets" panel
│   └── <footer>
│
└── <script> block 2 — DOM RENDER & EVENT HANDLERS
    ├── renderAll()         Master re-render (calls all sub-renders)
    ├── renderAuthArea()    Logged-out buttons vs. user avatar menu
    ├── renderEventGrid()   Builds event cards from filteredEvents()
    ├── renderNotifBell()   Updates unread badge count
    ├── renderNotifPanel()  Populates notification dropdown list
    ├── renderTicketsDrawer() Populates My Tickets slide-over
    ├── Auth modal handlers switchAuthTab(), handleAuth()
    ├── Purchase modal      openPurchaseModal(), adjustQty(), handlePurchase()
    └── Organizer form      handlePublishEvent()
```

### Component Map

```
┌─────────────────────────────────────────────────────┐
│  NAV BAR                                            │
│  [CampusPass logo]  [Events] [My Tickets*]          │
│                     [🔔 Bell*]  [User Avatar*]      │
│                                    └─ User dropdown │
│                                         ├─ My Tickets│
│                                         └─ Sign Out  │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  HERO SECTION                                       │
│  Tagline · CTA buttons · Live stats grid            │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  ORGANIZER PANEL  [organizer role only]             │
│  Event creation form (title, category, location,    │
│  date, capacity, price)                             │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  EVENTS SECTION                                     │
│  [Search] [Filter pills: All/Academic/Arts/…]       │
│  ┌──────┐ ┌──────┐ ┌──────┐                        │
│  │ Card │ │ Card │ │ Card │  ← responsive 1/2/3 col │
│  └──────┘ └──────┘ └──────┘                        │
└─────────────────────────────────────────────────────┘

MODALS (overlaid)
  ├── Auth Modal     — Login / Register tabs
  └── Purchase Modal — Inventory bar → qty selector → payment → confirm

SLIDE-OVER (right side)
  └── My Tickets Drawer — ticket cards with SVG QR codes

TOAST  (top center, auto-dismisses after ~4 s)

* visible only when authenticated
```

### Data Flow

```
User action
    │
    ▼
Event handler (e.g. handlePurchase)
    │
    ▼
State mutation (e.g. reserveTickets → State.tickets.push, event.ticketsRemaining--)
    │
    ├──▶ renderEventGrid()   — reflects updated inventory on cards
    ├──▶ addNotification()   ──▶ renderNotifBell()
    └──▶ showToast()         — surface confirmation to user
```

All state is held in the `State` object in JavaScript memory. There is no localStorage persistence — state resets on page reload by design (no backend to restore from).

---

## 3. Features

| Feature | Detail |
|---|---|
| `.edu` email gate | Client-side regex `/^[^\s@]+@[^\s@]+\.edu$/i` on both registration and sign-in |
| Role system | `student` or `organizer` chosen at registration; controls panel visibility and button behaviour |
| Event grid | Filterable by category, searchable by title / location / organizer |
| Inventory tracking | `ticketsRemaining` decrements on purchase; colour-coded availability bar (green → yellow → red) |
| Qty cap | Maximum 2 tickets per student per event, enforced in `reserveTickets()` |
| Mock payment | Two options: Student Account Charge or Credit/Debit Card — no real charge, no data sent |
| Ticket Hash ID | Format `CP-XXXXXXXX` (first 8 hex chars of UUID, uppercased) |
| SVG QR code | Deterministic 21×21 matrix generated from ticket payload; encodes Student ID, Event ID, Purchase Timestamp |
| Toast notifications | Auto-dismissing banner on purchase, login, and event publish |
| In-app notifications | Bell icon with unread badge; notification panel with mark-all-read |
| My Tickets drawer | Slide-over panel showing all user tickets with QR, event details, and payload metadata |
| Organizer panel | Inline form to publish events; new event prepended to grid instantly |
| Responsive layout | 1-column (mobile) → 2-column (sm) → 3-column (lg) event grid; hamburger menu on mobile |

---

## 4. Deploying via GitHub Pages

GitHub Pages serves static files directly from a repository — no server, no cost, no configuration beyond a few clicks.

### Prerequisites

- A free [GitHub account](https://github.com)
- Git installed on your computer (`git --version` to check), **or** just use the GitHub web UI
- The `campuspass.html` file from this repository

### Step-by-Step Deployment

#### Option A — GitHub Web UI (no Git required)

1. **Create a new repository**
   - Go to [github.com/new](https://github.com/new)
   - Name it `campuspass` (or any name you like)
   - Set visibility to **Public** *(GitHub Pages is free on public repos)*
   - Click **Create repository**

2. **Upload the file**
   - On the repository page, click **Add file → Upload files**
   - Drag `campuspass.html` into the upload area
   - In the "Commit changes" section, click **Commit changes**

3. **Rename to `index.html`** *(so GitHub Pages serves it as the root)*
   - Click on `campuspass.html` in the file list
   - Click the pencil icon (Edit) at the top right
   - Change the filename at the top of the editor to `index.html`
   - Click **Commit changes**

4. **Enable GitHub Pages**
   - Go to **Settings** (top tab of your repository)
   - In the left sidebar, click **Pages**
   - Under **Source**, select **Deploy from a branch**
   - Set the branch to `main` and folder to `/ (root)`
   - Click **Save**

5. **Visit your site**
   - After ~60 seconds, a green banner appears with your URL:
     `https://<your-username>.github.io/<repo-name>/`
   - Share this URL with your student body!

---

#### Option B — Git command line

```bash
# 1. Clone or initialise a repo
git init campuspass
cd campuspass

# 2. Copy campuspass.html and rename it
cp /path/to/campuspass.html index.html

# 3. Commit
git add index.html
git commit -m "Initial deploy: CampusPass"

# 4. Push to GitHub (replace <username> and <repo>)
git remote add origin https://github.com/<username>/<repo>.git
git branch -M main
git push -u origin main
```

Then follow **step 4** of Option A to enable Pages in the repository Settings.

---

### Enabling a Custom Domain (optional)

If your student organisation has a domain (e.g. `tickets.techclub.edu`):

1. In your repo, create a file named `CNAME` containing just your domain:
   ```
   tickets.techclub.edu
   ```
2. With your domain registrar, add a `CNAME` DNS record pointing to `<username>.github.io`.
3. In **Settings → Pages**, enter your custom domain and enable **Enforce HTTPS**.

Allow up to 24 hours for DNS propagation.

---

### Updating the Live Site

Every time you push a new commit to the `main` branch, GitHub Pages automatically redeploys within ~30–60 seconds. No manual steps needed.

```bash
# Edit index.html locally, then:
git add index.html
git commit -m "Update: add Spring Concert event"
git push
```

Or use the **pencil (Edit) icon** directly on GitHub to edit in the browser and commit in one step.

---

## 5. Maintenance Guide

All application logic and data live in a single file: `campuspass.html` (or `index.html` after deployment). There is no build step — every change takes effect immediately on reload.

### Event Data Model Reference

Each event in `State.events[]` is a plain JavaScript object with the following fields:

```js
{
  id              : string,   // UUID — auto-generated, do not set manually
  title           : string,   // Display name of the event
  category        : string,   // One of: 'academic' | 'arts' | 'social' | 'sports' | 'tech' | 'other'
  description     : string,   // Short paragraph shown on the event card
  organizer       : string,   // Name of the hosting club or office
  location        : string,   // Venue name or room number
  date            : string,   // ISO 8601 datetime: 'YYYY-MM-DDTHH:MM'
  capacity        : number,   // Total available seats (integer)
  ticketsRemaining: number,   // Starts equal to capacity; decremented on each purchase
  price           : number,   // In USD. Use 0 for free events
  status          : string,   // 'published' | 'draft' | 'cancelled'
}
```

> **Rule:** Only events with `status === 'published'` appear in the public grid.

---

### Adding a New Event Field

**Example:** Add a `tags` array (e.g. `["free food", "networking"]`) to each event.

**Step 1 — Add to seed data** (line ~66 in the file, inside `State.events`)

```js
{ id:uuid(), title:'Spring Hackathon 2025', ...,
  tags: ['prizes', '24hr', 'open to all'] },
```

**Step 2 — Render it in the event card** (inside `eventCardHTML()`)

Find the section that builds the event card's detail rows and add:

```js
// after the organizer row:
${ev.tags && ev.tags.length ? `
  <span class="flex items-center gap-1.5 flex-wrap mt-1">
    ${ev.tags.map(tag => `<span class="bg-gray-100 text-gray-500 text-[10px] px-2 py-0.5 rounded-full">#${esc(tag)}</span>`).join('')}
  </span>` : ''}
```

**Step 3 — Add to the organizer form** (inside `#organizer-form`)

```html
<div class="sm:col-span-2 lg:col-span-3">
  <label class="block text-xs font-semibold text-gray-600 mb-1">Tags (comma-separated)</label>
  <input id="ev-tags" type="text" placeholder="e.g. free food, networking"
    class="w-full border border-gray-300 rounded-lg px-3 py-2 text-sm ..."/>
</div>
```

**Step 4 — Read the field in `handlePublishEvent()`**

```js
function handlePublishEvent(e) {
  e.preventDefault();
  publishEvent({
    ...
    tags: document.getElementById('ev-tags').value
            .split(',').map(t => t.trim()).filter(Boolean),
  });
  ...
}
```

**Step 5 — Pass it through `publishEvent()`**

```js
function publishEvent(data) {
  const ev = {
    ...
    tags: data.tags || [],
    ...
  };
  ...
}
```

That's it — the field is live in both seed events and organizer-created events.

---

### Adding a New Event Category

Categories control filter pills, card colour tags, and the QR drawer's colour band.

**Step 1 — Add the CSS colour token** (in the `<style>` block near the top)

```css
.tag-music { background:#fef9c3; color:#854d0e; }
```

**Step 2 — Add to the organizer form** (`<select id="ev-category">`)

```html
<option value="music">Music</option>
```

**Step 3 — Add a filter pill** (in `#filter-pills`)

```html
<button onclick="setFilter('music')" data-filter="music" class="filter-pill">Music</button>
```

**Step 4 — Add to `catColor()`** (used by the drawer colour band)

```js
function catColor(c) {
  const map = { ..., music: '#854d0e' };
  return map[c] || map.other;
}
```

---

### Editing Seed Events

Seed events are the events pre-loaded when the page opens. They live in the `State.events` array starting around **line 66**.

To add a new seed event, copy an existing object and change the values:

```js
{ id:uuid(), title:'End-of-Year Banquet', category:'social',
  description:'Annual celebration for all graduating seniors. Dinner and awards ceremony.',
  organizer:'Student Government', location:'Grand Ballroom, Student Union',
  date:'2025-12-05T18:30', capacity:400, ticketsRemaining:400, price:15,
  status:'published' },
```

**Important rules when editing seed events:**

| Field | Rule |
|---|---|
| `id` | Always use `uuid()` — never hardcode a string |
| `ticketsRemaining` | Set equal to `capacity` for brand-new events |
| `status` | Must be `'published'` to appear in the grid |
| `date` | Must be `'YYYY-MM-DDTHH:MM'` format (HTML datetime-local compatible) |
| `price` | Use `0` (number, not string) for free events |

To **hide** a seed event without deleting it, set `status: 'draft'`.

---

### Changing the `.edu` Validation Rule

The `.edu` check is a single regex in the `isEduEmail()` function:

```js
function isEduEmail(e) {
  return /^[^\s@]+@[^\s@]+\.edu$/i.test(e.trim());
}
```

**Restrict to a specific university:**

```js
// Only allow @state.edu
function isEduEmail(e) {
  return /^[^\s@]+@state\.edu$/i.test(e.trim());
}
```

**Allow multiple domains:**

```js
const ALLOWED_DOMAINS = ['state.edu', 'community.edu', 'tech.edu'];
function isEduEmail(e) {
  const domain = e.trim().split('@')[1]?.toLowerCase();
  return !!domain && ALLOWED_DOMAINS.includes(domain);
}
```

**Allow any `.edu` plus a partner domain:**

```js
function isEduEmail(e) {
  return /^[^\s@]+@([^\s@]+\.edu|partnercollege\.org)$/i.test(e.trim());
}
```

The same `isEduEmail()` function is called in both the **auth modal handler** (`handleAuth`) and can be reused anywhere else in the file.

---

### Extending the Ticket Model

Each ticket in `State.tickets[]` has this shape:

```js
{
  id              : string,   // UUID
  eventId         : string,   // FK → event.id
  userId          : string,   // FK → State.currentUser.id
  hashId          : string,   // Display ID: 'CP-XXXXXXXX'
  qrPayload       : string,   // JSON string encoded into the QR SVG
  payMethod       : string,   // 'student_account' | 'credit_card'
  status          : string,   // 'reserved' | 'cancelled'
  reservedAt      : string,   // ISO 8601 timestamp
}
```

**The `qrPayload` JSON encodes:**

```json
{
  "studentId"        : "<user UUID>",
  "eventId"          : "<event UUID>",
  "purchaseTimestamp": "<ISO 8601>",
  "ticketId"         : "<ticket UUID>",
  "seq"              : 1
}
```

To add a new field to the QR payload (e.g. `seatNumber`), edit `reserveTickets()`:

```js
qrPayload: JSON.stringify({
  studentId: State.currentUser.id,
  eventId,
  purchaseTimestamp: now,
  ticketId: tid,
  seq: i + 1,
  seatNumber: assignSeat(ev),   // ← add here
}),
```

Then display it in `renderTicketsDrawer()` inside the monospace metadata block:

```js
<div><span class="text-gray-500">Seat:</span> ${qrData.seatNumber}</div>
```

---

### Theming & Branding

The brand colour palette is defined in the Tailwind config block at the top of the file:

```js
tailwind.config = {
  theme: {
    extend: {
      colors: {
        brand: {
          DEFAULT : '#4f46e5',  // primary indigo — buttons, links, active states
          light   : '#818cf8',  // hero gradient highlight
          dark    : '#3730a3',  // hover states, hero gradient base
        }
      }
    }
  }
}
```

Change all three values to rebrand. For example, a green theme for an environmental club:

```js
brand: { DEFAULT:'#16a34a', light:'#4ade80', dark:'#15803d' }
```

To change the **page title and browser tab name**, edit line 6:

```html
<title>YourClub Events — Ticketing</title>
```

To change the **nav bar brand name**, find `CampusPass` inside the `<header>` and update the text node.

---

## 6. Known Limitations

| Limitation | Reason | Workaround |
|---|---|---|
| State resets on page reload | No backend or localStorage | Export tickets as JSON; integrate with a serverless backend (Supabase, Firebase) for persistence |
| QR codes are visual placeholders | No real QR encoding library loaded from CDN | Replace `buildQrSvg()` with [qrcode.js](https://github.com/davidshimjs/qrcodejs) for scannable codes |
| `.edu` check is client-side only | No server to verify email existence | Add email verification flow with a backend (e.g. a Cloudflare Worker sending a magic link) |
| All users share the same session view | Single-page, no server auth | Separate user stores or add localStorage-based session tokens for a multi-user demo |
| Payment is mock-only | By design | Integrate Stripe.js or a payment API for real transactions |
| No image uploads | Front-end only | Add a URL field to events and render it as a card banner image |

---

## 7. Roadmap

Potential next steps if the project grows:

- [ ] **LocalStorage persistence** — save `State.events` and `State.tickets` between sessions
- [ ] **CSV/JSON export** — let organizers download ticket lists
- [ ] **Real QR encoding** — swap placeholder SVG for a `qrcode.js` implementation
- [ ] **Email confirmation** — connect to a serverless function (Cloudflare Worker / Vercel Edge) for real `.edu` verification and confirmation emails
- [ ] **Backend migration** — follow the [`campuspass-plan.md`](campuspass-plan.md) roadmap to add a Node.js/Express API and PostgreSQL database
- [ ] **Seat maps** — visual seat selection for fixed-venue events
- [ ] **Waitlist** — queue students when capacity is 0

---

## License

This project is released for educational and demonstration purposes.  
Free to use, fork, and adapt for any student organisation.

---

*Built with ❤️ for campuses everywhere.*
