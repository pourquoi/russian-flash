---
name: notion-vocab
description: Pull vocabulary from a Notion page that links to Miro boards, for any student registered in students.json. Use when the user says "pull vocab", "get vocabulary from notion", "new lesson vocab", "add a new student", or similar.
---

Extract vocabulary from a Notion page whose lessons link to Miro boards, for one of possibly several students.

Each student has their own Notion page, own vocab file, and own flashcards site under `<id>/`. The roster lives in `students.json` at the repo root.

## When to Use

- User says "pull vocab", "get vocab from notion", "new lesson", "add vocab"
- User says "add a new student" / "set up a new student"
- User provides a Notion URL and asks for vocabulary extraction

## Workflow

### Step 0: Identify the student

Read `students.json`:

```json
[
  { "id": "ad3b65", "name": "Codename", "notionUrl": "https://app.notion.com/p/..." }
]
```

- If the user named a student, match by `name` or `id`.
- If ambiguous or unspecified, list the students and ask which one.
- If the student isn't in the roster ("new student"):
  1. Ask for their name and Notion page URL.
  2. Derive `id` as a lowercase-kebab slug of the name (e.g. "Jane Doe" → `jane-doe`).
  3. Append `{ id, name, notionUrl }` to `students.json`.
  4. Create `<id>/`:
     - `russian-vocabulary.md` with a single `# <Name> — Russian Vocabulary` heading.
     - `flashcards.html` copied from `.claude/skills/notion-vocab/template-flashcards.html` (blank VOCAB/LESSON_NAMES/lesson dropdown).
  5. Add a link for the student in root `index.html` if it isn't already driven dynamically from `students.json` (currently `index.html` fetches `students.json` at runtime, so no edit needed there).

All following steps operate on the resolved student's `notionUrl` and `<id>/` files.

### Step 1: Scrape Notion page and show available Miro links

```js
const { chromium } = require('./node_modules/playwright');

const NOTION_URL = process.argv[2]; // pass the student's notionUrl explicitly

(async () => {
  const browser = await chromium.launch({ headless: true });
  const page = await browser.newPage();
  await page.goto(NOTION_URL, { waitUntil: 'domcontentloaded', timeout: 30000 });
  await page.waitForTimeout(6000);

  const links = await page.evaluate(() =>
    Array.from(document.querySelectorAll('a[href]'))
      .map(a => ({ text: a.innerText.trim(), href: a.href }))
      .filter(l => l.href.includes('miro.com') && l.text.length > 0)
  );
  console.log(JSON.stringify(links, null, 2));
  await browser.close();
})();
```

Run as: `node list_boards.js <student-notion-url>`

Then **ask the user which board to pull** (default: last one = newest lesson).

### Step 2 (after user confirms): Scrape the selected Miro board

Miro renders on WebGL canvas — no DOM text. Only path: the **"Explore board content"** accessibility button, which outputs a readable text outline.

Wait until `"Explore board content"` appears in `document.body.innerText` (up to ~40s), then click the leaf element with that exact text.

```js
async function scrapeMiroBoard(browser, { name, url }) {
  const page = await browser.newPage({ viewport: { width: 1920, height: 1080 } });
  try {
    await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 30000 });

    let loaded = false;
    for (let i = 0; i < 8; i++) {
      await page.waitForTimeout(5000);
      const text = await page.evaluate(() => document.body.innerText);
      if (text.includes('Explore board content')) { loaded = true; break; }
    }
    if (!loaded) return { name, content: 'Board did not load' };

    await page.evaluate(() => {
      for (const el of document.querySelectorAll('*')) {
        if (el.children.length === 0 && el.textContent.trim() === 'Explore board content') {
          el.click();
          return;
        }
      }
    });

    await page.waitForTimeout(4000);
    const content = await page.evaluate(() => document.body.innerText);
    return { name, content };
  } finally {
    await page.close();
  }
}
```

### Step 3: Compile vocabulary

From the raw text:
- Flip card pattern: `Word\nFlip` — word is on the line before "Flip"
- Table rows, numbered lists with word pairs
- Strip Miro UI noise: "Skip to...", "Accessibility controls", "Sign up for free", percentages, image filenames (UUID.png)

### Step 4: Output — update the student's files

Append to `<id>/russian-vocabulary.md`:

```markdown
## Lesson name

| Russian | English |
|---|---|
| Привет! | Hi! |
```

Update `<id>/flashcards.html` in **three places** (all must stay in sync):
1. `VOCAB` array — append `{ru:"...",en:"...",l:N}` entries for the new lesson.
2. `LESSON_NAMES` map — add `N:"Урок N — Topic"`.
3. `<select id="lessonFilter">` — add `<option value="N">Урок N — Topic</option>`.

### Step 5: Publish (optional)

If the user wants the update live, commit and push — GitHub Pages rebuilds automatically from the pushed branch:

```
git add students.json <id>/ index.html
git commit -m "Add <student>'s lesson N vocab"
git push
```

Confirm with the user before pushing.

## Notes

- playwright node module lives in the scratchpad dir — use `require('./node_modules/playwright')`, not `require('playwright')`
- Chromium headless shell at `~/Library/Caches/ms-playwright/`
- Miro boards need ~15–40s to render before the accessibility button appears
- Notion uses `<a class="notion-link-token ...">` for Miro links — `querySelectorAll('a[href]')` works
- Miro REST API v1 returns 401 for unauthenticated requests; accessibility trick is the only reliable path
- Never hardcode a Notion URL in this skill — always resolve it from `students.json` per student
