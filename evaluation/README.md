# AI Lead Qualification Evaluation

## Qualification Policy

**Policy version:** `production-v1`

This policy documents the qualification logic in the current production agent prompt. Each evaluation run records this version so its results can be tied to the exact policy tested.

The fictional offer is the **AI Workflow Implementation Package**, starting at **$3,000**.

### Qualification Criteria

The agent evaluates the following factors together:

| Criterion | Current policy |
| --- | --- |
| Business status | The business should be actively operating or clearly preparing to launch. |
| Operational problem | The lead should describe a real business process or operational problem that could plausibly benefit from automation. |
| Frequency and impact | A recurring problem with meaningful business impact generally indicates a stronger fit. |
| Budget | Budget should be considered seriously because the offer starts at $3,000. |
| Decision authority | The lead should have meaningful involvement in approving the investment. |
| Timeline | A timeline within 90 days generally indicates a stronger fit. |

These are judgment-based criteria. The production prompt does not use a numerical score or make any single factor an automatic rejection.

For example, the prompt does not require automatic rejection based only on:

- A budget below $3,000
- A timeline longer than 90 days
- The applicant sharing decision-making authority
- A particular business category
- A particular problem frequency

### Expected Decisions

| Decision | Expected use |
| --- | --- |
| `qualified` | The application provides clear evidence of a strong overall fit. |
| `not_qualified` | The application provides clear evidence that the lead is not a good fit. |
| `human_review` | The application is ambiguous, contradictory, incomplete, or presents an edge case that should not be accepted or rejected automatically. |

The agent must evaluate only the information provided in the application and must not invent missing facts.

An empty optional `additional_notes` field does not, by itself, make an application incomplete.

### Deterministic Requirements

The structured response must contain exactly these fields:

| Field | Requirement |
| --- | --- |
| `decision` | `qualified`, `not_qualified`, or `human_review` |
| `reason` | String |
| `confidence` | Number from 0 through 1 (inclusive)|
| `recommended_next_action` | String |
| `highlevel_contact_id` | Exact contact ID returned by the contact tool |

The agent must also: 

1. Send the submitted name, email, and business name to the contact tool without altering or inventing values.
2. Use the contact ID returned by that tool for the tagging request.
3. Apply exactly one qualification-status tag matching the decision.
4. Preserve existing contact tags.
5. Return the same contact ID in `highlevel_contact_id`.

| Decision | Required tag |
| --- | --- |
| `qualified` | `qualified` |
| `not_qualified` | `not-qualified` |
| `human_review` | `human-review` |

### Judgment-Based Evaluation

The returned reason will be reviewed for:

- Consistency with the decision
- Appropriate use of the qualification criteria
- Support from the submitted application
- Recognition of relevant ambiguity or contradictions
- Unsupported or invented claims

The wording does not need to match an expected answer exactly.

The returned `confidence` is the agent’s self-reported confidence. It is not treated as a calibrated probability of correctness.

### Failure Taxonomy

| Failure type | Meaning |
| --- | --- |
| `false_qualification` | The agent returns `qualified` when qualification is not an acceptable outcome. |
| `false_rejection` | The agent returns `not_qualified` when rejection is not an acceptable outcome. |
| `unnecessary_human_review` | The agent returns `human_review` when the evidence supports a clear automatic decision. |
| `reasoning_issue` | The explanation misapplies the policy, contradicts the application, or invents facts. |
| `tool_use_issue` | The agent violates the contact, contact-ID, or tagging requirements. |

More than one failure type may apply to a result. Structured-output validation failures will be recorded separately.