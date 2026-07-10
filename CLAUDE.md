# Russian Learning — Multi-Student

## What this is

Russian language flashcard site for multiple students. Each student has their own Notion page linking to Miro boards; vocab is scraped, compiled into a markdown file, and served as a standalone HTML flashcard game. Published as a static site to GitHub Pages.

## Structure

- `students.json` — roster: `[{ id, name, notionUrl }, ...]`
- `students/<id>/russian-vocabulary.md` — that student's full vocab list, organized by lesson
- `students/<id>/flashcards.html` — that student's self-contained flashcard game
- `index.html` — landing page, lists students from `students.json`, links to `students/<id>/flashcards.html`

## GitHub Pages

Repo: `pourquoi/russian-flash` → published at `https://pourquoi.github.io/russian-flash/`. Pages serves from the `main` branch root. Pushing to `main` republishes automatically.

## Adding a student / pulling new vocabulary

Use `/notion-vocab` skill. It will:
1. Resolve the student from `students.json` (or register a new one, asking for name + Notion URL)
2. Scrape that student's Notion page for Miro board links
3. Ask which board to pull (default: last one = newest lesson)
4. Extract text via Miro's accessibility "Explore board content" button
5. Append the new lesson to `students/<id>/russian-vocabulary.md`
6. Add the new words to `VOCAB`, `LESSON_NAMES`, and the lesson `<select>` in `students/<id>/flashcards.html`
7. Optionally commit + push to publish

Never hardcode a student's Notion URL in the skill — it always comes from `students.json`.

## Students so far

| id | name |
|---|---|
| ad3b65 | Codename |

Codename's lessons: see `students/ad3b65/russian-vocabulary.md`.

## Tech notes

- Playwright 1.61.1 installed, Chromium headless shell at `~/Library/Caches/ms-playwright/`
- Playwright node module lives in the scratchpad dir — use `require('./node_modules/playwright')`, not `require('playwright')`
- Miro renders on WebGL canvas — DOM text extraction doesn't work. Only reliable path: wait ~15–40s for board to load, then click the leaf element with text `"Explore board content"` to activate the accessibility outline
- Notion REST API works fine for scraping links (`querySelectorAll('a[href]')` with `notion-link-token` class)
- Miro REST API v1 returns 401 for unauthenticated widget requests
