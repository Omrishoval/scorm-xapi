# scorm-xapi — Claude Code Skill

Add LMS reporting to a Vite/React learning module.

Covers SCORM 1.2, SCORM 2004 4th Edition, xAPI/Tin Can, and cmi5 — with production-tested patterns from real deployments on SCORM Cloud and SuccessFactors.

---

## What it does

When you invoke `/scorm-xapi` (or Claude detects you need LMS reporting), the skill:

1. Asks which LMS you're targeting and picks the right standard
2. Provides the inline `<script>` for `index.html` (SCORM init must NOT be in React)
3. Provides a TypeScript bridge (`scorm.ts` or `xapi.ts`) for React to call
4. Provides the manifest file (`imsmanifest.xml` or `tincan.xml`)
5. Provides a packaging script (`pack-scorm.mjs`)
6. Walks you through testing on SCORM Cloud before delivery

---

## Installation

**Claude Code:**
```bash
npx skills add https://github.com/Omrishoval/scorm-xapi --skill scorm-xapi -g -a claude-code
```

**Codex:**
```bash
npx skills add https://github.com/Omrishoval/scorm-xapi --skill scorm-xapi -g -a codex
```

**All agents at once:**
```bash
npx skills add https://github.com/Omrishoval/scorm-xapi --skill scorm-xapi -g -a '*'
```

Or manually — copy the `scorm-xapi/` folder into your agent's skills directory (`~/.claude/skills/`, `~/.codex/skills/`, etc.).

---

## Usage

Trigger it by asking Claude:

- "Add xAPI reporting to my course"
- "Wrap this in SCORM and package it"
- "How do I report a quiz score to the LMS?"
- "תוסיף דיווח SCORM ל-LMS"
- "תארז את הלומדה"

Claude will ask 4 mandatory questions before writing any code:
1. Which LMS?
2. Does the learner need to resume mid-course?
3. Does the course have a quiz?
4. Single-file or multi-asset delivery?

---

## Reference files

| File | Contents |
|------|----------|
| `references/decision-tree.md` | Choosing SCORM 1.2 vs 2004 vs xAPI |
| `references/scorm-12.md` | Full SCORM 1.2 implementation |
| `references/scorm-2004.md` | Full SCORM 2004 4th Ed implementation |
| `references/xapi.md` | Full xAPI service class + React integration |
| `references/cmi5.md` | cmi5 launch protocol and MoveOn policies |
| `references/interactions.md` | Per-question interaction reporting |
| `references/suspend-data.md` | Mid-course resume / bookmark support |
| `references/lms-compatibility.md` | Per-LMS quirks (SuccessFactors, Moodle, etc.) |
| `references/packaging-vite.md` | Vite config, ZIP script, standalone packaging |
| `references/testing-checklist.md` | Pre-delivery checklist, SCORM Cloud smoke test |

---

## Key pitfalls (production-validated)

**SCORM init must be in `index.html`** — not in React or any ES module. Vite builds `<script type="module">` which runs deferred and in its own scope. The LMS injects `window.API` synchronously at page load — if you miss it, you miss it.

**SCORM Cloud sends actors in legacy Tin Can format** — `name` and `account` arrive as arrays, and `account` uses `accountServiceHomePage`/`accountName` instead of the xAPI 1.0 `homePage`/`name` fields. The `xapi.ts` service normalizes this automatically.

**`window.close()` must be synchronous** — Chrome blocks it if called inside `setTimeout`. Call it directly in the click handler.

**`terminated` fires twice** — if `beforeunload` fires after your close button calls `xapi.finish()`, you get a duplicate. The `finish()` method has an idempotency guard.

---

## Standards comparison

| | SCORM 1.2 | SCORM 2004 4th Ed | xAPI |
|--|-----------|-------------------|------|
| LMS API | `window.API` | `window.API_1484_11` | HTTP POST to LRS |
| Completion | `cmi.core.lesson_status` | `cmi.completion_status` + `cmi.success_status` | `completed` + `passed`/`failed` verbs |
| Score | `cmi.core.score.raw` (0–100) | `cmi.score.scaled` (–1 to 1) | `result.score.scaled` |
| Session time | `HH:MM:SS` | `PTxHxMxS` | ISO 8601 duration |
| Mid-course exit | `LMSFinish("")` | `cmi.exit = 'suspend'` | N/A |
| Resume data | ~4 KB | 64 KB | LRS-dependent |
| Best for | Old/unknown LMS | Modern LMS | Mobile, multi-platform, non-LMS |

---

## Tested on

- SCORM Cloud (xAPI reference LRS + SCORM 1.2 / 2004 host)
- SuccessFactors (SCORM 1.2)
- Moodle 3+ (SCORM 2004)
