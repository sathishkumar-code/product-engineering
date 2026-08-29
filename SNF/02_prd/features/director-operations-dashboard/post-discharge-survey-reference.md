# Post Discharge FollowUp — Weekly Survey Reference

Source: `SNF Census Dashboard.dc.html`, `wk1Questions(week)` (lines 1108–1130).

| Week | Question | High-Risk-Answer | Feedback Task |
|---|---|---|---|
| 1 | Are you alone at home? | Yes | Arrange caregiver / social work support — resident alone at home |
| 1 | Were you able to pick up your medications from the pharmacy? | No | Resolve pharmacy pickup barrier and confirm meds in hand |
| 1 | Did the home health agency contact you? | No | Call home health agency — start of care not made |
| 1 | Did you get the durable medical equipment that was ordered? | No | Follow up with DME vendor on undelivered equipment |
| 1 | Have you had any falls since discharge? | Yes | Fall follow-up: notify PCP and request home safety evaluation |
| 2 | Have you started receiving home health rehabilitation? | No | Escalate to home health agency — rehab visits not started |
| 2 | Are you taking your medications as prescribed? | No | Pharmacist medication review call |
| 2 | Do you have access to meals? | No | Refer to meal delivery / community nutrition program |
| 2 | Do you have a follow-up appointment with your PCP? | No | Book PCP follow-up appointment and confirm with resident |
| 2 | Have you had any falls since discharge? | Yes | Fall follow-up: notify PCP and request home safety evaluation |
| 3 | Are you taking your medications as prescribed? | No | Pharmacist medication review call |
| 3 | Have you seen any provider since discharge? | No | Arrange provider visit — no post-discharge visit yet |
| 3 | Are you still receiving home health services? | No | Verify home health status — services appear stopped |
| 3 | Have you had any falls since discharge? | Yes | Fall follow-up: notify PCP and request home safety evaluation |
| 3 | Have you needed to go to the ER / urgent care since discharge? | Yes | Review ER visit — confirm diagnosis and readmission risk plan |
| 4 | Are you taking your medications as prescribed? | No | Pharmacist medication review call |
| 4 | Have you seen any provider since discharge? | No | Arrange provider visit — no post-discharge visit yet |
| 4 | Are you still receiving home health services? | No | Verify home health status — services appear stopped |
| 4 | Have you had any falls since discharge? | Yes | Fall follow-up: notify PCP and request home safety evaluation |
| 4 | Have you needed to go to the ER / urgent care since discharge? | Yes | Review ER visit — confirm diagnosis and readmission risk plan |

## Notes

- Weeks 3 and 4 share an identical question set in code (`if (week === 3 || week === 4)`) — no week-specific variation between them.
- A week is scored High-risk if **any** of its 5 questions is answered with the listed High-Risk-Answer (OR logic, not a threshold count) — `wk1HighRisk`, lines 1206–1208.
- Survey risk is a strict running maximum across weeks 1–4: once a week scores High, the resident stays High even if later weeks come back clean — `surveyRisk`, lines 1158–1173.
- Displayed risk is the higher of the physician-recorded risk and the survey-derived risk — `effectiveRisk`, lines 1174–1189.
