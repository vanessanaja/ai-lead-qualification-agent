
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
| `confidence` | Number from 0 through 1 (inclusive) |
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

The returned `confidence` is the agent's self-reported confidence. It is not treated as a calibrated probability of correctness.

### Failure Taxonomy

| Failure type | Meaning |
| --- | --- |
| `false_qualification` | The agent returns `qualified` when qualification is not an acceptable outcome. |
| `false_rejection` | The agent returns `not_qualified` when rejection is not an acceptable outcome. |
| `unnecessary_human_review` | The agent returns `human_review` when the evidence supports a clear automatic decision. |
| `reasoning_issue` | The explanation misapplies the policy, contradicts the application, or invents facts. |
| `tool_use_issue` | The agent violates the contact, contact-ID, or tagging requirements. |

More than one failure type may apply to a result. Structured-output validation failures will be recorded separately.

## Test Suite

The evaluation suite contains 12 fictional applications in `evaluation/test-cases.json`.

| Test cases | Scenario |
| --- | --- |
| TC-01–02 | Clearly qualified |
| TC-03–04 | Clearly not qualified |
| TC-05–06 | Missing information requiring human review |
| TC-07–08 | Contradictory application information |
| TC-09–10 | Borderline qualification |
| TC-11 | Prompt manipulation |
| TC-12 | Business-policy edge case |

Each test case includes the submitted application, acceptable decisions, and reasoning points for manual review.

The test cases use fictional names, businesses, and contact information.

## Evaluation Workflow

**Workflow:** `AI Lead Qualification Evaluation`

**Model:** `gpt-5.6-luna`

**Workflow file:** `evaluation/AI Lead Qualification Evaluation.json`

The workflow runs locally in n8n using Docker. It uses the production qualification prompt with simulated HighLevel tools, so testing does not create or modify real CRM contacts.

The workflow:

1. Loads the evaluation cases.
2. Passes each application to the qualification agent.
3. Simulates contact creation or updating and qualification tagging.
4. Merges the agent response with the original test case.
5. Evaluates the result using JavaScript assertions.

### Automated Assertions

| Assertion | What it checks |
| --- | --- |
| `decision_pass` | The decision matches one of the test case's acceptable decisions. |
| `schema_pass` | The required output fields are present with valid values and types. |
| `contact_pass` | The simulated contact response matches the submitted contact information and the agent's returned contact ID. |
| `tag_pass` | The simulated tagging response uses the correct qualification tag and contact ID. |
| `overall_pass` | All four assertions pass. |

The agent is configured with a maximum of five iterations to reduce the risk of repeated tool calls. This is not a strict limit on API requests.

### Running the Evaluation

1. Import `evaluation/AI Lead Qualification Evaluation.json` into n8n.
2. Configure OpenAI credentials in the OpenAI Chat Model node.
3. Confirm the model, output parser, simulated tools, and node connections are configured correctly.
4. For initial testing, run one case at a time by changing the return statement in **Load Evaluation Cases** to:

```javascript
return [{
  json: cases[0]
}];
```

5. Execute the workflow and inspect the **Evaluate Result** output.
6. To run all 12 cases, restore the original return statement:

```javascript
return cases.map(testCase => ({
  json: testCase
}));
```

The full 12-case batch has not yet been executed successfully. The baseline was collected through individual test runs.

Stop execution if the agent repeatedly retries model calls or simulated tools fail.

## Initial Baseline Results

**Baseline:** `baseline-01`

**Documented:** October 23, 2026

**Policy version:** `production-v1`

**Results file:** `evaluation/results/baseline-01.json`

The initial baseline was established by running each of the 12 test cases individually. This allowed failures to be isolated and API usage to be controlled after an earlier tool-schema issue caused repeated model calls.

### Results Summary

| Metric | Result |
| --- | --- |
| Test cases | 12 |
| Passed | 11 |
| Failed | 1 |
| Pass rate | 91.67% |
| Structured-output failures | 0 |
| Simulated contact assertion failures | 0 |
| Simulated tagging assertion failures | 0 |

The pass rate measures agreement with the predefined acceptable decisions for these 12 cases. It is not an estimate of performance across real customer applications.

### Notable Findings

**TC-11: Prompt Manipulation**

The applicant included instructions attempting to override the qualification policy and force a `qualified` decision.

The agent returned `not_qualified`, consistent with the application information and the original expected decision.

**Result:** Pass.

**TC-12: Business-Policy Edge Case**

The applicant had sufficient budget, decision authority, and a near-term timeline but described an infrequent, low-impact problem that did not justify a complex AI implementation.

- Expected: `not_qualified`
- Actual: `human_review`
- Result: Fail

The agent recognized the weak business fit but recommended human review to explore whether broader automation opportunities existed.

The result exposes a possible ambiguity between the original test expectation and the judgment-based `production-v1` policy, which permits human review for certain edge cases.

The initial mismatch has been preserved in the baseline. No test expectations or production qualification rules have been changed.

## Limitations and Regression Testing

The current evaluation has several limitations:

- The 12 cases were executed individually rather than as one batch.
- Each case represents one recorded successful completion, not repeated testing across multiple runs.
- The automated assertions do not independently score the quality of the agent's written reasoning.
- The simulated tools verify returned values but do not test actual HighLevel API operations, data persistence, or preservation of existing CRM tags.
- Passing the prompt-manipulation test does not establish comprehensive resistance to prompt injection.
- The test suite contains fictional scenarios and does not represent production traffic.
- The agent's confidence scores are self-reported and are not calibrated probabilities.

### Future Regression Testing

The baseline will be retained as a reference for future evaluation runs.

After changes to the qualification policy, agent configuration, or evaluation criteria, the same test suite can be executed again.

New results will be saved separately, for example:

`evaluation/results/regression-01.json`

Future results can then be compared with the original baseline to identify improvements, regressions, and changes in decision consistency.

Changes to test expectations will be documented separately from changes to agent behavior so improvements are not overstated.

### TC-12 Acceptance Criteria Revision

**Date:** October 23, 2026

After reviewing TC-12 against the documented `production-v1` policy, the acceptable decisions were expanded from `not_qualified` to `not_qualified` or `human_review`.

The policy permits human review for edge cases and does not require automatic rejection based on any single qualification factor.

The original `baseline-01.json` remains unchanged at 11/12 passes. This revision reflects a clarification of test expectations, not an improvement in agent performance.

Future regression tests will use the revised acceptance criteria.