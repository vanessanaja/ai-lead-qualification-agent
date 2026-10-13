# AI Lead Qualification Agent

An n8n-based AI agent that evaluates discovery-call applications, classifies leads as qualified, not qualified, or requiring human review, and routes each outcome through controlled workflow actions.

The project is designed around bounded AI autonomy: the agent makes judgment-based qualification decisions, while n8n handles predictable workflow logic and approved downstream actions.

## Current Workflow

The workflow currently:

1. Accepts a discovery application through an n8n form.
2. Collects business information including the lead's problem, business impact, budget, decision authority, and implementation timeline.
3. Normalizes the submitted data into consistent internal field names.
4. Captures the form submission timestamp.
5. Appends the normalized lead data to Google Sheets using automatic column mapping.

## Planned Workflow

Version 1 is designed to:

1. Accept and normalize a discovery application.
2. Pass the lead data to an AI Agent.
3. Evaluate the application against defined qualification criteria.
4. Choose one of three outcomes:
   - `qualified`
   - `not_qualified`
   - `human_review`
5. Return structured decision data:
   - `decision`
   - `reason`
   - `confidence`
   - `recommended_next_action`
6. Route the selected outcome through controlled n8n actions.
7. Store the lead information and AI decision in Google Sheets.

## Planned Workflow Flow

```text
n8n Form Trigger
→ Edit Fields / Data Normalization
→ AI Lead Qualification Agent
→ Controlled Decision Routing
→ Google Sheets Logging
→ Approved Follow-up Actions
```

## Tech Stack

- n8n
- OpenAI API / LLM
- n8n AI Agent
- Google Sheets API
- Google Drive API
- Google OAuth
- Docker
- Git / GitHub

## Agent Design

The agent will handle tasks that require contextual judgment, including:

- Assessing whether the lead has a meaningful automation opportunity
- Evaluating the business problem and impact
- Considering budget, decision authority, urgency, and business stage together
- Identifying ambiguous or contradictory applications
- Choosing human review when the available information does not support a confident automated decision

n8n will handle deterministic tasks such as:

- Form collection
- Data normalization
- Data storage
- Controlled routing
- Logging
- Error handling
- Approved downstream actions

The agent will only be allowed to choose from predefined outcomes and controlled actions rather than receiving unrestricted access to business systems.

## Qualification Outcomes

### Qualified

The lead appears to be a strong fit for the AI Workflow Implementation Package and can proceed to the next stage.

### Not Qualified

The application clearly falls outside the current offer criteria.

### Human Review

The application is ambiguous, contradictory, or represents an edge case that should be reviewed by a person before a decision is made.

## Google Sheets Output

The workflow uses a `Leads` worksheet with the following columns:

- `timestamp`
- `name`
- `email`
- `business_name`
- `business_type`
- `operating_status`
- `problem`
- `frequency`
- `business_impact`
- `desired_outcome`
- `budget`
- `decision_authority`
- `timeline`
- `additional_notes`
- `decision`
- `reason`
- `confidence`
- `recommended_next_action`

## Security

- OAuth credentials are stored outside the repository.
- Credential references and instance-specific metadata are removed from public workflow exports.
- The real Google Sheet ID is replaced with a placeholder before publication.
- Workflow exports are reviewed and sanitized before being committed.

## Version 1 Status

### Working

- n8n discovery application form
- Required form fields
- Business and qualification data collection
- Form submission timestamp
- Edit Fields data normalization
- Google Sheets OAuth integration
- Google Sheets append-row integration
- Automatic input-to-column mapping
- Successful form-to-Google-Sheets test
- Persistent local n8n environment using Docker
- Sanitized GitHub workflow export
- Git / GitHub version control

### Still to Add

- AI Agent node
- Qualification criteria and agent instructions
- Structured decision output
- `qualified` outcome
- `not_qualified` outcome
- `human_review` outcome
- Confidence score
- Decision reasoning
- Recommended next action
- Controlled agent tools / actions
- Final Google Sheets logging of agent decisions
- Invalid AI-output handling
- Qualified, unqualified, ambiguous, and contradictory test cases
- End-to-end workflow testing
- Workflow screenshot
- Final documentation
- Setup / reproduction instructions