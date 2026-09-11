# How this project is published

Companion to `../HANDOVER.md`. Everything here went live on **2026-09-12**. Before that date
nothing had ever been pushed, and older notes that say so are stale.

---

## 1. Where everything is

| what | where |
|---|---|
| repository | https://github.com/shaynebennetts/load-the-train |
| project website | https://shaynebennetts.github.io/load-the-train/ |
| the game | https://shaynebennetts.github.io/load-the-train/app/ |
| self tests | https://shaynebennetts.github.io/load-the-train/app/index.html?test |
| class wall card | `TIGP-Experimental-Methods/showcase-2026` → `projects/shaynebennetts--load-the-train.json` + `images/shaynebennetts--load-the-train.gif` |

GitHub Pages is enabled on `main`, path `/`.

## 2. Repository layout — and why the root is no longer off limits

The course requires, for **every** project: one public repository, one GitHub-Pages project
website, one card on the class wall. Its layout is the app in `app/`, the project website as
`index.html` at the repository root.

```
index.html      the project website  (root — this is what the root was reserved FOR)
favicon.svg     icon, drawn in the game's palette
images/         screenshots + icon PNG, for the website only
app/index.html  the game, one self-contained file
SPEC.md PLAN.md README.md HANDOVER.md docs/
```

Older handover notes carried a rule "**never write to the repository root** — `git ls-files`
must never show a root `index.html`". That rule existed because the root was reserved for a
website that had not been written yet. **It is satisfied, not broken, by the current
layout.** Do not restore the prohibition. Page assets belong in `images/`; nothing else new
belongs at the root.

## 3. Updating the website

`index.html` is hand-written, no build step. Every number on it is measured — the per-test
worst relative errors, the ~91 s floor, 6.00 m/s terminal speed, 4.78 m/s one-pass limit,
0.31 s gap window. They come from `MEASUREMENTS.md`. **If a parameter changes, the website
goes stale silently**; there is no test covering it.

Screenshots in `images/` were captured headlessly — method in `TESTING.md` §4.

## 4. The wall card

Full operational detail is in `showcase-2026/MAINTENANCE.md`, cloned at
`C:/Claude/TIGP-2026-2/showcase-2026`. The three things that matter here:

- **Its `updated` field is 2026-09-10, the date the project began — not the date the card was
  last edited.** This is deliberate and is what keeps the instructor's cards below the
  students' work on the projector. A `rank` field was tried on 2026-09-12 and removed the
  same day at Shayne's request; the dates carry the ordering instead. **Do not "correct" this
  date.**
- **The blurb is Claude's draft.** Course rule is that it should be the student's own two
  sentences. Still waiting to be replaced.
- Validate with `python scripts/build.py` before pushing; never hand-edit `projects.json`.

## 5. Before announcing anything is live

GitHub Pages 404s for a few minutes after a repository is first pushed, and the
`pages build and deployment` Action finishes well after the repository's other workflows.
Check the served file, not the run status:

```bash
for u in / app/ images/loading.png favicon.svg; do
  echo "$(curl -s -o /dev/null -w '%{http_code}' "https://shaynebennetts.github.io/load-the-train/$u")  $u"
done
```

I reported "it's live" once on the strength of a completed Action while the deploy was still
in progress, and it was not. Wait for the 200s.
