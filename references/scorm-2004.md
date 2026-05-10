# SCORM 2004 4th Edition — Implementation Reference

## How it works

The LMS injects `window.API_1484_11` into the iframe. Same synchronous call pattern as 1.2, but with a richer data model and a critical difference: **completion and success are separate fields**.

A learner can be `completed` (finished the content) but `failed` (didn't pass the quiz). Or `incomplete` and `unknown`. SCORM 1.2 cannot express this distinction.

---

## Critical: Vite/React placement

Same rule as 1.2: the SCORM handler must be an **inline plain `<script>`** in `index.html`, before the React bundle.

---

## Critical: exit mode — suspend vs. normal

**This is the most misunderstood part of SCORM 2004.**

`cmi.exit` tells the LMS *why* the session ended:

| Scenario | Set `cmi.exit` to | Effect |
|----------|------------------|--------|
| Learner completed the course successfully | `normal` | LMS records final status; no resume expected |
| Learner quit mid-course | `suspend` | LMS will restore `suspend_data` on re-entry |
| Browser crash / no Terminate called | `time-out` (set by LMS automatically) | LMS may reset status |

**The `beforeunload` handler must use `suspend`, not `normal`**, because you cannot know at page unload whether the learner completed the course or just closed the tab. Only call `normal` from your explicit "Close & Exit" button after the quiz is submitted.

If `beforeunload` always sends `normal`, a learner who exits mid-course will have their progress wiped.

---

## index.html inline script

```html
<script>
(function () {
  var api = null;
  var sessionStart = 0;
  var courseCompleted = false;  // set to true only after quiz submission

  function findAPI(win) {
    var tries = 0;
    while (tries < 10) {
      if (win.API_1484_11) return win.API_1484_11;
      if (win.parent && win.parent !== win) { win = win.parent; }
      else if (win.opener) { win = win.opener; }
      else break;
      tries++;
    }
    return null;
  }

  function initSCORM() {
    api = findAPI(window);
    if (!api) { console.log('[SCORM 2004] standalone mode'); return; }
    var r = api.Initialize('');
    if (r === 'true') {
      sessionStart = Date.now();
      api.SetValue('cmi.completion_status', 'incomplete');
      api.SetValue('cmi.success_status', 'unknown');
      api.Commit('');
    }
  }

  function closeSCORM(exitMode) {
    if (!api) return;
    var elapsed = Math.floor((Date.now() - sessionStart) / 1000);
    var h = Math.floor(elapsed / 3600);
    var m = Math.floor((elapsed % 3600) / 60);
    var s = elapsed % 60;
    api.SetValue('cmi.session_time', 'PT' + h + 'H' + m + 'M' + s + 'S');
    api.SetValue('cmi.exit', exitMode);
    api.Commit('');
    api.Terminate('');
    api = null;
  }

  window._scorm = {
    available:       function () { return !!api; },
    setValue:        function (el, val) { if (api) api.SetValue(el, val); },
    getValue:        function (el) { return api ? api.GetValue(el) : ''; },
    commit:          function () { if (api) api.Commit(''); },
    terminate:       function () { closeSCORM('normal'); },   // call only after explicit completion
    suspend:         function () { closeSCORM('suspend'); },  // call on mid-course exit
    markCompleted:   function () { courseCompleted = true; }  // React calls this after quiz
  };

  document.addEventListener('DOMContentLoaded', initSCORM);
  // Use suspend on automatic close — learner may not have finished
  window.addEventListener('beforeunload', function () {
    closeSCORM(courseCompleted ? 'normal' : 'suspend');
  });
  window.addEventListener('pagehide', function () {
    closeSCORM(courseCompleted ? 'normal' : 'suspend');
  });
})();
</script>
```

---

## TypeScript bridge (src/scorm.ts)

```typescript
export interface QuizAnswer {
  questionId: string;
  questionText: string;
  learnerChoice: number;
  correctChoice: number;
  options: string[];
}

const S = () => (window as any)._scorm as {
  available: () => boolean;
  setValue: (el: string, val: string) => void;
  getValue: (el: string) => string;
  commit: () => void;
  terminate: () => void;
  suspend: () => void;
  markCompleted: () => void;
} | undefined;

export const scormAvailable  = () => !!(S()?.available());
export const scormTerminate  = () => S()?.terminate();
export const scormSuspend    = () => S()?.suspend();

export function scorm2004ReportQuiz(
  score: number,
  total: number,
  answers: QuizAnswer[],
  passFraction = 0.8
): void {
  const s = S();
  if (!s?.available()) return;
  const scaled = parseFloat((score / total).toFixed(4));
  const passed = scaled >= passFraction;

  s.setValue('cmi.score.raw',         String(score));
  s.setValue('cmi.score.min',         '0');
  s.setValue('cmi.score.max',         String(total));
  s.setValue('cmi.score.scaled',      String(scaled));  // range -1 to 1
  s.setValue('cmi.completion_status', 'completed');
  s.setValue('cmi.success_status',    passed ? 'passed' : 'failed');

  // Report per-question interactions (see references/interactions.md for full guide)
  answers.forEach((ans, i) => {
    const ts = new Date().toISOString().replace('Z', '');
    s.setValue(`cmi.interactions.${i}.id`,                          ans.questionId);
    s.setValue(`cmi.interactions.${i}.type`,                        'choice');
    s.setValue(`cmi.interactions.${i}.description`,                 ans.questionText);
    s.setValue(`cmi.interactions.${i}.timestamp`,                   ts);
    s.setValue(`cmi.interactions.${i}.correct_responses.0.pattern`, `option_${ans.correctChoice}`);
    s.setValue(`cmi.interactions.${i}.learner_response`,            `option_${ans.learnerChoice}`);
    s.setValue(`cmi.interactions.${i}.result`,
      ans.learnerChoice === ans.correctChoice ? 'correct' : 'incorrect');
    s.setValue(`cmi.interactions.${i}.weighting`, '1');
  });

  s.commit();
  s.markCompleted();  // tells beforeunload to use 'normal' exit
}
```

---

## Data model quick reference

| Field | Description | Values |
|-------|-------------|--------|
| `cmi.completion_status` | Did learner finish the content? | `completed`, `incomplete`, `not attempted`, `unknown` |
| `cmi.success_status` | Did learner pass? | `passed`, `failed`, `unknown` |
| `cmi.score.raw` | Raw score | Number (as string) |
| `cmi.score.scaled` | Score as fraction | -1.0 to 1.0 (as string) |
| `cmi.score.min` / `.max` | Score range | Numbers (as strings) |
| `cmi.session_time` | Session duration | ISO 8601: `PT1H30M45S` |
| `cmi.exit` | Why session ended | `normal`, `suspend`, `time-out`, `logout` |
| `cmi.location` | Bookmark | String |
| `cmi.suspend_data` | Resume state | JSON string (max 64,000 chars) |
| `cmi.progress_measure` | Completion fraction | 0.0 to 1.0 |

---

## imsmanifest.xml — SCORM 2004 4th Edition

Place in `public/`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest identifier="COURSE-ID-HERE" version="1.0"
  xmlns="http://www.imsglobal.org/xsd/imscp_v1p1"
  xmlns:adlcp="http://www.adlnet.org/xsd/adlcp_v1p3"
  xmlns:adlseq="http://www.adlnet.org/xsd/adlseq_v1p3"
  xmlns:adlnav="http://www.adlnet.org/xsd/adlnav_v1p3"
  xmlns:imsss="http://www.imsglobal.org/xsd/imsss"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.imsglobal.org/xsd/imscp_v1p1 imscp_v1p1.xsd
    http://www.adlnet.org/xsd/adlcp_v1p3 adlcp_v1p3.xsd
    http://www.adlnet.org/xsd/adlseq_v1p3 adlseq_v1p3.xsd
    http://www.adlnet.org/xsd/adlnav_v1p3 adlnav_v1p3.xsd
    http://www.imsglobal.org/xsd/imsss imsss_v1p0.xsd">
  <metadata>
    <schema>ADL SCORM</schema>
    <schemaversion>2004 4th Edition</schemaversion>
  </metadata>
  <organizations default="ORG-1">
    <organization identifier="ORG-1" adlseq:objectivesGlobalToSystem="false">
      <title>COURSE TITLE HERE</title>
      <item identifier="ITEM-1" identifierref="RES-1">
        <title>COURSE TITLE HERE</title>
        <adlcp:completionThreshold completedByMeasure="false" minProgressMeasure="1.0"/>
        <imsss:sequencing>
          <imsss:objectives>
            <imsss:primaryObjective objectiveID="primary" satisfiedByMeasure="true">
              <imsss:minNormalizedMeasure>0.8</imsss:minNormalizedMeasure>
            </imsss:primaryObjective>
          </imsss:objectives>
          <imsss:deliveryControls completionSetByContent="true" objectiveSetByContent="true"/>
        </imsss:sequencing>
      </item>
    </organization>
  </organizations>
  <resources>
    <resource identifier="RES-1" type="webcontent" adlcp:scormType="sco" href="index.html">
      <file href="index.html"/>
    </resource>
  </resources>
</manifest>
```

---

## Common mistakes

| Mistake | Effect | Fix |
|---------|--------|-----|
| `beforeunload` sends `cmi.exit = 'normal'` always | Mid-course exits wipe suspend_data | Send `'suspend'` unless `courseCompleted` flag is set |
| Setting `completion_status` without `success_status` | LMS shows `completed` but no pass/fail | Always set both fields |
| `score.scaled` not set | Some LMSes show no score | Compute `score / total`, set as string |
| Init inside `useEffect` or ES module | LMS API not found | Inline script in `index.html` |
| Missing `Commit` after `SetValue` | Data lost on unexpected close | Commit after every group of SetValues |
| Using SCORM 1.2 field names (`LMSSetValue`, `cmi.core.*`) | Silently ignored | All 2004 calls are `SetValue`, fields are `cmi.*` (no `core.`) |
