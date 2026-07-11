# Russian Learning — Multi-Student

## What this is

Russian language flashcard site for multiple students. Each student has their own Notion page linking to Miro boards; vocab is scraped, compiled into a markdown file, and served as a standalone HTML flashcard game. Published as a static site to GitHub Pages.

## Structure

- `students.json` — roster: `[{ id, name, notionUrl }, ...]`
- `<id>/russian-vocabulary.md` — that student's full vocab list, organized by lesson
- `<id>/flashcards.html` — that student's self-contained flashcard game
- `index.html` — landing page, lists students from `students.json`, links to `<id>/flashcards.html`

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
1. Resolve the student from `students.json` (or register a new one, asking for name + Notion URL)
2. Scrape that student's Notion page for Miro board links
3. Ask which board to pull (default: last one = newest lesson)
4. Extract text via Miro's accessibility "Explore board content" button
5. Append the new lesson to `<id>/russian-vocabulary.md`
6. Add the new words to `VOCAB`, `LESSON_NAMES`, and the lesson `<select>` in `<id>/flashcards.html`
7. Optionally commit + push to publish

Never hardcode a student's Notion URL in the skill — it always comes from `students.json`.

## Theming

`index.html` and every `<id>/flashcards.html` support 4 themes (Cyberpunk, Retro 80s, Classic, Terminal), all driven by CSS custom properties scoped under `:root[data-theme="..."]`. The choice is stored in `localStorage['rf-theme']` and applied via `document.documentElement.dataset.theme` — same key/values across all pages, so a choice made on one page carries over to the next (shared origin under `pourquoi.github.io/russian-flash/`). Default (no saved preference) is `cyberpunk`.

The new-student template (`.claude/skills/notion-vocab/template-flashcards.html`) has the identical theme block + selector — keep it in sync if the theme system changes.

## Students so far

| id | name |
|---|---|
| ad3b65 | Codename |

Codename's lessons: see `ad3b65/russian-vocabulary.md`.

## Tech notes

- Playwright 1.61.1 installed, Chromium headless shell at `~/Library/Caches/ms-playwright/`
- Playwright node module lives in the scratchpad dir — use `require('./node_modules/playwright')`, not `require('playwright')`
- Miro renders on WebGL canvas — DOM text extraction doesn't work. Only reliable path: wait ~15–40s for board to load, then click the leaf element with text `"Explore board content"` to activate the accessibility outline
- Notion REST API works fine for scraping links (`querySelectorAll('a[href]')` with `notion-link-token` class)
- Miro REST API v1 returns 401 for unauthenticated widget requests
- `.claude/settings.local.json` is auto-gitignored (no `.gitignore` needed) — safe to hold local permission tweaks without leaking into commits
