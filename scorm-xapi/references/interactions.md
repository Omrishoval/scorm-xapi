# Quiz Interaction Reporting

## What interactions actually give you

SCORM interactions let the LMS record which answer a learner chose per question. In practice:

- **SCORM 1.2**: Most LMSes store interaction data but display it inconsistently or not at all. The meaningful output is still `lesson_status` + `score.raw`.
- **SCORM 2004**: Better support. Moodle, SCORM Cloud, and Docebo show interaction breakdowns. SuccessFactors and Cornerstone often ignore them.
- **xAPI**: The natural format for per-question data. Each answer is a full statement with structured interaction data.

The bottom line: don't promise the client detailed question-by-question reports unless you've verified the LMS actually surfaces that data.

---

## SCORM 1.2 interactions

Field names use `student_response` (not `learner_response` — that's 2004).

```typescript
function reportInteractions12(answers: QuizAnswer[]): void {
  const s = (window as any)._scorm;
  if (!s?.available()) return;

  answers.forEach((ans, i) => {
    s.setValue(`cmi.interactions.${i}.id`,                              ans.questionId);
    s.setValue(`cmi.interactions.${i}.type`,                            'choice');
    s.setValue(`cmi.interactions.${i}.student_response`,                `option_${ans.learnerChoice}`);
    s.setValue(`cmi.interactions.${i}.correct_responses.0.pattern`,     `option_${ans.correctChoice}`);
    s.setValue(`cmi.interactions.${i}.result`,
      ans.learnerChoice === ans.correctChoice ? 'correct' : 'incorrect');
    s.setValue(`cmi.interactions.${i}.weighting`, '1');
    // Note: description and timestamp not available in SCORM 1.2
  });

  s.commit();
}
```

**SCORM 1.2 interaction limitations:**
- Max 4 characters for interaction ID in some LMSes — use short IDs like `"q1"`, `"q2"`
- No `description` field
- No `timestamp` field
- `correct_responses.N.pattern` has only one element for `choice` type

---

## SCORM 2004 interactions

```typescript
export interface QuizAnswer {
  questionId: string;
  questionText: string;
  learnerChoice: number;
  correctChoice: number;
  options: string[];
}

function reportInteractions2004(answers: QuizAnswer[]): void {
  const s = (window as any)._scorm;
  if (!s?.available()) return;

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
}
```

---

## Field name differences: 1.2 vs 2004

| Field | SCORM 1.2 | SCORM 2004 |
|-------|-----------|------------|
| Learner answer | `student_response` | `learner_response` |
| Question text | Not available | `description` |
| Timestamp | Not available | `timestamp` (ISO 8601) |
| Interaction type | `choice`, `true-false`, `fill-in`, `matching`, `performance`, `sequencing`, `likert`, `numeric` | Same + `long-fill-in`, `other` |

---

## choice response pattern format

Both versions use a pattern string that identifies the chosen option:

```
"option_0"    // learner chose option at index 0
"option_2"    // learner chose option at index 2
```

Use a consistent naming convention. `option_N` (zero-indexed) works well. The LMS may display these raw strings, so make them readable if possible.

---

## Approaches: State vs. Journaling

**State (recommended for most courses):**
Record interactions once, at the end of the quiz, with final answers. Simpler, no risk of duplication.

```typescript
// Call once after all questions answered
scorm2004ReportQuiz(score, total, allAnswers, 0.8);
```

**Journaling:**
Record each interaction as the learner answers it. Useful for adaptive courses or when you need timestamps per question.

```typescript
// Call after each question
function onAnswerSelected(questionIndex: number, ans: QuizAnswer) {
  reportSingleInteraction(questionIndex, ans);
}
```

For most projects, **use State** — it's simpler and LMS support is more reliable.

---

## xAPI vs. SCORM for interaction data

xAPI gives you a proper first-class `answered` statement per question with the full interaction definition:

```json
{
  "verb": { "id": "http://adlnet.gov/expapi/verbs/answered" },
  "object": {
    "id": "https://example.com/course/questions/q1",
    "definition": {
      "interactionType": "choice",
      "choices": [...],
      "correctResponsesPattern": ["option_1"]
    }
  },
  "result": { "response": "option_2", "success": false }
}
```

If the client needs reliable per-question reports and has an LRS, use xAPI (see `references/xapi.md`). SCORM interaction data is a best-effort approximation.
