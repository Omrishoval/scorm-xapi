# xAPI / Tin Can — Implementation Reference

## What xAPI is

xAPI sends JSON "statements" via HTTP POST to a Learning Record Store (LRS). No iframe, no synchronous JS API, no LMS injection. The course can run anywhere — mobile browser, standalone app, external website.

Every statement follows: **Actor** (who) → **Verb** (did what) → **Object** (to what)

Optional **Result**: `score`, `success`, `completion`, `response`, `duration` (ISO 8601: `PT10M30S`)
Optional **Context**: `registration` (links the statement to an LMS enrollment attempt)

---

## Launch mechanism

The LMS (or SCORM Cloud in xAPI mode) launches the course with URL params:

```
index.html?endpoint=https://lrs.example.com/xapi/
          &auth=Basic%20dXNlcjpwYXNz
          &actor=%7B%22mbox%22%3A%22mailto%3Auser%40example.com%22%7D
          &activity_id=https://example.com/course/1
          &registration=UUID
```

The `auth` param is already Base64-encoded by most LMSes — pass it as-is to the `Authorization` header.

---

## src/xapi.ts — Full service class

```typescript
export interface QuizAnswer {
  questionId: string;
  questionText: string;
  learnerChoice: number;
  correctChoice: number;
  options: string[];
}

interface XAPIConfig {
  endpoint: string;
  auth: string;
  actor: { mbox?: string; name?: string; objectType: 'Agent' };
  activityId: string;
  registration?: string;
}

const VERBS = {
  initialized: { id: 'http://adlnet.gov/expapi/verbs/initialized',  display: { 'he-IL': 'התחיל',      'en-US': 'initialized' } },
  answered:    { id: 'http://adlnet.gov/expapi/verbs/answered',     display: { 'he-IL': 'ענה',         'en-US': 'answered'    } },
  scored:      { id: 'http://adlnet.gov/expapi/verbs/scored',       display: { 'he-IL': 'קיבל ציון',  'en-US': 'scored'      } },
  completed:   { id: 'http://adlnet.gov/expapi/verbs/completed',    display: { 'he-IL': 'השלים',       'en-US': 'completed'   } },
  passed:      { id: 'http://adlnet.gov/expapi/verbs/passed',       display: { 'he-IL': 'עבר',         'en-US': 'passed'      } },
  failed:      { id: 'http://adlnet.gov/expapi/verbs/failed',       display: { 'he-IL': 'נכשל',        'en-US': 'failed'      } },
  terminated:  { id: 'http://adlnet.gov/expapi/verbs/terminated',   display: { 'he-IL': 'סיים',        'en-US': 'terminated'  } },
};

class XAPIService {
  private cfg: XAPIConfig | null = null;
  private startTime = Date.now();
  private finished = false;

  constructor() {
    const p = new URLSearchParams(window.location.search);
    const endpoint = p.get('endpoint');
    const auth     = p.get('auth');
    const actorStr = p.get('actor');
    if (endpoint && auth) {
      let actor;
      try { actor = JSON.parse(decodeURIComponent(actorStr || '{}')); } catch { actor = null; }
      if (actor) {
        // SCORM Cloud sends actor in legacy Tin Can format — normalize to xAPI 1.0:
        // name / mbox / account arrive as arrays → unwrap to single values
        // account uses old field names: accountServiceHomePage → homePage, accountName → name
        if (Array.isArray(actor.name))    actor.name    = actor.name[0];
        if (Array.isArray(actor.mbox))    actor.mbox    = actor.mbox[0];
        if (Array.isArray(actor.account)) actor.account = actor.account[0];
        if (actor.account) {
          if (actor.account.accountServiceHomePage) {
            actor.account.homePage = actor.account.accountServiceHomePage;
            delete actor.account.accountServiceHomePage;
          }
          if (actor.account.accountName) {
            actor.account.name = actor.account.accountName;
            delete actor.account.accountName;
          }
        }
      }
      this.cfg = {
        endpoint:     decodeURIComponent(endpoint),
        auth:         decodeURIComponent(auth),
        actor:        actor || { mbox: 'mailto:anonymous@example.com', name: 'Anonymous', objectType: 'Agent' },
        activityId:   decodeURIComponent(p.get('activity_id') || 'https://experteam.co.il/courses/default'),
        registration: p.get('registration') ?? undefined,
      };
    }
  }

  private buildStmt(verbKey: keyof typeof VERBS, extra: object = {}) {
    return {
      actor: this.cfg!.actor,
      verb:  VERBS[verbKey],
      object: {
        id:         this.cfg!.activityId,
        objectType: 'Activity',
        definition: { type: 'http://adlnet.gov/expapi/activities/course' },
      },
      timestamp: new Date().toISOString(),
      ...(this.cfg!.registration ? { context: { registration: this.cfg!.registration } } : {}),
      ...extra,
    };
  }

  private send(verbKey: keyof typeof VERBS, extra: object = {}, keepalive = false) {
    if (!this.cfg) { console.log(`[xAPI offline] ${verbKey}`, extra); return; }
    const stmt = this.buildStmt(verbKey, extra);
    // Normalize endpoint — always end with /
    const base = this.cfg.endpoint.endsWith('/') ? this.cfg.endpoint : `${this.cfg.endpoint}/`;
    fetch(`${base}statements`, {
      method: 'POST',
      keepalive,  // allows fetch to survive page unload
      headers: {
        'Content-Type':             'application/json; charset=utf-8',
        'Authorization':            this.cfg.auth,
        'X-Experience-API-Version': '1.0.3',
      },
      body: JSON.stringify(stmt),
    })
      .then(r => { if (!r.ok) r.text().then(t => console.error('[xAPI] 4xx body:', t)); })
      .catch(e => console.error('[xAPI] send error', e));
  }

  private isoDuration(): string {
    const s = Math.floor((Date.now() - this.startTime) / 1000);
    return `PT${Math.floor(s / 3600)}H${Math.floor((s % 3600) / 60)}M${s % 60}S`;
  }

  /** Call on course load (App useEffect mount) */
  init() { this.send('initialized'); }

  /** Call once per quiz question as learner answers */
  answered(questionId: string, questionText: string, learnerChoice: number, correctChoice: number, options: string[], isCorrect: boolean) {
    const activityId = this.cfg
      ? `${this.cfg.activityId}/questions/${questionId}`
      : `https://experteam.co.il/courses/default/questions/${questionId}`;
    this.send('answered', {
      object: {
        id:         activityId,
        objectType: 'Activity',
        definition: {
          type:                    'http://adlnet.gov/expapi/activities/cmi.interaction',
          interactionType:         'choice',
          description:             { 'en-US': questionText },
          choices:                 options.map((text, i) => ({ id: `option_${i}`, description: { 'en-US': text } })),
          correctResponsesPattern: [`option_${correctChoice}`],
        },
      },
      result: {
        response: `option_${learnerChoice}`,
        success:  isCorrect,
        duration: this.isoDuration(),
      },
    });
  }

  /** Call after quiz with raw score */
  score(raw: number, max: number) {
    this.send('scored', { result: { score: { raw, scaled: raw / max, min: 0, max } } });
  }

  /** Call after quiz with pass/fail result */
  complete(passed: boolean, raw: number, max: number) {
    this.send(passed ? 'passed' : 'failed', {
      result: { success: passed, completion: true, score: { raw, scaled: raw / max, min: 0, max } },
    });
  }

  /** Call on course exit — keepalive so the request survives page unload. Idempotent. */
  finish() {
    if (this.finished) return;
    this.finished = true;
    this.send('terminated', { result: { completion: true, duration: this.isoDuration() } }, true);
  }
}

export const xapi = new XAPIService();
```

---

## React integration (App.tsx)

```tsx
import { useEffect } from 'react';
import { xapi } from './xapi';

export default function App() {
  useEffect(() => {
    xapi.init();
    const onUnload = () => xapi.finish();
    window.addEventListener('beforeunload', onUnload);
    window.addEventListener('pagehide', onUnload);      // iOS / Safari
    return () => {
      window.removeEventListener('beforeunload', onUnload);
      window.removeEventListener('pagehide', onUnload);
    };
  }, []);
  // ...
}
```

---

## After quiz (in FinalQuizSection)

```tsx
import { xapi } from './xapi';

// As each question is answered:
xapi.answered(questionId, questionText, learnerChoice, correctChoice, options, isCorrect);

// After final question:
xapi.score(rawScore, totalQuestions);
xapi.complete(rawScore >= passingThreshold, rawScore, totalQuestions);
```

---

## Close button

```tsx
<button onClick={() => {
  xapi.finish();
  try { (window.top as Window).close(); } catch { window.close(); }
}}>
  Close & Exit
</button>
```

---

## tincan.xml manifest

Place in `public/`. Include BOTH language variants if the course is bilingual.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<tincan xmlns="http://projecttincan.com/tincan.xsd">
  <activities>
    <activity id="https://DOMAIN/courses/COURSE-ID"
              type="http://adlnet.gov/expapi/activities/course">
      <name lang="en-US">COURSE TITLE</name>
      <name lang="he-IL">שם הקורס</name>
      <description lang="en-US">COURSE DESCRIPTION</description>
      <launch lang="en-US">index.html</launch>
    </activity>
  </activities>
</tincan>
```

The ZIP must NOT contain `imsmanifest.xml` — the LMS will detect it as SCORM if it's present.

---

## Authentication

| Method | `Authorization` header | When |
|--------|----------------------|------|
| Basic Auth | `Basic base64(user:pass)` | SCORM Cloud, simple LRS |
| Bearer token | `Bearer <token>` | cmi5, enterprise LRS |
| OAuth 1.0 | `OAuth realm=...` | Enterprise, rarely needed |

The URL param `auth` is already encoded by most LMSes — pass as-is. Do not re-encode.

---

## Common issues

| Issue | Cause | Fix |
|-------|-------|-----|
| 400: `name` is array / `accountServiceHomePage` unknown | SCORM Cloud sends actor in legacy Tin Can format, not xAPI 1.0 | Normalize actor in constructor — see code above |
| 400 from LRS (other) | Statement format error | Log `r.text()` in the `.then()` handler to see the exact rejection reason |
| `terminated` fires twice | `beforeunload` fires after `window.close()` in close button handler | Add `finished` guard to `finish()` — it's idempotent in the code above |
| `terminated` not received | `fetch` aborted at page unload | `keepalive: true` is set in `finish()` — also register `pagehide` for iOS/Safari |
| `window.close()` ignored | Called inside `setTimeout` — not a synchronous user gesture | Call `window.close()` directly in the click handler, without `setTimeout` |
| Double-fired `init` | Component remounts in StrictMode | Guard with `useRef` flag, or move to vanilla `init` outside React |
| Actor not identified | URL params missing / malformed | Add fallback anonymous actor |
| Score not showing in LMS | LMS needs `passed`/`failed` not just `scored` | Send `complete(passed, raw, max)` after `score()` |

---

## Warning: xAPI without a data contract

Without agreeing on a data contract (xAPI Profile), different systems will interpret verbs differently. If the LRS and LMS are not configured to recognize your activity IDs and verb patterns, reports will be empty or inconsistent.

Before implementing xAPI, confirm:
- Which LRS? (SCORM Cloud, Watershed, Learning Locker, etc.)
- Which verbs does the LMS track for completion / passing?
- Is there an xAPI Profile the client expects?
