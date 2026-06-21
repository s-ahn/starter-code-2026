# Deploying Café Cozy — a teaching guide

This guide walks through putting our Flask app on the public internet so anyone
can visit it with a URL. It's written to be followed step by step *and* to
explain **why** each step exists, so you understand deployment — not just copy
commands.

---

## 1. Why not GitHub Pages?

GitHub Pages can only serve **static files** (HTML, CSS, JS, images). It hands
the browser files exactly as they are on disk.

Our app is **not** static. It is a **Flask (Python) application**:

- The pages in `templates/` are **Jinja2 templates**, not finished HTML. They
  contain code like `{% for item in menu %}` and `{{ url_for('shop') }}` that a
  browser doesn't understand — it has to be *run* on a server first.
- The menu comes from a **SQLite database** (`instance/coffee.db`), read by
  Python when a page is requested.

So we need a host that can **run Python**, not just serve files. That's what
this guide sets up, using **Render** (free tier).

> 🧠 **Concept:** "static site" vs. "web application". A static site is files.
> A web application is a *program* that runs and generates responses on the fly.

---

## 2. What a host needs from us

To run our app, a hosting platform needs answers to three questions:

| Question | Our answer | Where it lives |
|----------|-----------|----------------|
| What do we need installed? | The packages in `requirements.txt` | `requirements.txt` |
| How do we get ready? (build) | Install packages, create the database | `render.yaml` → `buildCommand` |
| How do we start the app? (run) | Start a web server pointing at our app | `render.yaml` → `startCommand` |

We've put these answers in two files so the whole thing is **reproducible** —
anyone with the repo can deploy it the same way.

### `requirements.txt`
```
Flask>=2.0
gunicorn>=21.0
```
`gunicorn` is new. Flask's built-in server (`python app.py`) is for development
only — it even prints a warning telling you not to use it in production.
**Gunicorn** is a proper production web server that runs our Flask app.

> 🧠 **Concept:** dev server vs. production server. `app.run(debug=True)` is
> convenient for coding locally; gunicorn is built to handle real traffic.

### `render.yaml`
```yaml
services:
  - type: web
    name: cafe-cozy
    runtime: python
    plan: free
    buildCommand: "pip install -r requirements.txt && python init_db.py"
    startCommand: "gunicorn app:app --bind 0.0.0.0:$PORT"
    envVars:
      - key: PYTHON_VERSION
        value: "3.12.3"
```

Reading the two important lines:

- **buildCommand** — runs once when deploying. It installs our packages, then
  runs `init_db.py` to create the database and insert the sample menu.
- **startCommand** — `gunicorn app:app` means "run gunicorn, and find the Flask
  app in the `app` variable inside `app.py`." `--bind 0.0.0.0:$PORT` tells it
  which network port to listen on. **`$PORT` is given to us by Render** — we
  don't choose it; we read it from the environment.

> 🧠 **Concept:** environment variables. The platform passes settings to our app
> through the environment (like `$PORT`). Our code adapts to wherever it runs.

---

## 3. Deploy it (step by step)

**Prerequisites:** the project is pushed to a GitHub repository.

1. Go to <https://render.com> and sign up (you can sign in with GitHub).
2. Click **New +** → **Blueprint**.
   - "Blueprint" means "read the `render.yaml` in my repo and set everything up
     for me." This is why we committed that file.
3. Connect your GitHub account and select the **Café Cozy** repository.
4. Render reads `render.yaml`, shows you the service it will create, and you
   click **Apply**.
5. Watch the **Logs** tab. You'll see, in order:
   - `pip install` downloading Flask and gunicorn,
   - `Initialized database at ...` (that's `init_db.py` running),
   - `Starting gunicorn` / `Listening at: http://0.0.0.0:...`.
6. When it says **Live**, click the URL at the top
   (e.g. `https://cafe-cozy.onrender.com`). Your site is on the internet. 🎉

### The magic part: auto-deploy
From now on, **every `git push` to your main branch redeploys automatically.**
Change a price in `init_db.py`, push, and watch Render rebuild. This is the core
loop of modern deployment — students *see* their commit become a live change.

> 🧠 **Concept:** continuous deployment. Your Git history *is* your deploy
> history. The source of truth is the repo, not files on a server.

---

## 4. Things to know (good discussion points)

- **First load is slow on free tier.** Free services "spin down" after ~15 min
  of no traffic and take ~30s to wake up. Normal — not a bug.
- **The database resets on each deploy.** The free tier has an *ephemeral*
  filesystem: the disk is wiped and rebuilt every deploy. That's exactly why we
  run `init_db.py` in the build step — so the menu always exists. If a real app
  needed data to *persist*, we'd use a managed database instead of a file.
  - 🧠 Great teaching moment about **stateless** hosting and *where state lives*.
- **`debug=True` is for local only.** Render runs the app through gunicorn, so
  the `if __name__ == '__main__': app.run(debug=True)` block in `app.py` is
  simply ignored in production. We didn't have to change it.

---

## 5. Optional "Level 2": Docker (for later)

Once students understand the deploy above, Docker is a natural next lesson:
*"package the app and its environment into one portable image."* Render, Railway,
and Fly.io can all deploy from a `Dockerfile` instead of `render.yaml`. It's the
same app — just a different way to describe "how to build and run it." Save it
until the fundamentals here feel comfortable, so Docker is a *refinement*, not a
prerequisite.

---

## Quick reference

| File | Role |
|------|------|
| `app.py` | The Flask application (unchanged) |
| `init_db.py` | Creates the SQLite DB + sample menu |
| `requirements.txt` | Python packages to install (Flask + gunicorn) |
| `render.yaml` | How Render builds and runs the app |

**Run locally** (development):
```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python init_db.py
python app.py          # http://127.0.0.1:5000
```

**Run like production** (what Render does):
```bash
gunicorn app:app --bind 0.0.0.0:8000   # http://127.0.0.1:8000
```
