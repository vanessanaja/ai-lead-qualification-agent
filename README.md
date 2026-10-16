# AI Lead Qualification Agent

An agentic lead-qualification workflow built with n8n, OpenAI, GoHighLevel, and Google Sheets.

The system evaluates inbound leads for a fictional AI workflow implementation service, makes a qualification decision, takes bounded CRM actions, creates sales opportunities for active leads, and preserves human judgment for ambiguous cases.

This project was built to demonstrate practical AI implementation skills beyond basic prompt-and-response automation, including agent tool use, API integration, deterministic routing, structured output, CRM state management, human-in-the-loop workflows, and end-to-end testing.

---

## What the System Does

A lead submits an application containing information about their business, operational problem, budget, authority, timeline, and desired outcome.

The system then:

1. Normalizes the submitted lead data.
2. Sends the application to an AI agent for evaluation.
3. Classifies the lead as:
   - `qualified`
   - `human_review`
   - `not_qualified`
4. Uses AI-agent tools to:
   - Create or update the contact in GoHighLevel.
   - Add the appropriate qualification-status tag.
5. Returns the exact GoHighLevel contact ID as part of the structured AI output.
6. Merges the AI decision back with the original lead data.
7. Logs the complete lead record and decision to Google Sheets.
8. Uses deterministic n8n routing based on the qualification result.
9. Creates a GoHighLevel opportunity for `qualified` and `human_review` leads.
10. Uses native GoHighLevel workflows to handle opportunity routing and subsequent human decisions.

The AI is responsible for judgment and bounded CRM tool use. Deterministic workflow logic handles logging, routing, opportunity creation, and downstream CRM state changes.

---

## Architecture

```text
Lead Application Form
        |
        v
    Edit Fields
        |
        +-----------------------------+
        |                             |
        v                             |
Lead Qualification Agent             |
        |                             |
        |  OpenAI Chat Model          |
        |  Structured Output Parser   |
        |  Upsert HighLevel Contact   |
        |  Add Qualification Tag      |
        |                             |
        +-------------+---------------+
                      |
                      v
          Merge Lead Data + AI Decision
                      |
                      v
             Prepare Final Lead Record
                      |
                      v
             Log Lead to Google Sheets
                      |
                      v
          Route by Qualification Decision
               /          |          \
              /           |           \
       qualified     human_review   not_qualified
            \             /              |
             \           /               v
              v         v        Not Qualified Action
          Create Active
         Lead Opportunity
              |
              v
        GoHighLevel CRM
```

GoHighLevel then handles the CRM lifecycle through native workflows.

---

## AI Agent Responsibilities

The `Lead Qualification Agent` performs the parts of the system that require judgment.

It evaluates factors including:

- Whether the business is operating or preparing to launch.
- Whether the lead describes a real operational problem.
- Whether that problem could plausibly benefit from automation.
- How frequently the problem occurs.
- Its business impact.
- Budget relative to the fictional $3,000 starting price.
- Decision-making authority.
- Implementation timeline.
- Ambiguous, incomplete, or contradictory information.

The agent is instructed not to invent information that is not present in the application.

---

## Structured Output

The agent must return:

```json
{
  "decision": "qualified | not_qualified | human_review",
  "reason": "string",
  "confidence": 0,
  "recommended_next_action": "string",
  "highlevel_contact_id": "string"
}
```

A Structured Output Parser enforces the schema before downstream processing.

---

## Agentic CRM Tool Use

This project intentionally goes beyond using an LLM only as a classifier.

The AI agent has access to bounded GoHighLevel tools and can take actions based on its decision.

### Tool 1: Upsert HighLevel Contact

The agent creates or updates the lead in GoHighLevel using the submitted:

- Name
- Email
- Business name

The contact source is recorded as:

```text
AI Lead Qualification Agent
```

Using an upsert operation prevents the workflow from blindly creating duplicate contacts when the same email is submitted again.

### Tool 2: Add HighLevel Qualification Tag

After the contact upsert succeeds, the agent uses the returned contact ID to add exactly one status tag:

```text
qualified
human-review
not-qualified
```

The tool adds the qualification tag without removing unrelated existing tags.

This behavior was explicitly tested by placing an unrelated tag on an existing contact, resubmitting the lead, and confirming that both the existing tag and the new qualification tag remained.

### Returning the CRM Contact ID

The agent must return the exact contact ID produced by the contact-upsert tool.

That ID is then passed through the rest of the n8n workflow and used when creating the GoHighLevel opportunity.

---

## Separation of AI Judgment and Deterministic Logic

A key design decision in this project is that the AI does not control every part of the system.

The agent handles:

- Lead evaluation.
- Qualification reasoning.
- Contact creation/update.
- Qualification tagging.

n8n handles deterministic operations such as:

- Data merging.
- Flattening structured output.
- Google Sheets logging.
- Qualification routing.
- Opportunity creation.

GoHighLevel native workflows handle:

- Opportunity stage movement.
- Human-review notifications.
- Manual qualification outcomes.
- Lost opportunity status.

This keeps probabilistic AI reasoning limited to tasks where judgment is actually useful while using predictable workflow logic for operations that should behave consistently.

---

## Qualification Outcomes

### Qualified

A qualified lead receives the:

```text
qualified
```

tag.

n8n creates an open opportunity in the dedicated:

```text
AI Lead Qualification Agent
```

pipeline.

The opportunity initially enters:

```text
New Lead
```

A GoHighLevel workflow detects that the newly created opportunity belongs to a contact with the `qualified` tag and moves it to:

```text
Qualified
```

### Human Review

An ambiguous lead receives:

```text
human-review
```

The lead is still considered active, so n8n creates an opportunity in:

```text
New Lead
```

A GoHighLevel workflow detects the `human-review` tag and reaches an internal notification action so a person can review the lead.

The opportunity remains active while the human decision is pending.

### Not Qualified

A lead confidently classified as not qualified receives:

```text
not-qualified
```

No active sales opportunity is created automatically for this outcome.

This prevents clearly disqualified leads from cluttering the active sales pipeline.

---

## Human-in-the-Loop Decision Handling

Human review is treated as a real workflow state rather than a dead-end classification.

If a lead initially receives:

```text
human-review
```

a person can later make the final decision directly in GoHighLevel.

### Human Approves the Lead

The reviewer manually adds:

```text
qualified
```

A separate GoHighLevel workflow then:

```text
Qualified Tag Added
        |
        v
Was Human Review?
        |
       Yes
        |
        v
Find Lead Opportunity
        |
        v
Opportunity Found
        |
        v
Move Reviewed Lead to Qualified
```

The workflow verifies that the contact also has the `human-review` tag so that normal AI-qualified leads do not accidentally enter the manual-review workflow.

The existing opportunity is located in the `AI Lead Qualification Agent` pipeline and moved to:

```text
Qualified
```

This path was tested successfully.

### Human Rejects the Lead

The reviewer can instead add:

```text
not-qualified
```

A separate workflow performs the same human-review verification and opportunity lookup, then updates the opportunity status to:

```text
Lost
```

The opportunity record and history are preserved without leaving the lead in the active pipeline.

This path was also tested successfully.

---

## GoHighLevel Pipeline

The project uses a dedicated pipeline:

```text
AI Lead Qualification Agent
```

Stages:

```text
New Lead
Qualified
Discovery scheduled
Proposal sent
```

The qualification decision and the sales pipeline stage are intentionally treated as separate concepts.

For example:

- `human_review` is a qualification state.
- `New Lead` is a sales pipeline stage.
- `qualified` is a qualification result.
- `Qualified` is the corresponding pipeline stage after the lead is accepted.

This separation makes the system easier to reason about and prevents AI classification state from being conflated with sales progression.

---

## GoHighLevel Native Workflows

The CRM side uses three native GoHighLevel workflows.

### 1. AI Lead Qualification - Opportunity Routing

Trigger:

```text
Opportunity Created
```

Filtered to:

```text
AI Lead Qualification Agent
```

pipeline.

The workflow checks the contact's qualification tag.

If:

```text
qualified
```

the opportunity is moved to:

```text
Qualified
```

If:

```text
human-review
```

the opportunity remains in:

```text
New Lead
```

and an internal notification action is triggered for human review.

---

### 2. AI Lead Qualification - Manual Review Qualified

Trigger:

```text
qualified tag added
```

The workflow then verifies:

```text
Tags includes human-review
```

If true:

1. Find the most recently created opportunity for the contact in the `AI Lead Qualification Agent` pipeline.
2. If an opportunity is found, move it to the `Qualified` stage.

This prevents normal AI-qualified leads from entering the manual-review path.

---

### 3. AI Lead Qualification - Manual Review Not Qualified

Trigger:

```text
not-qualified tag added
```

The workflow verifies:

```text
Tags includes human-review
```

If true:

1. Find the most recently created opportunity for the contact in the `AI Lead Qualification Agent` pipeline.
2. If an opportunity is found, update its status to:

```text
Lost
```

This removes the opportunity from the active sales pipeline while preserving its history.

---

## Google Sheets Audit Log

Every evaluated lead is logged to Google Sheets.

The audit log captures both submitted data and AI decision data, including:

- Timestamp
- Name
- Email
- Business name
- Business type
- Operating status
- Problem
- Frequency
- Business impact
- Desired outcome
- Budget
- Decision authority
- Timeline
- Additional notes
- Decision
- Reason
- Confidence
- Recommended next action
- HighLevel contact ID

This provides a simple audit trail for reviewing how the agent evaluated leads and what downstream CRM record was used.

---

## Example Qualification Logic

The fictional offer used for the project is:

```text
AI Workflow Implementation Package
Starting at $3,000
```

Stronger-fit leads generally have:

- An active business or a business clearly preparing to launch.
- A real repetitive operational problem.
- Meaningful business impact.
- A process that occurs regularly.
- Budget consistent with the offer.
- Meaningful decision authority.
- A timeline within approximately 90 days.

The agent uses:

```text
human_review
```

when information is ambiguous, contradictory, incomplete, or presents an edge case that should not be automatically accepted or rejected.

---

## Why Use an Agent Here?

A simple classifier could return:

```text
qualified
not_qualified
human_review
```

but that alone would not make the system agentic.

In this project, the agent can:

1. Evaluate the lead.
2. Decide the qualification outcome.
3. Select and call the appropriate CRM tools.
4. Create or update the GoHighLevel contact.
5. Capture the real CRM contact ID returned by the tool.
6. Use that returned ID in a subsequent tagging tool call.
7. Return the result in a strict structured format for deterministic downstream processing.

The agent therefore combines reasoning with bounded external actions while the surrounding automation maintains control over predictable business logic.

---

## Key Design Decisions

### AI Does Not Create Opportunities Directly

Opportunity creation is deterministic once the qualification outcome is known.

Rather than giving the agent another tool for this predictable operation, n8n creates the opportunity after the qualification branch.

This reduces unnecessary AI autonomy and makes the workflow easier to debug.

### Human Review Is an Active CRM State

`human_review` does not simply stop the automation.

The system creates an opportunity, surfaces the lead for human judgment, and supports both later approval and rejection.

### Existing Tags Are Preserved

Qualification tagging is additive.

The agent is explicitly instructed not to remove or replace unrelated CRM tags.

### Not-Qualified Leads Do Not Create Active Opportunities

Clearly rejected leads are logged and tagged but do not enter the active sales pipeline.

### AI Output Is Flattened Before Downstream Processing

The structured AI response is converted into top-level fields before logging and routing.

This keeps downstream nodes simple and makes values such as:

```text
decision
confidence
highlevel_contact_id
```

directly available to later nodes.

---

## Testing Completed

The following behaviors were tested successfully:

- Form submission.
- Lead data normalization.
- AI qualification.
- `qualified` classification.
- `human_review` classification.
- `not_qualified` classification.
- Structured output parsing.
- GoHighLevel contact upsert.
- Existing-contact updates.
- Preservation of unrelated existing tags.
- `qualified` tag application.
- `human-review` tag application.
- `not-qualified` tag application.
- Returning and preserving the exact GoHighLevel contact ID.
- Google Sheets logging.
- Qualification routing in n8n.
- GoHighLevel opportunity creation.
- Automatic placement of new opportunities in `New Lead`.
- Automatic movement of AI-qualified opportunities to `Qualified`.
- Human-review CRM routing.
- Human-review internal notification action.
- Manual `human-review` to `qualified` transition.
- Finding the correct existing opportunity from a contact-tag workflow.
- Moving a manually approved lead to `Qualified`.
- Manual `human-review` to `not-qualified` transition.
- Marking a manually rejected opportunity as `Lost`.
- Prevention of duplicate active opportunity creation for the same contact during testing.
- Individual downstream-node testing without unnecessarily rerunning the AI agent.

---

## Technology

- n8n
- OpenAI API
- GPT-5.6 Luna
- GoHighLevel
- GoHighLevel REST API
- Google Sheets
- HTTP / REST
- JSON
- JSON Schema
- Docker
- Git
- GitHub

---

## n8n Workflow Nodes

The main workflow includes:

```text
AI Workflow Discovery Application
Edit Fields
Lead Qualification Agent
OpenAI Chat Model
Upsert HighLevel Contact
Add HighLevel Qualification Tag
Structured Output Parser
Merge Lead Data + AI Decision
Prepare Final Lead Record
Log Lead to Google Sheets
Route by Qualification Decision
Create Active Lead Opportunity
Not Qualified Action
```

The agent's tool nodes are connected through the n8n AI Agent tool interface rather than as ordinary sequential workflow nodes.

---

## Security

The public workflow export is sanitized before being committed to GitHub.

Secrets and account-specific identifiers are not intended to be included in the repository.

Examples of sanitized values include:

- API credentials
- Bearer tokens
- Google credentials
- Google Sheet IDs
- GoHighLevel location IDs
- GoHighLevel pipeline IDs
- Credential references
- Instance-specific metadata
- Test contact data

The repository uses placeholders such as:

```text
YOUR_GHL_LOCATION_ID
YOUR_GHL_PIPELINE_ID
YOUR_GOOGLE_SHEET_ID
```

where configuration is required.

Credentials should be configured inside n8n rather than embedded directly in workflow JSON.

---

## Running Locally

This project was developed using n8n running locally in Docker with a persistent volume.

Example:

```bash
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

Then open:

```text
http://localhost:5678
```

---

## Setup Overview

To reproduce the project, you need:

- Docker
- n8n
- An OpenAI API account
- A GoHighLevel sub-account or test account
- A GoHighLevel Private Integration with the required scopes
- Google Sheets access
- Google OAuth configured in n8n

After importing the sanitized workflow:

1. Configure the OpenAI credential.
2. Configure Google Sheets OAuth.
3. Configure the GoHighLevel Bearer credential.
4. Replace placeholder GoHighLevel IDs.
5. Replace the placeholder Google Sheet ID.
6. Recreate or connect the required GoHighLevel pipeline and native workflows.
7. Run test leads through the workflow.

---

## Future Enhancements

Potential next steps include:

- Sending a discovery-call scheduling email after qualification.
- Automatically moving an opportunity to `Discovery scheduled` after a booking occurs.
- Moving an opportunity to `Proposal sent` after a proposal is created or sent.
- Adding dedicated error-handling branches.
- Adding retries for external API failures.
- Adding explicit invalid-AI-output handling.
- Expanding audit logging.
- Adding Slack or email alerts for human-review leads.
- Adding reporting on qualification rates and pipeline conversion.

These are intentionally separate from the core workflow so the current project remains focused on qualification, CRM action, deterministic routing, and human-in-the-loop decision handling.

---

## Project Goal

The goal of this project is to demonstrate how an AI system can be integrated into a real business workflow without giving the model unnecessary control.

The project combines:

- AI reasoning.
- Structured LLM output.
- Agent tool use.
- CRM API integration.
- Deterministic automation.
- Human review.
- Audit logging.
- Pipeline management.
- Secure credential handling.
- Testing and troubleshooting.

The result is not just an AI classifier.

It is an interconnected lead-processing system in which the AI makes bounded judgments and takes bounded actions while n8n and GoHighLevel handle the predictable operational workflow around those decisions.