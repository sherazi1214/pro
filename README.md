# AI Powered BabyPaws Care (AIBPC)

A full-stack platform connecting parents/pet owners with verified babysitters and
pet caregivers — built with **Django + Django REST Framework** (backend) and
**React + Vite + Bootstrap 5** (frontend).

```
AIBPC/
├── backend/     Django REST API (accounts + hiring apps)
└── frontend/    React (Vite) single-page app
```

---

## 1. Backend setup (Django)

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

python manage.py migrate
python manage.py createsuperuser   # or use the seed command below
python manage.py seed_data         # optional: creates demo admin/parent/caregivers
python manage.py runserver         # http://127.0.0.1:8000
```

### Demo accounts created by `seed_data`
The seed command intentionally does **not** contain default passwords. Set these in `backend/.env` before running it:

```env
SEED_ADMIN_PASSWORD=choose-a-strong-admin-password
SEED_DEMO_PASSWORD=choose-a-strong-demo-password
```

The command creates the demo admin/parent/caregiver accounts using those passwords.

### Database: MySQL only (no SQLite)
This project uses **MySQL exclusively** — there is no SQLite fallback in
development or production. Django refuses to start with a clear error if
the required `DB_*` environment variables aren't set.

1. Install the MySQL driver (already in `requirements.txt`): `pip install mysqlclient`
   (Ubuntu/Debian first needs `sudo apt install default-libmysqlclient-dev build-essential pkg-config`)
2. In MySQL Workbench (or the `mysql` CLI), create a database, e.g. `aibpc_db`.
3. Copy `backend/.env.example` to `backend/.env` and fill in all five variables:
   ```
   DB_NAME=aibpc_db
   DB_USER=root
   DB_PASSWORD=yourpassword
   DB_HOST=localhost
   DB_PORT=3306
   ```
   `.env` is loaded automatically (via `python-dotenv`) every time you run
   `manage.py` — no need to `export` the variables by hand.
4. Run `python manage.py migrate`.
5. Sanity-check the connection any time with `python manage.py check_db`.

### Connecting to a shared MySQL database from two laptops
If MySQL Server is installed on Laptop A and you want Laptop B's Django app
to use that *same* database (so both laptops see the same data):

1. **On Laptop A (where MySQL runs):**
   - Find its LAN IP address (e.g. `192.168.1.23`) — both laptops must be on
     the same network/Wi-Fi.
   - Allow remote connections by editing MySQL's config file
     (`mysqld.cnf` / `my.ini`, usually under `[mysqld]`) and changing:
     ```
     bind-address = 0.0.0.0
     ```
     then restart the MySQL service.
   - Create a MySQL user allowed to connect from *any* host (not just
     `localhost`), and grant it access:
     ```sql
     CREATE USER 'aibpc_user'@'%' IDENTIFIED BY 'change-me';
     GRANT ALL PRIVILEGES ON aibpc_db.* TO 'aibpc_user'@'%';
     FLUSH PRIVILEGES;
     ```
   - Open port `3306` on Laptop A's firewall (Windows Defender Firewall →
     Advanced Settings → Inbound Rules → New Rule → Port → TCP 3306; on
     Ubuntu, `sudo ufw allow 3306`).
2. **On Laptop B (this laptop, running Django):**
   - Copy `backend/.env.example` to `backend/.env` and set:
     ```
     DB_NAME=aibpc_db
     DB_USER=aibpc_user
     DB_PASSWORD=change-me
     DB_HOST=192.168.1.23   # Laptop A's LAN IP, NOT "localhost"
     DB_PORT=3306
     ```
   - Run `python manage.py check_db` — it should print
     `✓ Connected successfully. MySQL version: ...`. If it fails, the error
     message lists the most common causes (firewall, bind-address, user
     host permissions, etc.).
   - Run `python manage.py migrate` (only needs to be done once — the
     tables now live on Laptop A's server, shared by both machines) and
     `python manage.py runserver` as usual.
3. Repeat step 2's `.env` setup on Laptop A too (with `DB_HOST=localhost`)
   if you want Laptop A's own Django instance to use the same database.

### Key API endpoints
| Endpoint | Method | Description |
|---|---|---|
| `/api/auth/register/parent/` | POST | Register a parent/pet owner |
| `/api/auth/register/caregiver/` | POST | Register a caregiver |
| `/api/auth/login/` | POST | Login with `email` or `cnic` + `password` |
| `/api/auth/me/` | GET/PATCH | Current user profile |
| `/api/auth/caregivers/` | GET | Paginated (10/page) list/search of caregivers — `?page=`, `?search=`, `?service=`. Response: `{data: [...], pagination: {...}}` |
| `/api/auth/caregivers/<id>/` | GET | Caregiver public profile |
| `/api/auth/favorites/` | GET/POST/DELETE | Parent's saved caregivers |
| `/api/auth/admin/users/` | GET | Admin: list users for verification |
| `/api/auth/admin/users/<id>/verify/` | POST | Admin: approve/reject documents |
| `/api/requests/` | GET/POST | Hiring requests. Backend rejects POSTs targeting a Hired caregiver |
| `/api/requests/<id>/respond/` | POST | Caregiver accepts/rejects. Accept is transaction-locked to prevent double-booking, and sets the caregiver to Hired |
| `/api/requests/<id>/complete/` | POST | Mark a hiring request completed — auto-frees the caregiver and auto-generates the payment slip |
| `/api/tasks/` | GET/PATCH | Caregiver tasks (pending/completed) |
| `/api/reviews/` | GET/POST | Ratings & reviews |
| `/api/payments/` | GET | The auto-generated payment slip(s) for the logged-in user |
| `/api/payments/<id>/slip/` | GET | The printable/downloadable payment slip (same data, dedicated URL) |
| `/api/payments/<id>/received/` | POST | Caregiver confirms payment was received |
| `/api/payments/<id>/not-sure/` | POST | Caregiver isn't sure — requests the parent upload a transaction slip |
| `/api/payments/<id>/verify/` | POST | Caregiver/admin verifies an uploaded transaction slip |
| `/api/payment-proofs/` | GET/POST | Parent-uploaded transaction slip, linked to a payment |
| `/api/requests/<id>/request_receipt/` | POST | Caregiver asks for a receipt (before any proof uploaded) |
| `/api/auth/care-requirement/` | GET/POST/PATCH | Parent's one-time (editable) preferences form |
| `/api/auth/caregiver-suggestions/` | GET | AI-suggested caregivers based on the saved preferences |
| `/api/auth/cnic-ocr/` | POST | Upload a CNIC photo, get back auto-extracted fields |
| `/api/auth/notifications/` | GET | List the logged-in user's notifications |
| `/api/auth/notifications/<id>/mark_read/` | POST | Mark one notification read |
| `/api/auth/notifications/mark_all_read/` | POST | Mark all notifications read |

Admin panel: `http://127.0.0.1:8000/admin/`

---

## 2. Frontend setup (React + Vite)

```bash
cd frontend
npm install
npm run dev      # http://localhost:5173
```

The frontend talks to the backend at `http://127.0.0.1:8000/api` by default.
To change this, create a `.env` file in `frontend/`:

```
VITE_API_URL=http://127.0.0.1:8000/api
```

Build for production:
```bash
npm run build      # outputs to frontend/dist
```

---

## 3. What's included

- **Landing page** with hero, services, "how it works", featured caregivers, and CTA.
- **Registration**: separate forms for Parent/Pet Owner and Caregiver, with CNIC
  and police-certificate document upload.
- **Login**: email or CNIC + password, JWT-based sessions (auto refresh).
- **Parent dashboard**: profile, favorites, hiring history, auto-generated
  payment slips (view/print, upload transaction slip when verification is
  requested).
- **Caregiver dashboard**: hiring requests (accept/reject with parent contact
  details shown, backend-enforced against double-booking), Free/Hired status
  badge, tasks (pending/completed), payment slips (Received / Not Sure About
  Payment / Verify), reviews, FAQ.
- **Admin dashboard**: user management and CNIC/police-certificate document
  verification (approve/reject).
- **Caregiver public profile**: rating, bio, services, reviews, Free/Hired
  status badge, and an "Apply for Service" form that's disabled while the
  caregiver is Hired (profile stays viewable either way).

## 4. Payment slip & caregiver availability workflow (latest change request)

- **Database is MySQL only.** No SQLite anywhere, in dev or production —
  see section 1's "Database: MySQL only" above.
- **Caregiver Free/Hired status** (`CaregiverProfile.status`) is the
  server-side source of truth for whether a caregiver can be hired:
  - Accepting a request flips the caregiver to Hired inside a DB
    transaction with `select_for_update()`, so two parents racing to hire
    the same caregiver can't both succeed.
  - The backend rejects `POST /api/requests/` targeting an already-Hired
    caregiver (not just a disabled frontend button).
  - A Hired caregiver's profile stays fully visible; only the hire action
    is blocked.
  - When a service completes, the caregiver automatically flips back to
    Free.
- **Automatic payment slip.** The moment a service completes (auto-ended
  on schedule, or via the `complete` action), a `Payment` record is
  created automatically with a unique `slip_number` (e.g.
  `AIBPC-202608-3DF9CCAB`), the computed amount, and status `Pending`. It's
  immediately visible on both the parent's and caregiver's dashboards, and
  printable/downloadable from the parent's Payments tab.
- **Received / Not Sure workflow.** From their Payments tab the caregiver
  clicks **Received** (status → Received) or **Not Sure About Payment**
  (status → Verification Required, and the parent is notified to upload
  their transaction slip). Once the parent uploads it, the caregiver (or
  an admin) can **Verify** it (status → Verified).
- **Caregiver pagination.** `GET /api/auth/caregivers/` now paginates
  server-side at exactly 10 per page, returning
  `{data, pagination: {currentPage, itemsPerPage, totalItems, totalPages,
  hasNextPage, hasPreviousPage}}` — the frontend's caregiver list renders
  Previous/Next controls from this directly.

## 5. Earlier updates

**Auto-ending services & payments**
- A hired service now auto-completes the moment its selected end date/time
  passes (checked lazily whenever hiring requests are fetched, and also via
  `python manage.py complete_due_services`, safe to run on a cron/celery-beat
  schedule every few minutes).
- Once completed, the parent's dashboard shows the exact amount due,
  calculated automatically from the caregiver's rate (hourly/weekly/yearly)
  and the number of days/hours originally selected when hiring.
- The parent uploads a payment proof; the caregiver taps **Received** to
  confirm, which notifies the parent. If payment hasn't arrived, the
  caregiver can tap **Ask for receipt** instead, which also notifies the
  parent.
- After a booking is completed, the parent can leave a star rating & review,
  which is saved automatically to the caregiver's public profile (already
  existed, now surfaced directly from the completed-booking card).
- A notification bell (top navbar) shows these events in real time (polls
  every 30s) for both parents and caregivers.

**AI-suggested caregivers (one-time preferences form)**
- The first time a Parent/Pet Owner logs in after admin approval, they're
  taken to `/parent/preferences` to fill a short form: Location, Pricing
  Range (Rs 1,000–5,000), Service Plan (Daily/Weekly/Monthly/Yearly), and
  Service Type. It can be edited any time afterwards from the navbar or
  dashboard.
- Submitting it calls a rule-based matching engine
  (`backend/accounts/matching.py`) that scores every approved, available
  caregiver on rating, price fit, experience, review volume, and exact
  service-type match, then returns the best matches with a `match_score`
  (0–100) shown on each suggested caregiver card.

**Extra registration fields + AI-powered CNIC auto-fill**
- Registration (both Parent/Pet Owner and Caregiver) now also collects
  **Date of Birth**, **CNIC Issue Date**, and **CNIC Expiry Date** (expiry
  must be in the future; must be 18+ years old).
- An optional "Auto-fill from your CNIC" uploader on both forms sends the
  photo to `/api/auth/cnic-ocr/`, which pre-fills the name, CNIC number, and
  the three dates (the user should still double-check them before
  submitting).

**How extraction works — AI first, local OCR as fallback**
1. If `GROQ_API_KEY` is set in `backend/.env`, the photo is sent to Groq's
   (free) vision model (`accounts/groq_ocr.py`), which reads the card's
   actual field labels (Date of Birth / Issue / Expiry) the way a person
   would — this is the accurate, recommended path.
2. If that key isn't set, or the AI call fails for any reason, it falls back
   automatically to a local `pytesseract` pipeline (`accounts/ocr.py`). The
   failure reason is written to the Django log, so a wrong key / rate limit is
   no longer silent.

   How the local fallback reads the card:
   - Dates are matched to their printed label **by position** (value under or
     beside its label). This fixes the two-column CNIC layout, where the label
     row and the value row are separate lines and the old regex gave the
     *issue* date as the *expiry* date.
   - Month-first dates such as `12.31.2001` are accepted (day-first is tried
     first).
   - Several OCR passes are combined by voting; the name is cleaned of OCR
     debris (`— Sahil Khan ve 3` -> `Sahil Khan`).
   - Results are sanity-checked (birth < issue < expiry); anything that cannot
     be resolved is left blank for the user to type, never guessed.
   - If something is still missing it retries once with a slight tilt
     correction (phone photos are rarely perfectly straight). Worst case this
     takes ~20 s; a normal read takes ~8 s.
   - `TESSERACT_CMD` is only used if the file exists; otherwise `tesseract` is
     looked up on PATH (and the default Windows install folders).

**Enabling the AI path (recommended, free)**
1. Get a free API key at https://console.groq.com/keys (no card required).
2. In `backend/.env`, set:
   ```
   GROQ_API_KEY=gsk_...
   ```
3. `pip install groq` (already added to `requirements.txt`).
4. Optional: `GROQ_CNIC_MODEL` (default `qwen/qwen3.8-27b`). Reasoning is switched
   off for this call so the model's "thinking" tokens can't truncate the JSON.

**Enabling the local OCR fallback**
The fallback needs both the `pytesseract` Python package (already in
`requirements.txt`) **and** the Tesseract OCR engine binary installed on the
machine running Django:
```bash
# Ubuntu/Debian
sudo apt install tesseract-ocr
# macOS
brew install tesseract
# Windows: install from https://github.com/UB-Mannheim/tesseract/wiki
```
If neither the AI key nor Tesseract is available, the endpoint returns a
clear error instead of crashing, and the rest of the app keeps working —
the fields can simply be filled in by hand.

## 5. Where this app is (and isn't) actually using AI

Being upfront about this, since it matters for an "AI Powered" product:

| Feature | Real AI, or rule-based? |
|---|---|
| CNIC photo auto-fill | **Real AI** (Groq vision, free) when `GROQ_API_KEY` is set — see above. Falls back to plain OCR + regex otherwise. |
| "AI caregiver matching" (`accounts/matching.py`, the suggestions endpoint) | **Rule-based scoring today** — a transparent formula (rating × weight + price-range fit + experience + reviews + service match), no model call. It works and is explainable, but it isn't AI. |

If you want the matching feature to also use real AI (e.g. Claude reasoning
over each parent's requirement and caregiver profiles to rank/explain
matches in natural language, not just a numeric score), that's a separate,
well-scoped change — say the word and it can be built the same way the CNIC
extraction above was.

## 6. Notes
- Uploaded documents/photos are stored under `backend/media/` in development.
- JWT access tokens last 1 day, refresh tokens 7 days (see `SIMPLE_JWT` in
  `backend/babypaws/settings.py`).
- This is a development-ready scaffold: before deploying, set `DEBUG=False`,
  configure `ALLOWED_HOSTS`, use a real `SECRET_KEY`, and serve media via
  proper storage (e.g. S3) rather than Django's dev server.# pro
