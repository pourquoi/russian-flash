# Russian Learning — Multi-Student

## What this is

Russian language flashcard site for multiple students. Each student has their own Notion page linking to Miro boards; vocab is scraped, compiled into a markdown file, and served as a standalone HTML flashcard game. Published as a static site to GitHub Pages.

## Structure

- `students.json` — public roster: `[{ id, notionUrl }, ...]` — `id` is an opaque codename, no real names
- `students.local.json` — gitignored, real-name lookup: `[{ id, realName }, ...]` — never committed
- `<id>/russian-vocabulary.md` — that student's full vocab list, organized by lesson
- `<id>/flashcards.html` — that student's self-contained flashcard game
- `index.html` — static landing page; does not list students (no roster) — each student gets their own `<id>/flashcards.html` link directly from the teacher

## GitHub Pages

Repo: `pourquoi/russian-flash` → published at `https://pourquoi.github.io/russian-flash/`. Pages serves from the `main` branch root. Pushing to `main` republishes automatically.

Repo + Pages were set up once via `gh`:
```
gh repo create pourquoi/russian-flash --public --source=. --remote=origin --push
gh api -X POST repos/pourquoi/russian-flash/pages -f "source[branch]=main" -f "source[path]=/"
```
No CI/build step — plain static files served as-is, so `git push` is the entire deploy.

## Adding a student / pulling new vocabulary

Use `/notion-vocab` skill. It will:
1. Resolve the student from `students.local.json` by real name (or register a new one: ask for name + Notion URL, generate an opaque `id`)
2. Scrape that student's Notion page for Miro board links
3. Ask which board to pull (default: last one = newest lesson)
4. Extract text via Miro's accessibility "Explore board content" button
5. Append the new lesson to `<id>/russian-vocabulary.md`
6. Add the new words to `VOCAB`, `LESSON_NAMES`, and the lesson `<select>` in `<id>/flashcards.html`
7. Optionally commit + push to publish

Never hardcode a student's Notion URL in the skill — it always comes from `students.json`.

## Anonymity

Students are identified only by opaque codenames (random, e.g. `openssl rand -hex 3`) — never a slug of their real name. `students.json` (public/committed) holds `{id, notionUrl}` only. `students.local.json` (gitignored) holds the real-name mapping and never leaves the machine. `index.html` has no roster; students receive their personal `<id>/flashcards.html` link directly from the teacher, not by browsing the site. Watch out: a student's Notion page URL slug is derived from their Notion page title, so it can leak their real name — rename the Notion page itself to strip it.

## Theming

`index.html` and every `<id>/flashcards.html` support 4 themes (Cyberpunk, Retro 80s, Classic, Terminal), all driven by CSS custom properties scoped under `:root[data-theme="..."]`. A single icon button (`#themeToggle`, in the `.topbar` next to the title) cycles through them on click — no dropdown. The choice is stored in `localStorage['rf-theme']` and applied via `document.documentElement.dataset.theme` — same key/values across all pages, so a choice made on one page carries over to the next (shared origin under `pourquoi.github.io/russian-flash/`). Default (no saved preference) is `retro`.

The new-student template (`.claude/skills/notion-vocab/template-flashcards.html`) has the identical theme block + selector — keep it in sync if the theme system changes.

## Tech notes

- Playwright 1.61.1 installed, Chromium headless shell at `~/Library/Caches/ms-playwright/`
- Playwright node module lives in the scratchpad dir — use `require('./node_modules/playwright')`, not `require('playwright')`
- Miro renders on WebGL canvas — DOM text extraction doesn't work. Only reliable path: wait ~15–40s for board to load, then click the leaf element with text `"Explore board content"` to activate the accessibility outline
- Notion REST API works fine for scraping links (`querySelectorAll('a[href]')` with `notion-link-token` class)
- Miro REST API v1 returns 401 for unauthenticated widget requests
- `.claude/settings.local.json` is auto-gitignored (no `.gitignore` needed) — safe to hold local permission tweaks without leaking into commits
