# Labs site (MkDocs Material)

Student-facing lab guides for the EPITA Computer Security course, published to GitHub Pages — same
pipeline as the `epita-n8n` docs site.

## Edit / preview locally

```bash
cd epita-softwaresecurity/docs-site
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/mkdocs serve      # live preview at http://127.0.0.1:8000
```

Edit the `.md` files under `docs/`. The source of truth for lab content stays here now (the old flat
files in `labs/class5/*.md` have been folded into these pages).

```bash
.venv/bin/mkdocs build --strict   # CI runs this; --strict fails on any broken link
```

## Structure

```
docs/
  index.md                     landing page + prerequisites
  class5/
    https-lab.md               Lab 1 — capture over HTTP, turn on HTTPS, compare certs
    password-crack.md          Lab 2A — crack an unsalted-MD5 dump
    password-fix.md            Lab 2B — migrate NotesApp login to Argon2
```

Add a future class by creating `docs/classN/` and a nav section in `mkdocs.yml`.

## Publish to GitHub Pages

Two options:

1. **Automatic (recommended)** — move `deploy-docs.yml` to `.github/workflows/` at your repo root,
   then in the repo: **Settings → Pages → Source = GitHub Actions**. Every push that touches
   `docs-site/**` rebuilds and deploys. The header of that file notes the one path tweak if
   `docs-site` is the repo root rather than a subfolder.
2. **Manual, one command** — from `docs-site/`, run `.venv/bin/mkdocs gh-deploy`. This builds and
   pushes to the `gh-pages` branch. Set **Settings → Pages → Source = Deploy from branch →
   `gh-pages`**.

## Conventions (match the n8n site)

- Checkpoints and expected output → `!!! success` admonitions.
- Discussion questions → `!!! question` with the answer in a collapsible `??? success "Answer"`.
- Troubleshooting and optional bonuses → collapsible `??? bug` / `??? tip`.
- "On purpose" traps (the planted SQL injection, the self-signed cert warning) → `!!! danger`.
- Every code block gets a language tag so the copy button and highlighting work.
