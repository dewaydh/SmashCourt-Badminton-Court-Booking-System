# 🏸 SmashCourt — Badminton Court Booking System

A simple **role-based** web app built with **Laravel 12 + PHP 8.2**.
Created as a case study for a **Manual Testing (SIT) vs AI Assisted Code Review** assignment.

## Roles (Point of View)

| Role | Who | Can do |
|------|-----|--------|
| **Admin** | Court owner (the one who rents courts out) | Manage courts (add/edit/delete), view **all** bookings, change booking status, delete bookings, see dashboard stats |
| **Customer** | Renter | Browse active courts, create bookings, view **their own** booking history, cancel a booking while it is still "Booked" |

Flow: **Register / Login → routed by role → do the work → Logout.**
Public sign-up always creates a **Customer**. Admin accounts are created by the seeder.

## Modules

1. **Court Management** (admin) — CRUD for courts (name, type, price/hour, status)
2. **Make a Booking** (customer) — booking form with validation, overlap check, and automatic price calculation
3. **Booking History** — customers see their own bookings; admin sees all bookings with status control

## Tech Stack

- **Laravel 12** (PHP ^8.2)
- **MySQL** (default — works great with Laragon). SQLite supported as a fallback.
- **Blade** + plain CSS (no Node / build step)
- Built-in session authentication + a custom `role` middleware

---

## 🚀 Getting Started (with Laragon + MySQL)

Requirements: **PHP ≥ 8.2**, **Composer**, and **MySQL running** (e.g. via Laragon → *Start All*).

```bash
# 1. Enter the project folder
cd badminton-booking

# 2. Install dependencies
composer install

# 3. Create the environment file
cp .env.example .env

# 4. Generate the app key (REQUIRED — without it, login/forms break)
php artisan key:generate
```

**5. Create the database.** In Laragon: open **HeidiSQL** (Menu → MySQL → HeidiSQL) and create a database named **`badminton_booking`** (must match `DB_DATABASE` in `.env`).

```bash
# 6. Run migrations + seed demo data (users + courts)
php artisan migrate --seed

# 7. Start the server
php artisan serve
```

Open **http://localhost:8000**. You'll be redirected to the login page.

> Tip: if you place the project inside `C:\laragon\www\`, Laragon auto-creates `http://badminton-booking.test` pointing at `/public` — no need for `php artisan serve`.

---

## 🔑 Demo Accounts (after `migrate --seed`)

| Role | Email | Password |
|------|-------|----------|
| Admin | `admin@smashcourt.test` | `password` |
| Customer | `customer@smashcourt.test` | `password` |

Seeded courts (price range around **Rp 25.000/hour**):

- Court A — Indoor — Rp 25.000 — Active
- Court B — Indoor — Rp 25.000 — Active
- Court C — Outdoor — Rp 30.000 — Active
- Court D — Indoor — Rp 35.000 — Active
- Court E (VIP) — Indoor — Rp 40.000 — **Inactive**

---

## 🗄️ Prefer SQLite instead of MySQL?

No DB server needed. Edit `.env`:

```env
DB_CONNECTION=sqlite
```

Remove (or ignore) the `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` lines, then:

```bash
# Windows
type nul > database\database.sqlite
# Mac/Linux
touch database/database.sqlite

php artisan migrate --seed
```

---

## ⬆️ Push to GitHub

```bash
git init
git add .
git commit -m "Initial commit: SmashCourt booking system"

git remote add origin https://github.com/USERNAME/badminton-booking.git
git branch -M main
git push -u origin main
```

> `vendor/`, `.env`, and `*.sqlite` are ignored on purpose (see `.gitignore`).
> Anyone who clones the repo just repeats the **Getting Started** steps.

---

## 🔒 Access Rules (useful for Test Cases)

- Guests can only reach `/login` and `/register`. Any protected page redirects to login.
- Customers **cannot** reach admin pages (`/dashboard`, `/courts`, `/admin/bookings`) → **403**.
- Admins are not given the customer booking screens.
- A customer can only cancel **their own** booking, and only while status is **Booked**.

## ✅ Validation Rules (useful for Test Cases)

**Court (admin)**
- `name` required, max 100 chars
- `type` required (Indoor / Outdoor)
- `price_per_hour` required, integer, ≥ 0
- `status` required (Active / Inactive)

**Booking (customer)**
- `court_id` required, must exist, and the court must be **Active**
- `date` required, **cannot be in the past**
- `end_time` must be **after** `start_time`
- the time slot **must not overlap** another booking on the same court & date (cancelled ones ignored)
- `total_price` = duration (hours) × price per hour (auto)

**Auth**
- Register: `name`, valid unique `email`, `password` min 6 + confirmation
- Login: wrong credentials show an error and do not log in

---

## 📁 Project Structure

```
badminton-booking/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Auth/AuthController.php       # login / register / logout
│   │   │   ├── Admin/BookingController.php   # admin: all bookings
│   │   │   ├── CourtController.php           # admin: Module 1
│   │   │   ├── BookingController.php         # customer: Module 2 & 3
│   │   │   └── DashboardController.php       # admin dashboard
│   │   └── Middleware/EnsureUserHasRole.php  # role-based access
│   └── Models/ (User, Court, Booking)
├── database/migrations/                      # users, courts, bookings
├── database/seeders/DatabaseSeeder.php       # demo users + courts
├── resources/views/                          # Blade (auth, courts, bookings, admin)
├── routes/web.php
└── public/css/app.css
```

---

## ⚠️ Version Issues?

If `composer install` fails because of a PHP/Laravel version mismatch, create a fresh Laravel project that matches your PHP and copy these folders into it:
`app/`, `database/`, `routes/web.php`, `resources/views/`, `public/css/`, plus the `config/auth.php` and `bootstrap/app.php` changes.

---

*Built for academic use — feel free to adapt it for your group's needs.*
