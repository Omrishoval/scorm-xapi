# SCORM 1.2 — Implementation Reference

## How it works

The LMS injects `window.API` into the iframe's window (or a parent window). Your code calls it synchronously — no HTTP, no async. The LMS saves whatever you set before `LMSFinish("")`.

There is no separate completion vs. pass/fail: `lesson_status` is the single field that carries both (`incomplete`, `passed`, `failed`, `completed`, `browsed`, `not attempted`).

---

## Critical: Vite/React placement

The SCORM handler MUST be an **inline plain `<script>`** in `index.html`, not inside React or any ES module. Vite builds as `<script type="module">` which runs deferred — by then the LMS may have already given up looking for your API calls.

---

## index.html inline script

```html
<script>
(function () {
  var api = null;
  var sessionStart = 0;

  function findAPI(win) {
    var tries = 0;
    while (tries <= 10) {
      if (win.API) return win.API;
      if (win.parent && win.parent !== win) { win = win.parent; }
      else if (win.opener) { win = win.opener; }
      else break;
      tries++;
    }
    return null;
  }

  function initSCORM() {
    api = findAPI(window);
    if (!api) { console.log('[SCORM 1.2] standalone mode'); return; }
    var r = api.LMSInitialize('');
    if (r === 'true') {
      sessionStart = Date.now();
      api.LMSSetValue('cmi.core.lesson_status', 'incomplete');
      api.LMSCommit('');
    }
  }

  function terminateSCORM() {
    if (!api) return;
    var elapsed = Math.floor((Date.now() - sessionStart) / 1000);
    var h = Math.floor(elapsed / 3600);
    var m = Math.floor((elapsed % 3600) / 60);
    var s = elapsed % 60;
    var pad = function(n) { return String(n).padStart(2, '0'); };
    api.LMSSetValue('cmi.core.session_time', pad(h) + ':' + pad(m) + ':' + pad(s));
    api.LMSCommit('');
    api.LMSFinish('');
    api = null;
  }

  window._scorm = {
    available: function () { return !!api; },
    setValue:  function (el, val) { if (api) api.LMSSetValue(el, val); },
    getValue:  function (el) { return api ? api.LMSGetValue(el) : ''; },
    commit:    function () { if (api) api.LMSCommit(''); },
    terminate: terminateSCORM,
    suspend:   terminateSCORM  // 1.2 has no exit concept — LMSFinish is the only exit
  };

  document.addEventListener('DOMContentLoaded', initSCORM);
  window.addEventListener('beforeunload', terminateSCORM);
  window.addEventListener('pagehide', terminateSCORM);  // iOS / Safari
})();
</script>
```

---

## TypeScript bridge (src/scorm.ts)

```typescript
const S = () => (window as any)._scorm as {
  available: () => boolean;
  setValue: (el: string, val: string) => void;
  getValue: (el: string) => string;
  commit: () => void;
  terminate: () => void;
} | undefined;

export const scormAvailable = () => !!(S()?.available());
export const scormTerminate  = () => S()?.terminate();

export function scorm12ReportQuiz(score: number, total: number, passMark = 60): void {
  const s = S();
  if (!s?.available()) return;
  const raw = Math.round((score / total) * 100);
  const passed = raw >= passMark;
  s.setValue('cmi.core.score.raw', String(raw));
  s.setValue('cmi.core.score.min', '0');
  s.setValue('cmi.core.score.max', '100');
  s.setValue('cmi.core.lesson_status', passed ? 'passed' : 'failed');
  s.commit();
}
```

---

## Data model quick reference

| Field | Description | Values |
|-------|-------------|--------|
| `cmi.core.lesson_status` | Combined completion + success | `incomplete`, `passed`, `failed`, `completed`, `browsed`, `not attempted` |
| `cmi.core.score.raw` | Score 0–100 | String |
| `cmi.core.score.min` | Min score | String (usually `"0"`) |
| `cmi.core.score.max` | Max score | String (usually `"100"`) |
| `cmi.core.session_time` | Session duration | `HH:MM:SS` |
| `cmi.core.lesson_location` | Bookmark / resume point | String (max ~255 chars) |
| `cmi.suspend_data` | Arbitrary resume state | JSON string (max ~4,000 chars) |

---

## imsmanifest.xml

Place in `public/` — Vite copies it to `dist/` automatically.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest identifier="COURSE-ID-HERE" version="1.2"
  xmlns="http://www.imsproject.org/xsd/imscp_rootv1p1p2"
  xmlns:adlcp="http://www.adlnet.org/xsd/adlcp_rootv1p2"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.imsproject.org/xsd/imscp_rootv1p1p2 imscp_rootv1p1p2.xsd
                      http://www.adlnet.org/xsd/adlcp_rootv1p2 adlcp_rootv1p2.xsd">
  <metadata>
    <schema>ADL SCORM</schema>
    <schemaversion>1.2</schemaversion>
  </metadata>
  <organizations default="ORG-1">
    <organization identifier="ORG-1">
      <title>COURSE TITLE HERE</title>
      <item identifier="ITEM-1" identifierref="RES-1" isvisible="true">
        <title>COURSE TITLE HERE</title>
        <adlcp:masteryscore>80</adlcp:masteryscore>
      </item>
    </organization>
  </organizations>
  <resources>
    <resource identifier="RES-1" type="webcontent" href="index.html" adlcp:scormtype="sco">
      <file href="index.html"/>
    </resource>
  </resources>
</manifest>
```

---

## Common mistakes

| Mistake | Effect | Fix |
|---------|--------|-----|
| Init in `useEffect` or ES module | LMS injects API before React loads — `window.API` not found | Inline `<script>` in `index.html` |
| `lesson_status` stays `incomplete` | Learner shows as not completed | Set to `passed`/`failed`/`completed` before `LMSFinish` |
| `score.raw` as number, not string | Some LMSes reject it | Always `String(raw)` |
| `masteryscore` missing in manifest | LMS uses default (varies) | Always set explicitly |
| Missing `LMSCommit` after `SetValue` | Data lost if tab closes before implicit commit | Always commit after important writes |
| `session_time` format wrong | LMS may reject | Must be `HH:MM:SS` — always zero-pad |

---

## Quiz interaction reporting in SCORM 1.2

See `references/interactions.md` for the full guide.

Short version: SCORM 1.2 interactions are `cmi.interactions.N.*` with `student_response` (not `learner_response`). Many LMSes ignore them or display them poorly. The meaningful reporting is via `lesson_status` + `score.raw`.
