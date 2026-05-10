# Testing Checklist — SCORM & xAPI

## Before you zip and upload

- [ ] `base: './'` is set in `vite.config.ts`
- [ ] Manifest file (`imsmanifest.xml` or `tincan.xml`) is in `public/`
- [ ] For xAPI ZIP: `imsmanifest.xml` is excluded by the packer filter
- [ ] Manifest is at ZIP root (not inside a subfolder)
- [ ] SCORM init code is an **inline `<script>`** in `index.html` — NOT in a React component or ES module
- [ ] `beforeunload` + `pagehide` listeners are registered in the inline script

---

## SCORM Cloud smoke test (do this before delivering to client LMS)

SCORM Cloud is the reference implementation — if it fails there, it will fail everywhere.

1. Upload the ZIP to [SCORM Cloud](https://cloud.scorm.com)
2. Launch as a test learner (not admin — admin sessions sometimes behave differently)
3. Open browser DevTools → Network tab — filter by XHR/Fetch
4. Watch for LMS API calls in the Console tab

### What to verify

| Check | How to verify |
|-------|---------------|
| `Initialize` was called | Console: `[SCORM] initialized` or no error on launch |
| `Commit` is called after SetValue | No data lost on refresh |
| Score appears in SCORM Cloud transcript | Check Reports → Learner Transcript after completing quiz |
| `completion_status = completed` | Transcript shows "Completed" |
| `success_status = passed` or `failed` | Transcript shows correct pass/fail |
| `Terminate` was called | Transcript shows session end time |
| Data persists after page reload | Close and reopen course — position should restore |

---

## SCORM 1.2 specific checks

- [ ] `lesson_status` is set to `passed` or `failed` (not just `completed`)
- [ ] `score.raw` is between 0–100
- [ ] `session_time` format: `HH:MM:SS` (not ISO duration)
- [ ] Interactions use `student_response` (not `learner_response`)
- [ ] Interaction IDs are short — some LMSes truncate to 4 characters

**SuccessFactors extra:**
- Test with a dedicated test user (not admin account)
- Check that `masteryscore` is set in `imsmanifest.xml` if SF needs to show pass/fail
- If SF shows "In Progress" after completion, check that `lesson_status` is `passed` not `completed`

---

## SCORM 2004 specific checks

- [ ] `completion_status` = `completed` AND `success_status` = `passed`/`failed` (both fields, not one)
- [ ] `score.scaled` is between -1.0 and 1.0 (not raw percentage)
- [ ] `session_time` format: `PTxHxMxS` (not `HH:MM:SS`)
- [ ] Interactions use `learner_response` (not `student_response`)
- [ ] `cmi.exit` = `suspend` on mid-course exit, `normal` after quiz completion
- [ ] Test mid-course resume: close at slide 3, reopen — should land back at slide 3

**Exit mode test (critical):**
1. Launch the course
2. Navigate to slide 3 (don't finish)
3. Close the browser tab
4. Reopen and relaunch — should resume at slide 3, not restart
5. If it restarts: `cmi.exit` is being set to `normal` on all exits — fix to `suspend`

---

## xAPI specific checks

Open browser DevTools → Network tab, filter by `statements`:

- [ ] `initialized` statement fires on course load
- [ ] `answered` statement fires for each quiz question
- [ ] `scored` statement fires after final quiz question
- [ ] `passed` or `failed` statement fires after quiz
- [ ] `terminated` statement fires when learner exits (check `keepalive: true` is set)
- [ ] HTTP 200 response for each statement (not 400 or 401)
- [ ] Actor object has correct `mbox` (learner email)
- [ ] `registration` UUID matches the enrollment

**Debugging 400 errors:**
The LRS returns 400 when the statement format is invalid. Log the response body:
```javascript
fetch(url, options)
  .then(r => { if (!r.ok) r.text().then(t => console.error('[xAPI] rejection:', t)); })
```
Common causes: missing `objectType: 'Agent'` on actor, invalid verb IRI, `actor` not parsed correctly from URL params.

**Debugging terminated not received:**
If `terminated` never appears in the LRS after closing:
- Confirm `keepalive: true` is set on the `finish()` fetch call
- Add a `beforeunload` listener directly in `useEffect` as backup
- On iOS/Safari: use `pagehide` event (not `beforeunload`)

---

## Delivery checklist (before sending to client)

- [ ] Tested on SCORM Cloud — all transcript fields correct
- [ ] Tested on the **actual client LMS** (see `lms-compatibility.md` for known quirks)
- [ ] Tested as a non-admin learner account
- [ ] Verified score appears in LMS reports UI
- [ ] Verified completion status appears in LMS reports UI
- [ ] ZIP file size is reasonable (< 50 MB recommended, < 100 MB max for most LMSes)
- [ ] Course launches without console errors in Chrome and Edge
