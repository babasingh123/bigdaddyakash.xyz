# learning-site

Source for [bigdaddyakash.xyz](https://bigdaddyakash.xyz) — practical, concept-first notes on **HLD**, **LLD**, **DSA disguised as design**, and **NeetCode 150**.

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/), hosted on GitHub Pages.

## What's inside

- **HLD** — system design fundamentals (databases, caching, queues, distributed systems, etc.) + 15 architect-level problems
- **LLD** — OOP, SOLID, design patterns + 20 LLD problems
- **DSA Disguised as Design** — 50 "design X" problems that are really DSA in costume
- **NeetCode 150** — 18 topics with prerequisites, patterns, and problem lists

Every problem cross-links to the underlying concept; every concept lists the problems that use it.

## Local development

Install MkDocs Material once:

```bash
python3 -m pip install --user mkdocs-material
```

Then, from this directory:

```bash
mkdocs serve   # live preview at http://127.0.0.1:8000
```

Edit any markdown file under `docs/` and the preview hot-reloads.

## Build

```bash
mkdocs build --strict
```

`--strict` turns warnings (e.g. broken nav links) into errors. CI runs the same command.

## Deploy

Push to `main`. The GitHub Actions workflow in `.github/workflows/deploy.yml` builds the site and publishes to GitHub Pages, which serves `bigdaddyakash.xyz` (via the `CNAME` in `docs/`).

Typical loop:

```bash
git pull
mkdocs serve              # edit + preview
git add docs/
git commit -m "..."
git push                  # auto-deploys in ~1 minute
```

## Adding a new page

1. Create `docs/<section>/<slug>.md`
2. Add it to the `nav:` block in `mkdocs.yml`
3. `mkdocs serve` to verify
4. Commit + push

## Repository layout

```
learning-site/
├── mkdocs.yml                    # site config + nav tree
├── docs/
│   ├── index.md                  # landing page
│   ├── CNAME                     # custom domain
│   ├── hld/                      # 18 fundamentals + 15 problems
│   ├── lld/                      # 4 fundamentals + 20 problems
│   ├── dsa-design/               # 10 categories + 50 problems
│   └── neetcode-150/             # 18 topics
└── .github/workflows/deploy.yml  # GitHub Pages deploy
```

## Philosophy

- **Concept-first.** Every problem links back to the fundamentals it exercises.
- **Practical, not memorization.** Explain the *why*, not just the *what*.
- **No backend.** Pure markdown + static site generation.
- **YAGNI.** No progress tracking, no comments, no quizzes — just reading.
