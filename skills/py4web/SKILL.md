---
name: py4web
description: Canonical py4web development. Use for ANY work on a py4web app (routes, fixtures, models, forms, templates, auth, sessions, Redis, tests). Enforces pyDAL as the only ORM (native SQL through pyDAL), the native yatl template system, and Bootstrap 5.3.x for the UI.
---

# py4web canonical development

Follow the official py4web documentation (https://py4web.com/_documentation/). If you are not sure an API exists or how it behaves, say so and check the docs or the installed source (`py4web`, `pydal`, `yatl`) instead of guessing.

## Hard rules (never break these)

1. **ORM**: use ONLY pyDAL. No SQLAlchemy, Peewee, Django ORM or any other ORM.
2. **Native SQL**: only through pyDAL (`db.executesql(...)` with placeholders). Never build SQL with f-strings, `%` or `+` on user input.
3. **Templates**: use ONLY the native py4web template system (yatl, `[[ ]]` delimiters). No Jinja2, Mako or other engines.
4. **UI**: Bootstrap 5.3.x. Do not use Bulma, Tailwind, Bootstrap 4 or Bootstrap 6/alpha versions unless the user asks.
5. **Import shared objects** (`db`, `session`, `auth`, `flash`, `T`, `cache`) from `.common`. Never recreate them in other modules.
6. **Import from py4web, not pydal**, inside apps: `from py4web import action, request, response, redirect, URL, Field, DAL`. In py4web, `py4web.DAL` is a fixture; `pydal.DAL` is not.
7. Declare everything an action needs with `@action.uses(...)`. Do not use globals or hidden state for request data.
8. Do not edit py4web itself, the Dashboard (`_dashboard`) or other apps in `apps/` unless asked.
9. Do not add dependencies without asking.

## App layout (scaffold convention)

```
apps/<app_name>/
  __init__.py        # imports controllers (and models) so routes get registered
  settings.py        # config (DB_URI, DB_FOLDER, SESSION_TYPE, secrets from env)
  common.py          # db, session, auth, flash, T, cache, shared fixtures
  models.py          # pyDAL table definitions
  controllers.py     # actions (routes)
  templates/         # yatl templates (layout.html, index.html, ...)
  static/            # css/js/images
  tests/             # pytest tests
  databases/         # sqlite files, pyDAL migration files (do not delete)
```

Start new apps from the py4web scaffold when possible and adapt it, rather than writing the structure by hand.

## common.py (canonical shape)

```python
from py4web import DAL, Session, Translator, Flash
from py4web.utils.auth import Auth
from . import settings

db = DAL(
    settings.DB_URI,
    folder=settings.DB_FOLDER,
    pool_size=settings.DB_POOL_SIZE,
    migrate=settings.DB_MIGRATE,
    fake_migrate=settings.DB_FAKE_MIGRATE,
)
session = Session(secret=settings.SESSION_SECRET_KEY)
flash = Flash()
T = Translator(settings.T_FOLDER)
auth = Auth(session, db)
auth.enable()   # registers the auth routes; do this once, after defining auth
```

- Keep secrets and URIs in `settings.py` / environment variables, never hard-coded in code.
- Redis sessions: only if the project already uses them. Follow the scaffold's `SESSION_TYPE = "redis"` pattern (a `redis.Redis(...)` connection passed as `storage=` to `Session`).

## models.py (pyDAL)

```python
from py4web import Field
from pydal.validators import IS_NOT_EMPTY, IS_IN_SET
from .common import db, auth

db.define_table(
    "thing",
    Field("name", requires=IS_NOT_EMPTY()),
    Field("status", default="new", requires=IS_IN_SET(["new", "done"])),
    Field("owner_id", "reference auth_user", default=lambda: auth.user_id, writable=False, readable=False),
    auth.signature,       # created_on/by, modified_on/by
    format="%(name)s",
)
db.commit()
```

Rules:
- Use field types, validators (`requires=`), `default`, `unique`, `notnull`, `format`, `reference <table>` properly.
- Tables are created/migrated by pyDAL automatically. Before changing an existing table definition, explain the migration impact and ask for confirmation. Never delete migration files in `databases/` while the tables exist.
- Computed/virtual fields, `_common_filter` (soft delete, multi-tenant) and callbacks (`_before_insert`, ...) are the canonical pyDAL way to put behavior in the model.

### Queries

```python
rows = db(db.thing.status == "new").select(orderby=~db.thing.created_on, limitby=(0, 20))
row = db.thing[thing_id]                      # by id, None if missing
db(db.thing.id == thing_id).update(status="done")
db.thing.validate_and_insert(name="x")        # returns errors dict if validation fails
```

- Prefer pyDAL expressions, joins (`db(a.id == b.a_id).select(a.ALL, b.name)`), aggregates (`.count()`, `.sum()`) and `left=` joins.
- Always paginate with `limitby`.
- Use `.first()`, `.as_dict()`, `.as_list()` when needed for JSON.

### Native SQL (only when pyDAL expressions are not enough)

```python
rows = db.executesql(
    "SELECT status, COUNT(*) AS n FROM thing WHERE owner_id = %s GROUP BY status",
    placeholders=[auth.user_id],
    as_dict=True,
)
```

- ALWAYS use `placeholders=`. Check the placeholder style of the database in use (`%s` or `?`) in the pyDAL docs.
- Use table/field names from `db.thing` (`db.thing._tablename`, `db.thing.name.longname`) instead of hard-coded names when possible.
- `db(query)._select()` / `_update()` / `_insert()` show generated SQL for debugging without running it.

## controllers.py (actions and fixtures)

```python
from py4web import action, request, redirect, URL, abort
from py4web.utils.form import Form, FormStyleBootstrap5
from .common import db, session, auth, flash

@action("index")
@action.uses("index.html", db, session, auth, flash)
def index():
    things = db(db.thing).select(limitby=(0, 50))
    return dict(things=things)

@action("thing/new", method=["GET", "POST"])
@action.uses("form.html", db, session, auth.user, flash)
def thing_new():
    form = Form(db.thing, formstyle=FormStyleBootstrap5)
    if form.accepted:
        flash.set("Saved", _class="success")
        redirect(URL("index"))
    return dict(form=form)

@action("api/things", method="POST")
@action.uses(db, auth.user)
def api_things_post():
    return db.thing.validate_and_insert(**request.json)
```

Rules:
- **Put the template FIRST in `@action.uses(...)`**, then the other fixtures (`"index.html", db, session, auth`).
- Actions return a `dict` (rendered by the template) or a dict/list for JSON. Do not return HTML strings.
- Use `auth.user` (instead of `auth`) to require login; use `Condition(...)` and `auth.has_membership` / permissions for authorization.
- Use `URL("action_name", vars=...)` to build URLs and `redirect(URL(...))` to redirect. Never hard-code paths.
- Path variables: `@action("thing/<thing_id:int>")`. Use `abort(404)` when a record is missing.
- Keep controllers thin: validation in the model (`requires=`), reusable queries in functions in `models.py` or a helper module.
- Always filter data by the current user when records are owned (`db.thing.owner_id == auth.user_id`).
- Prefer `Form` / `Grid` (`py4web.utils.form`, `py4web.utils.grid`) for CRUD. Use the Bootstrap 5 styles they provide (for example `FormStyleBootstrap5`); check the installed py4web version for the exact class names before using them.

## Templates (yatl, native)

Delimiters are `[[ ]]`. Output is HTML-escaped by default; use `XML(...)` only for trusted HTML.

```html
[[extend 'layout.html']]

<h1 class="h3 mb-3">Things</h1>
<ul class="list-group">
  [[for t in things:]]
  <li class="list-group-item d-flex justify-content-between">
    <span>[[=t.name]]</span>
    <a class="btn btn-sm btn-outline-primary" href="[[=URL('thing', t.id)]]">Open</a>
  </li>
  [[pass]]
</ul>
```

Rules:
- Blocks are closed with `[[pass]]` (or `[[end]]` where documented). There is no indentation rule.
- Use `[[extend 'layout.html']]` in pages, `[[include]]` inside `layout.html` for page content, `[[include 'partial.html']]` for partials, and `[[block name]] ... [[end]]` / `[[super]]` for overridable regions.
- Templates only render. No database access and no business logic in templates.
- Use `[[=URL('static', 'js/app.js')]]` for static files.
- Use `Inject(my_var=my_var)` (fixture) to expose a variable to templates instead of globals.
- Use yatl helpers (`from yatl.helpers import DIV, A, INPUT, ...`) in Python only when generating small fragments; do not build whole pages in Python.

## Bootstrap 5.3.x

Use the current Bootstrap 5.3 line (5.3.3 was verified as a valid release; check https://getbootstrap.com for the latest 5.3.x patch and pin that exact version). Do not float on `@latest`.

`templates/layout.html`:

```html
<!doctype html>
<html lang="en" data-bs-theme="light">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>App</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="[[=URL('static', 'css/app.css')]]" rel="stylesheet">
  [[block page_head]][[end]]
</head>
<body>
  <nav class="navbar navbar-expand-lg bg-body-tertiary mb-4">
    <div class="container">
      <a class="navbar-brand" href="[[=URL('index')]]">App</a>
      <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#nav">
        <span class="navbar-toggler-icon"></span>
      </button>
      <div class="collapse navbar-collapse" id="nav">
        <ul class="navbar-nav me-auto">
          <li class="nav-item"><a class="nav-link" href="[[=URL('index')]]">Home</a></li>
        </ul>
      </div>
    </div>
  </nav>
  <main class="container">
    [[include]]
  </main>
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
  [[block page_scripts]][[end]]
</body>
</html>
```

- For production or offline use, copy the pinned Bootstrap files into `static/` and reference them with `URL('static', ...)`. When using the CDN in production, add the SRI `integrity` hash from getbootstrap.com for the exact version.
- Use Bootstrap utilities and components (grid `row`/`col-*`, `card`, `table`, `form-control`, `alert`, `btn`, `modal`) instead of custom CSS. Put the few custom rules in `static/css/app.css`.
- Keep pages responsive and accessible (labels for inputs, `aria-*` attributes, semantic HTML, sufficient contrast).
- The scaffold layout is not Bootstrap-based. Replace the scaffold's styling in `layout.html`, but keep py4web's flash mechanism working (the `flash` fixture plus the scaffold's flash element and `utils.js`) and style it with Bootstrap alert classes.

## Security checklist

- Parameterized SQL only (pyDAL expressions, or `executesql` with `placeholders`).
- Do not disable template escaping on user data.
- Forms made with `Form` include py4web's form key protection; keep using them for POST actions that change data. Use `URLSigner` for signed URLs when exposing sensitive GET actions.
- Check ownership and permissions in every action that reads or changes owned data.
- Secrets in environment variables or `settings.py` loaded from the environment; never commit them or print them.
- Set a strong `SESSION_SECRET_KEY` outside development.

## Redis (if the project uses it)

- Use prefixed keys, e.g. `<app>:<feature>:<id>`, and ALWAYS set a TTL on cache keys.
- Use `SCAN`, never `KEYS *` or `FLUSHDB`/`FLUSHALL` on shared instances.
- Treat Redis as a cache or session store, never as the source of truth. The source of truth is the pyDAL database.

## Testing and quality

- Write pytest tests for each behavior change. Use a temporary SQLite database (settings override) for tests, never the real database.
- Run `ruff` (or the project's linter) and `pytest` before declaring a task done, and report the results.
- Keep functions small, add docstrings to public helpers, and use type hints where they help.

## Working method

1. **Research**: read `settings.py`, `common.py`, `models.py`, `controllers.py` and the relevant templates before changing anything.
2. **Plan**: list files to change and why. Ask before schema changes, auth changes or new dependencies.
3. **Implement**: smallest change that works; reuse existing fixtures, tables and templates.
4. **Verify**: run tests and lint, then summarize what changed.

## Anti-patterns to reject

- Importing `pydal.DAL` directly in an app, or creating a second `db`/`session`/`auth` outside `common.py`.
- Template not placed first in `@action.uses`.
- SQL built with string formatting.
- HTML built as strings in controllers instead of templates.
- Business logic or queries inside templates.
- Flask/Django habits: `@app.route`, `render_template`, Jinja2 syntax (`{{ }}`, `{% %}`), Flask's `request.args` / `request.form`. In py4web use `request.query`, `request.json` and `Form` (check the docs for form data).
- Hard-coded URLs instead of `URL(...)`.
- Deleting migration files or dropping tables without explicit confirmation.
