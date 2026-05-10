---
name: scorm-xapi
description: Add LMS reporting to a Vite/React learning module — SCORM 1.2, SCORM 2004 4th Edition, xAPI/Tin Can, or cmi5. Use whenever the user asks to: add SCORM or xAPI support, create or package a SCORM course, report quiz scores or completion to an LMS, generate imsmanifest.xml or tincan.xml, save or resume learner progress (suspend_data, bookmark), or integrate with any LMS. Also trigger on Hebrew: "תוסיף SCORM", "תארז לומדה", "דיווח ציון ל-LMS", "תוסיף דיווח", "איך מדווחים ציון", "תוסיף xAPI", "שמור התקדמות לומד", "חידוש מהמקום שעצרתי". Trigger even when the user is building any eLearning module and mentions LMS integration, without saying "SCORM" explicitly.
---

# SCORM & xAPI Skill — Navigation Guide

All implementation details live in `references/`. This file tells you which to read and in what order.

---

## Mandatory questions before starting

Ask these before writing a single line of code:

1. **Which LMS?** (SuccessFactors, Moodle, SCORM Cloud, Docebo, unknown…)
2. **Does the learner need to resume mid-course?** (bookmark + suspend_data)
3. **Does the course have a quiz?** (score + pass/fail reporting)
4. **Single-file delivery or multi-asset?** (affects packaging strategy)

---

## Step 1 — Choose the standard

Read `references/decision-tree.md` for the full decision logic. Short version:

| LMS / Scenario | Standard |
|----------------|----------|
| SuccessFactors, SAP, Moodle < 2.7, old Cornerstone, unknown LMS | **SCORM 1.2** |
| SCORM Cloud, Moodle 3+, Docebo, TalentLMS, Absorb | **SCORM 2004 4th Ed** |
| Mobile / offline / non-LMS / multi-platform tracking | **xAPI** |
| xAPI + LMS enrollment + MoveOn sequencing rules | **cmi5** (`references/cmi5.md`) |

Unsure which LMS? Default to SCORM 1.2 — widest compatibility.

---

## CRITICAL PITFALL (read before writing any code)

**Never put SCORM `Initialize()` inside React `useEffect` or an ES module.**

Vite builds `<script type="module">` — deferred, strict mode, own scope. The LMS API is only available synchronously at page load. `useEffect` cleanup also doesn't fire on LMS iframe close.

**The fix:** Inline plain `<script>` in `index.html`, running on `DOMContentLoaded`, exposing `window._scorm`. React calls it via a TypeScript bridge (`src/scorm.ts`).

---

## Reference files — what each covers

| File | When to read |
|------|-------------|
| `references/decision-tree.md` | Choosing SCORM 1.2 vs 2004 vs xAPI — mandatory questions, LMS compatibility table |
| `references/scorm-12.md` | Full SCORM 1.2 implementation: inline script, TypeScript bridge, manifest, data model |
| `references/scorm-2004.md` | Full SCORM 2004 4th Ed: inline script with `courseCompleted` flag, suspend vs normal exit, TypeScript bridge, manifest |
| `references/interactions.md` | Per-question interaction reporting for both SCORM 1.2 and 2004 — field names differ! |
| `references/suspend-data.md` | `cmi.suspend_data` + `cmi.location` for mid-course resume — read when the learner needs to pick up where they left off |
| `references/xapi.md` | Full XAPIService class, React integration, `tincan.xml`, per-question `answered` statements, `keepalive` for page unload |
| `references/cmi5.md` | cmi5 launch protocol, required verbs, MoveOn policy |
| `references/lms-compatibility.md` | Per-LMS quirks table — SuccessFactors edge cases, Moodle grading settings, Cornerstone versions |
| `references/packaging-vite.md` | `base: './'`, SCORM and xAPI packaging scripts (`adm-zip`), ZIP structure, standalone packaging |
| `references/testing-checklist.md` | Pre-delivery checklist, SCORM Cloud smoke test steps, debugging 400 errors and missing `terminated` |

---

## Quick API reference

| Field | SCORM 1.2 | SCORM 2004 |
|-------|-----------|------------|
| API object | `window.API` | `window.API_1484_11` |
| Initialize | `LMSInitialize("")` | `Initialize("")` |
| Set value | `LMSSetValue(el, val)` | `SetValue(el, val)` |
| Commit | `LMSCommit("")` | `Commit("")` |
| Terminate | `LMSFinish("")` | `Terminate("")` |
| Completion | `cmi.core.lesson_status` — `incomplete`/`passed`/`failed` | `cmi.completion_status` + `cmi.success_status` — separate fields! |
| Score | `cmi.core.score.raw` (0–100) | `cmi.score.raw` + `cmi.score.scaled` (-1 to 1) |
| Session time | `HH:MM:SS` | `PTxHxMxS` |
| Mid-course exit | `LMSFinish("")` | `cmi.exit = 'suspend'` before `Terminate("")` |
| Resume data | `cmi.suspend_data` (~4 KB max) | `cmi.suspend_data` (64 KB max) |
| Bookmark | `cmi.core.lesson_location` | `cmi.location` |
| Interaction learner response | `student_response` | `learner_response` |

---

## Delivery checklist

Before sending the ZIP to the client:

- [ ] `base: './'` in `vite.config.ts`
- [ ] Manifest (`imsmanifest.xml` or `tincan.xml`) in `public/` — at ZIP root, not in a subfolder
- [ ] For xAPI: `imsmanifest.xml` excluded from ZIP (packer filter)
- [ ] SCORM init is inline `<script>` in `index.html`, not in React
- [ ] `beforeunload` + `pagehide` listeners registered
- [ ] SCORM 2004: `cmi.exit = 'suspend'` on mid-course exit, `'normal'` only after quiz completion
- [ ] Tested on SCORM Cloud — score, completion, and success_status all appear in transcript
- [ ] Tested on actual client LMS — see `references/testing-checklist.md`
