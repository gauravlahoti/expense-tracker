# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**Spendly** is a Flask + SQLite expense-tracker web app built as a step-by-step teaching project. Many routes and the entire database layer are intentional stubs that students fill in incrementally.

## Running the app

```bash
pip install -r requirements.txt
python app.py          # runs on http://localhost:5001 with debug=True
```

## Running tests

```bash
pytest                         # all tests
pytest tests/test_foo.py       # single file
pytest -k "test_name"          # single test by name
```

## Architecture

| Layer | Location | Notes |
|---|---|---|
| Flask app & routes | `app.py` | Single-file; all routes live here |
| Database helpers | `database/db.py` | Stub — students implement `get_db()`, `init_db()`, `seed_db()` |
| Jinja2 templates | `templates/` | All pages extend `base.html` |
| Static assets | `static/css/style.css`, `static/js/main.js` | Plain CSS/JS; no build step |

### Database conventions (`database/db.py`)
- `get_db()` — returns a SQLite connection with `row_factory = sqlite3.Row` and `PRAGMA foreign_keys = ON`
- `init_db()` — idempotent table creation via `CREATE TABLE IF NOT EXISTS`
- `seed_db()` — sample data for development only

### Stub routes in `app.py`
Routes for `/logout`, `/profile`, `/expenses/add`, `/expenses/<id>/edit`, and `/expenses/<id>/delete` return placeholder strings and are implemented in later steps.

### Template structure
`base.html` defines the full page shell (navbar, `<main>`, footer, Google Fonts). Child templates override `{% block title %}`, `{% block head %}`, `{% block content %}`, and `{% block scripts %}`.
