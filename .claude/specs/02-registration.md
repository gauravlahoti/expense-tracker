# Spec: Registration

## Overview

Turn the existing `/register` view into a working sign-up flow.
Users must be able to submit the registration form, have their
credentials persisted to the `users` table with a hashed
password, and be redirected to the login page on success. This is
Step 02 in the Spendly roadmap and is the first feature that
actually writes user-supplied data to the database created in
Step 01. It enables every later authenticated feature
(login, profile, expense CRUD).

---

## Depends on

- **Step 01 — Database setup** (`users` table, `get_db()`,
  `init_db()`, `seed_db()` already implemented in
  `database/db.py`).

---

## Routes

- `GET  /register` — already exists; renders `register.html`.
  Will be extended to accept an optional `error` context var
  (template already supports it). Public.
- `POST /register` — **new**; processes the submitted form,
  validates input, inserts the user, and redirects to
  `/login` on success. Public.

The current single-method `register()` view in `app.py` will be
updated to accept both `GET` and `POST`.

---

## Database changes

No database changes. The existing `users` table already has
`id`, `name`, `email` (UNIQUE), `password_hash`, and
`created_at` — verified against `database/db.py`.

---

## Templates

- **Create:** none.
- **Modify:** none required for the happy path.
  `templates/register.html` already posts to `/register`,
  has `name`/`email`/`password` fields with `required`, and
  already renders an `{% if error %}` flash block. The view
  just needs to populate `error` on failures.

---

## Files to change

- `app.py`
  - Import `request`, `redirect`, `url_for` from `flask`.
  - Import `generate_password_hash` from `werkzeug.security`.
  - Import `sqlite3` (to catch `IntegrityError` on duplicate
    email).
  - Replace the existing `register()` view with one that
    accepts `methods=["GET", "POST"]` and implements the
    server-side logic described below.

## Files to create

- None.

---

## New dependencies

No new dependencies. `werkzeug` is already in
`requirements.txt`.

---

## Rules for implementation

- No SQLAlchemy or ORMs.
- Parameterised queries only — never f-strings or `%`
  interpolation in SQL.
- Passwords hashed with `werkzeug.security.generate_password_hash`
  before insert. Never store plaintext.
- Use the existing `get_db()` helper; do not open raw
  `sqlite3.connect()` calls in `app.py`.
- Close the connection in every code path (success, validation
  failure, integrity error).
- Trim whitespace on `name` and `email`; lowercase the email
  before insert and uniqueness check.
- Server-side validation, even though the form has
  `required`:
  - All three fields must be non-empty after trimming.
  - `email` must contain an `@`.
  - `password` must be at least 8 characters.
- On any validation or duplicate-email failure, re-render
  `register.html` with a human-readable `error` string and
  HTTP `400`.
- On success, redirect to `url_for("login")` with HTTP `302`
  (use Flask's default redirect status).
- Catch `sqlite3.IntegrityError` for the UNIQUE email
  constraint and surface a friendly message
  ("An account with that email already exists.").
- Use CSS variables — never hardcode hex values
  (no template/CSS changes are expected, but this rule
  applies if any are added).
- All templates extend `base.html` (already true).
- Do not log or echo back the submitted password in any form.
- Do not auto-login the user after registration; that is
  Step 03.

---

## Definition of done

- [ ] Visiting `GET /register` still renders the form
      unchanged.
- [ ] Submitting the form with a new name/email/password
      inserts a row in `users` with a hashed
      `password_hash` (verifiable via
      `sqlite3 spendly.db "SELECT email, password_hash FROM users;"`).
- [ ] After a successful registration the browser is
      redirected to `/login` (HTTP 302 → 200).
- [ ] Submitting an email that already exists (e.g.
      `demo@spendly.com`) re-renders the form with a
      visible error and does **not** create a duplicate row.
- [ ] Submitting a password shorter than 8 characters
      re-renders the form with a clear error.
- [ ] Submitting an email without `@` re-renders the form
      with a clear error.
- [ ] Submitting with any blank field (after trimming)
      re-renders the form with a clear error.
- [ ] Stored `password_hash` values start with the werkzeug
      prefix (e.g. `scrypt:` or `pbkdf2:`) — confirming the
      password was hashed, not stored as plaintext.
- [ ] `app.py` contains no string-formatted SQL; the insert
      uses `?` placeholders.
- [ ] App starts without errors and existing routes
      (`/`, `/login`, placeholder routes) still respond.
