# cmi5 — xAPI with LMS Structure

cmi5 = xAPI statements + a strict LMS launch protocol. Use when the client needs xAPI's data richness AND SCORM-like LMS enrollment control (defined pass/complete rules, sequencing).

---

## How cmi5 differs from plain xAPI

| Aspect | xAPI | cmi5 |
|--------|------|-------|
| Auth | Basic / OAuth from URL | JWT token fetched from `fetch_url` |
| Launch control | Manual URL params | LMS fully controls via spec |
| Pass/complete rules | Ad-hoc | MoveOn policy (Passed, Completed, Both, Either) |
| Required verbs | None | Initialized, Completed, Passed/Failed, Terminated, Abandoned |
| Content unit | None | Assignable Unit (AU) — like a SCORM SCO |

---

## MoveOn policies (set in course package, not in code)

| Policy | LMS marks complete when |
|--------|------------------------|
| `Passed` | AU sends Passed statement |
| `Completed` | AU sends Completed statement |
| `CompletedAndPassed` | Both sent |
| `CompletedOrPassed` | Either one |

---

## Required verb sequence (per AU session)

```
LMS sends:    Launched  →  (AU starts)
AU sends:     Initialized
              (learning happens)
AU sends:     Completed   (content done)
AU sends:     Passed / Failed  (if score-based)
AU sends:     Terminated  (on exit / beforeunload)
```

If the learner quits without finishing, the LMS may send `Abandoned` after a timeout.

---

## Launch URL params

The LMS launches: 
```
index.html?endpoint=https://lrs.example.com/xapi/
          &fetch_url=https://lms.example.com/token/UUID
          &registration=UUID
          &activityId=https://example.com/course/1
          &actor=...
```

Unlike plain xAPI, the auth token is NOT in the URL — it must be fetched.

---

## Minimal TypeScript bootstrap

```typescript
// src/cmi5.ts
interface Cmi5Config {
  endpoint: string;
  token: string;
  actor: object;
  activityId: string;
  registration: string;
}

async function fetchToken(fetchUrl: string): Promise<string> {
  const res = await fetch(fetchUrl, { method: 'POST' });
  const data = await res.json();
  // Field name varies by LMS; try common variants
  return data['auth-token'] ?? data.authToken ?? data.token ?? '';
}

export async function initCmi5(): Promise<Cmi5Config> {
  const p = new URLSearchParams(window.location.search);
  const token = await fetchToken(decodeURIComponent(p.get('fetch_url')!));
  return {
    endpoint:     decodeURIComponent(p.get('endpoint')!),
    token,
    actor:        JSON.parse(decodeURIComponent(p.get('actor') || '{}')),
    activityId:   decodeURIComponent(p.get('activityId')!),
    registration: p.get('registration')!,
  };
}

function sendStatement(cfg: Cmi5Config, verb: object, result?: object) {
  const stmt = {
    actor:  cfg.actor,
    verb,
    object: { id: cfg.activityId, objectType: 'Activity' },
    context: {
      registration: cfg.registration,
      contextActivities: {
        category: [{ id: 'https://w3id.org/xapi/cmi5/context/categories/cmi5' }]
      }
    },
    timestamp: new Date().toISOString(),
    ...(result ? { result } : {}),
  };
  fetch(`${cfg.endpoint}statements`, {
    method: 'POST',
    headers: {
      'Content-Type':             'application/json',
      'Authorization':            `Bearer ${cfg.token}`,
      'X-Experience-API-Version': '1.0.3',
    },
    body: JSON.stringify(stmt),
  });
}

// Verb URIs (cmi5 spec)
const V = {
  initialized: { id: 'http://adlnet.gov/expapi/verbs/initialized',  display: { 'en-US': 'initialized' } },
  completed:   { id: 'http://adlnet.gov/expapi/verbs/completed',    display: { 'en-US': 'completed'   } },
  passed:      { id: 'http://adlnet.gov/expapi/verbs/passed',       display: { 'en-US': 'passed'      } },
  failed:      { id: 'http://adlnet.gov/expapi/verbs/failed',       display: { 'en-US': 'failed'      } },
  terminated:  { id: 'http://adlnet.gov/expapi/verbs/terminated',   display: { 'en-US': 'terminated'  } },
};

// Usage:
// const cfg = await initCmi5();
// sendStatement(cfg, V.initialized);
// sendStatement(cfg, V.completed,  { completion: true, duration: 'PT5M' });
// sendStatement(cfg, V.passed,     { success: true, score: { scaled: 0.85 } });
// sendStatement(cfg, V.terminated, { duration: 'PT10M30S' });
```

---

## For production use

Consider the `@xapi/cmi5` npm package — it handles the full spec including:
- Token fetching and refresh
- Session management
- Required context activities
- Spec-compliant statement ordering

Install: `npm install @xapi/cmi5`

---

## Standards comparison

| Feature | SCORM 1.2 | SCORM 2004 | xAPI | cmi5 |
|---------|-----------|------------|------|------|
| Communication | JS API in iframe | JS API in iframe | HTTP POST to LRS | HTTP POST to LRS |
| Auth | Implicit (iframe) | Implicit (iframe) | Basic/OAuth | Bearer token |
| Resume data | 4KB | 64KB | Custom state API | Custom state API |
| Tracking detail | Basic | Interactions | Unlimited | Unlimited + structure |
| LMS enrollment | Required | Required | Optional | Required |
| Mobile/offline | No | No | Yes | Yes |
| Adoption (2024) | Very high | High | Growing | Emerging |
