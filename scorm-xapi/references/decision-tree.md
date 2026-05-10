# Decision Tree — Which Standard to Use

## Mandatory questions before any implementation

Ask all of these before writing a single line of code:

1. **Which LMS?** (name + version if known)
2. **What needs to be reported?** (completion only? score? pass/fail? individual questions?)
3. **Does the learner need to resume mid-course?** (bookmark / suspend_data)
4. **Is there a quiz?** If yes: how many questions, what passing threshold?
5. **Where does the content run?** (inside LMS iframe, or externally — mobile, video player, external app)
6. **What format does the client expect for the ZIP?**

Do not implement before you have answers to these questions.

---

## Standard selection

```
Does the content run OUTSIDE an LMS iframe?
  ├─ Yes → xAPI (or cmi5 if LMS needs enrollment control)
  └─ No (runs inside LMS)
       │
       Does the LMS support SCORM 2004?
       ├─ No / Unknown → SCORM 1.2  (safest default)
       └─ Yes
            │
            Do you need any of these?
            - Separate completion vs. pass/fail status
            - Interactions detail (per-question reports)
            - Suspend data > 4KB
            - Progress measure (cmi.progress_measure)
            ├─ Yes → SCORM 2004 4th Edition
            └─ No → SCORM 1.2 is simpler and sufficient
```

---

## LMS → standard mapping

| LMS | Recommended standard | Notes |
|-----|---------------------|-------|
| SuccessFactors | SCORM 1.2 | SCORM 2004 support varies by version; 1.2 is safest |
| SAP | SCORM 1.2 | Same as SuccessFactors |
| Moodle < 2.7 | SCORM 1.2 | No SCORM 2004 support |
| Moodle 3+ / 4.x | SCORM 2004 4th Ed | Full support; also supports xAPI with plugin |
| SCORM Cloud | SCORM 2004 4th Ed | Best testing platform; also supports xAPI/cmi5 |
| Canvas | SCORM 1.2 or 2004 | Both work; verify with client |
| Blackboard | SCORM 1.2 | 2004 support inconsistent |
| Docebo | SCORM 2004 | Also supports xAPI |
| TalentLMS | SCORM 2004 | Also supports xAPI/cmi5 |
| Absorb | SCORM 2004 | Also supports xAPI |
| Cornerstone (old) | SCORM 1.2 | 2004 support unreliable |
| Unknown LMS | SCORM 1.2 | Conservative default |
| External tracking (mobile, video, sim) | xAPI | Runs outside iframe |
| xAPI + LMS enrollment + MoveOn rules | cmi5 | Most complex option |

---

## When NOT to use each standard

| Standard | Avoid when |
|----------|-----------|
| SCORM 1.2 | Need detailed interaction reports per question; need > 4KB resume state; need separate pass vs complete |
| SCORM 2004 | LMS is old corporate system (SuccessFactors, old Cornerstone); client unsure of LMS version |
| xAPI | Need the LMS to natively display score/status without a separate LRS setup |
| cmi5 | Client's LMS doesn't have cmi5 support (most don't as of 2025) |

---

## Handling the "I don't know the LMS" case

If the client cannot tell you which LMS they use:

1. Default to **SCORM 1.2** — runs on virtually every LMS
2. Ask if they have any examples of other SCORM courses that work (check manifest version)
3. Ask if they can test with a SCORM Cloud free trial before delivery
4. Build with the SCORM 1.2 handler but structure the code so upgrading to 2004 is a manifest + handler swap

Never use SCORM 2004 as a "modern default" when the LMS is unknown.
