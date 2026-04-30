# Spec: Login and Logout

## Overview

Wire up real session-based authentication. Step 02 lets users
register; this step lets them sign in with those credentials,
keeps them signed in across requests via Flask's session
cookie, and gives them a working sign-out. After this step,
later features (profile, expense CRUD) can rely on a
`current_user` being available and reject anonymous traffic.
The Spendly navbar will also start reflecting auth state — a
signed-in user sees their name and a Sign out link instead of
the Sign in / Get started CTAs.

---

## Depends on

- **Step 01 — Database setup**: `users` table with
  `id`, `name`, `email` (UNIQUE), `password_hash`, and the
  `get_db()` helper.
- **Step 02 — Registration**: `POST /register` inserts users
  with werkzeug-hashed passwords, lowercased email. Step 03
  must mirror that lowercase normalisation when looking up by
  email so the demo seed user and any newly registered user
  can both sign in.

---

## Routes

- `GET  /login` — already exists; renders `login.html`. Will
  be extended to optionally pass an `error` to the template
  (already supported by `templates/login.html:16-18`). Public.
  If the user is already signed in, redirect to `/profile`.
- `POST /login` — **new**; validates credentials, sets
  `session["user_id"]`, redirects to `/profile` on success or
  re-renders `login.html` with HTTP 401 on failure. Public.
- `GET  /logout` — currently a placeholder
  (`app.py:71-73`); replace with a handler that clears the
  session and redirects to `/`. Logged-in (no-op for
  anonymous; still redirects to `/`).

---

## Database changes

No database changes. The `users.password_hash` column already
holds werkzeug hashes from Step 02 and the seed user, so
`check_password_hash` covers both.

---

## Templates

- **Create:** none.
- **Modify:**
  - `templates/base.html` — make the navbar auth-aware.
    When a `current_user` is exposed (via context processor),
    show "`Hi, {{ current_user.name }}`" and a "Sign out" link
    pointing at `url_for('logout')`. When anonymous, keep the
    existing "Sign in" + "Get started" links. No new icons or
    colours; reuse existing `nav-links` / `nav-cta` classes.
  - `templates/login.html` — no structural change; the
    `{% if error %}` block already renders failures.

---

## Files to change

- `app.py`
  - Import `session` and `flash`-free helpers from `flask`
    (`session` only — Step 02 already covers
    `request`/`redirect`/`url_for`).
  - Import `check_password_hash` from `werkzeug.security`
    (alongside the existing `generate_password_hash` import).
  - Set `app.secret_key` from the `SPENDLY_SECRET_KEY` env
    var, falling back to a development default
    (`"dev-secret-change-me"`) so the app still boots locally
    without configuration. Document the env var in the spec
    here, not in code comments.
  - Replace the existing `login()` view with one that
    accepts `methods=["GET", "POST"]`, validates the
    submitted credentials, and stores `user_id` in
    `session`. On `GET`, redirect already-authenticated
    users to `/profile`.
  - Replace the placeholder `logout()` view with one that
    calls `session.pop("user_id", None)` and redirects to
    `url_for("landing")`.
  - Add a `@app.context_processor` named `inject_current_user`
    that looks up the signed-in user (when
    `session.get("user_id")` is set) and exposes
    `current_user` to every template. Returns `None` when
    anonymous. Closes the connection in every code path.
- `templates/base.html`
  - Replace the static `nav-links` block with a Jinja
    `{% if current_user %}` / `{% else %}` switch. Anonymous
    branch keeps today's two links unchanged.

## Files to create

- None.

---

## New dependencies

No new dependencies. `flask.session` and
`werkzeug.security.check_password_hash` are already available
through the packages pinned in `requirements.txt`.

---

## Rules for implementation

- No SQLAlchemy or ORMs. Use `get_db()` and parameterised
  queries.
- Parameterised queries only — `?` placeholders, never
  string formatting.
- Passwords compared with
  `werkzeug.security.check_password_hash`. Never compare
  hashes with `==`, never trim or transform the submitted
  password before comparison (only `email` is normalised).
- Email lookup must lowercase the submitted value
  (`request.form.get("email", "").strip().lower()`) so it
  matches what Step 02 stored.
- Use a single generic error message for invalid email and
  invalid password ("Invalid email or password."). Do not
  leak which one was wrong. Validation-level errors
  (blank fields) may use a distinct message
  ("Email and password are required.").
- HTTP `401` on auth failure, HTTP `400` on blank-field
  validation failure, HTTP `302` redirect on success.
- Session cookie defaults are sufficient for this step
  (`HttpOnly` is on by default in Flask). Do **not** add
  `SESSION_COOKIE_SECURE=True` — it would break local
  `http://` testing; defer to a deployment step.
- `app.secret_key` must come from
  `os.environ.get("SPENDLY_SECRET_KEY", "dev-secret-change-me")`
  so production can override without code changes.
- The context processor must close its DB connection on every
  path (success, missing user, exception).
- If `session["user_id"]` points at a row that no longer
  exists (e.g. user deleted), the context processor must
  treat the session as anonymous **and** clear the stale key
  via `session.pop("user_id", None)`.
- Use CSS variables — never hardcode hex values. (No CSS
  edits expected; rule applies if any are added.)
- All templates extend `base.html` (already true for
  `login.html`).
- Do not auto-login a freshly registered user as part of this
  step — registration → login redirect from Step 02 stays
  unchanged. (Optional follow-up; out of scope.)
- Do not add CSRF tokens yet. Note in code review that this
  is a known gap to revisit when the app leaves
  teaching/local-only mode.

---

## Definition of done

- [ ] `app.secret_key` is set; the app boots without
      `RuntimeError: The session is unavailable...` when
      `session["user_id"]` is read.
- [ ] `GET /login` while anonymous renders the form
      unchanged.
- [ ] `GET /login` while signed in (cookie present) redirects
      to `/profile` (HTTP 302). The placeholder profile
      response from `app.py:76-78` is acceptable as the
      target.
- [ ] Submitting valid credentials for the seeded demo user
      (`demo@spendly.com` / `demo123`) redirects to
      `/profile` and sets a `session` cookie.
- [ ] Submitting valid credentials for a user created via
      `POST /register` (Step 02) also succeeds.
- [ ] Email lookup is case-insensitive: signing in as
      `DEMO@spendly.com` works.
- [ ] Submitting a wrong password for an existing email
      re-renders `login.html` with "Invalid email or
      password." and HTTP 401.
- [ ] Submitting a non-existent email returns the same
      generic "Invalid email or password." error and HTTP
      401 (no user enumeration).
- [ ] Submitting a blank email or password re-renders the
      form with "Email and password are required." and HTTP
      400.
- [ ] After signing in, the navbar in `base.html` shows
      "Hi, Demo User" and a "Sign out" link instead of
      "Sign in" / "Get started" on every authenticated page.
- [ ] `GET /logout` clears `session["user_id"]` and
      redirects to `/`. The next request shows the anonymous
      navbar again.
- [ ] `GET /logout` while already anonymous still redirects
      to `/` (does not 500).
- [ ] If the database is reset between requests (deleting
      the user behind a live cookie), the next request
      renders the anonymous navbar instead of crashing, and
      the stale `user_id` is removed from the session.
- [ ] `app.py` contains no string-formatted SQL; the user
      lookup uses `?` placeholders.
- [ ] No password is logged or echoed back in any response
      body.
- [ ] All other existing routes (`/`, `/register`,
      `/expenses/...`) still respond as before.
