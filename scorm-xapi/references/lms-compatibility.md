# LMS Compatibility Reference

## How to read this table

- **Recommended standard**: What reliably works based on known behavior
- **Reports score**: Whether the LMS natively displays score in its UI
- **Reports interactions**: Whether per-question data appears in the LMS UI
- **Notes**: Known quirks or gotchas

If a field says **Unknown / verify**, do not assume — test with SCORM Cloud first, then deliver.

---

## Compatibility table

| LMS | Recommended | SCORM 1.2 | SCORM 2004 | xAPI | Reports score | Reports interactions | Notes |
|-----|-------------|-----------|------------|------|--------------|---------------------|-------|
| **SuccessFactors** | SCORM 1.2 | ✅ | ⚠️ Varies | ❌ Usually not | ✅ | ❌ | SCORM 2004 support depends on SF version. Confirm with client. |
| **SAP** | SCORM 1.2 | ✅ | ⚠️ Varies | ❌ | ✅ | ❌ | Same engine as SuccessFactors in most deployments. |
| **Moodle < 2.7** | SCORM 1.2 | ✅ | ❌ | ❌ | ✅ | ⚠️ Limited | No SCORM 2004 support. |
| **Moodle 3+ / 4.x** | SCORM 2004 | ✅ | ✅ | ✅ with plugin | ✅ | ✅ | xAPI requires SCORM External module or Logstore xAPI plugin. |
| **SCORM Cloud** | SCORM 2004 | ✅ | ✅ | ✅ | ✅ | ✅ | Best test platform. Full support for all standards. |
| **Canvas (Instructure)** | SCORM 1.2 or 2004 | ✅ | ✅ | ❌ Native | ✅ | ⚠️ Limited | Canvas shows score and status but not interaction detail. Verify version. |
| **Blackboard** | SCORM 1.2 | ✅ | ⚠️ | ❌ | ✅ | ❌ | SCORM 2004 support inconsistent across Blackboard versions. Stick to 1.2 unless client confirms. |
| **Docebo** | SCORM 2004 | ✅ | ✅ | ✅ | ✅ | ✅ | Good xAPI support with built-in LRS. |
| **TalentLMS** | SCORM 2004 | ✅ | ✅ | ✅ | ✅ | ⚠️ | xAPI support via Tin Can. Interaction detail varies. |
| **Absorb LMS** | SCORM 2004 | ✅ | ✅ | ✅ | ✅ | ✅ | Good xAPI support. |
| **Cornerstone OnDemand (old)** | SCORM 1.2 | ✅ | ⚠️ | ❌ | ✅ | ❌ | Older Cornerstone installs have broken SCORM 2004 implementation. Verify. |
| **Cornerstone (modern)** | SCORM 2004 | ✅ | ✅ | ✅ | ✅ | ⚠️ | Unknown / verify — varies by client configuration. |
| **Unknown LMS** | SCORM 1.2 | ✅ | Unknown | Unknown | Unknown | Unknown | Default to 1.2. Ask client for examples of courses that work. |

---

## SuccessFactors — extra notes

SuccessFactors is widely used in enterprise clients and has the most edge cases:

- `cmi.core.lesson_status` must be set to `passed` or `completed` — `failed` alone may not register as "done"
- Some SF versions require `masteryscore` in the manifest to show pass/fail
- SF may cache the manifest — a re-upload might not pick up changes without clearing the cache
- Test with a dedicated test user, not an admin — SF sometimes behaves differently for admin sessions

---

## Moodle — extra notes

- Moodle's SCORM player has a "Grading method" setting: `Learning objects`, `Highest grade`, `Average grade`, `Sum of grades`. Confirm with the client which one is active.
- If `completion_status` is set by content (`objectiveSetByContent="true"` in manifest), Moodle respects it. Otherwise it uses the highest score.
- For xAPI on Moodle 4.x: the H5P plugin and the Logstore xAPI plugin both need to be installed. Confirm with the sysadmin.

---

## Testing recommendation

Always test on SCORM Cloud before delivering to the client LMS:

1. Upload the ZIP to SCORM Cloud
2. Launch as a test learner
3. Complete the course
4. Check the transcript — verify score, completion_status, success_status, and interactions
5. Only then deliver to the client LMS

SCORM Cloud's transcript viewer shows exactly what data the LMS received. If it's wrong there, it will be wrong everywhere.
