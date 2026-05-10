# Suspend Data — Resume / Bookmark Support

`cmi.suspend_data` stores arbitrary learner state so they can resume mid-course. The LMS persists it between sessions.

**Limits:**
- SCORM 1.2: ~4,000 characters (stay under 4,096)
- SCORM 2004 4th Ed: 64,000 characters

Combined with `cmi.location` (or `cmi.core.lesson_location` in 1.2), you can restore both arbitrary state and the learner's position in the course.

---

## What to store

Keep it minimal — especially in SCORM 1.2:

```json
{
  "slide": 4,
  "completedSlides": [0, 1, 2, 3],
  "quizAnswers": { "q1": 2, "q2": 0 }
}
```

Don't store question text, labels, or anything derivable from the app itself — save every byte in SCORM 1.2.

---

## src/suspend.ts

```typescript
// Works for both SCORM 1.2 and 2004 — the inline script handles field name differences

const LOCATION_FIELD_12   = 'cmi.core.lesson_location';
const LOCATION_FIELD_2004 = 'cmi.location';
const SUSPEND_FIELD       = 'cmi.suspend_data';
const MAX_12_CHARS        = 3900;  // safe buffer under 4096

interface SuspendState {
  slide?: number;
  completedSlides?: number[];
  quizAnswers?: Record<string, number>;
  [key: string]: unknown;
}

const s = () => (window as any)._scorm;

/** Read saved state on course load */
export function loadSuspendData<T extends SuspendState>(): T | null {
  const api = s();
  if (!api?.available()) return null;
  try {
    const raw = api.getValue(SUSPEND_FIELD);
    return raw ? (JSON.parse(raw) as T) : null;
  } catch {
    return null;
  }
}

/** Read the saved bookmark (slide number as string) */
export function loadLocation(): number | null {
  const api = s();
  if (!api?.available()) return null;
  const loc = api.getValue(LOCATION_FIELD_2004) || api.getValue(LOCATION_FIELD_12);
  const n = parseInt(loc, 10);
  return isNaN(n) ? null : n;
}

/** Save state without terminating the session (call periodically or on slide change) */
export function saveSuspendData(state: SuspendState, isScorm12 = false): void {
  const api = s();
  if (!api?.available()) return;

  const json = JSON.stringify(state);
  if (isScorm12 && json.length > MAX_12_CHARS) {
    console.warn('[SCORM] suspend_data too large for 1.2:', json.length, 'chars');
    return;
  }

  const slide = String(state.slide ?? '');
  api.setValue(isScorm12 ? LOCATION_FIELD_12 : LOCATION_FIELD_2004, slide);
  api.setValue(SUSPEND_FIELD, json);
  api.commit();
}

/** Call when learner exits mid-course (not on completion) */
export function exitSuspend(state: SuspendState, isScorm12 = false): void {
  saveSuspendData(state, isScorm12);
  s()?.suspend?.();   // sets cmi.exit = 'suspend' in SCORM 2004; noop in 1.2
}
```

---

## React Integration

```tsx
// App.tsx — load on mount, save on each slide change, suspend on unmount
import { useState, useEffect } from 'react';
import { loadSuspendData, saveSuspendData } from './suspend';

interface AppState {
  slide: number;
  completedSlides: number[];
  quizAnswers: Record<string, number>;
}

export default function App() {
  const [state, setState] = useState<AppState>(() => {
    const saved = loadSuspendData<AppState>();
    return saved ?? { slide: 0, completedSlides: [], quizAnswers: {} };
  });

  // Save on every state change (debounce in production if slides change rapidly)
  useEffect(() => {
    saveSuspendData(state);
  }, [state]);

  // On unmount (course closed mid-session) — use suspend not terminate
  useEffect(() => {
    return () => {
      saveSuspendData(state);
      // window._scorm.suspend() is called by the beforeunload handler in index.html
    };
  }, [state]);

  const goToSlide = (n: number) => {
    setState(prev => ({
      ...prev,
      slide: n,
      completedSlides: [...new Set([...prev.completedSlides, prev.slide])]
    }));
  };

  // ...render logic, start at state.slide
}
```

---

## SCORM 2004: exit mode matters

| Scenario | cmi.exit value | Effect on LMS |
|----------|---------------|---------------|
| Learner completed course | `normal` | LMS records final status, no resume needed |
| Learner quit mid-course | `suspend` | LMS will restore suspend_data on re-entry |
| Browser crash / no Terminate | `time-out` (set by LMS) | LMS may reset status — handle defensively |

The `window._scorm.suspend()` helper sets `cmi.exit = 'suspend'` before calling `Terminate("")`.
The `window._scorm.terminate()` helper sets `cmi.exit = 'normal'`.

In SCORM 1.2 there is no exit status — just save suspend_data and call `LMSFinish("")`.
