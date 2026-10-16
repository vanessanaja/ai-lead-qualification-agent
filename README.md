# AI Lead Qualification Agent

An agentic lead qualification and CRM automation system built in n8n and HighLevel that evaluates inbound discovery applications, makes a bounded qualification decision, takes controlled CRM actions, logs the complete decision record, routes the lead through downstream logic, manages sales-pipeline progression, and supports automated discovery-call follow-up.

The project is designed around a fictional **AI Workflow Implementation Package starting at $3,000** for service-based businesses.

The goal is not simply to classify leads with AI. The system demonstrates how an AI agent can operate safely inside a larger business workflow that spans intake, reasoning, structured output, CRM updates, pipeline progression, human review, automated follow-up, scheduling, and operational logging.

## What This System Does

A lead submits a discovery application through n8n.

The Lead Qualification Agent evaluates the application using contextual business criteria including:

- Business stage
- Operational problem
- Automation suitability
- Frequency of the problem
- Business impact
- Desired outcome
- Budget
- Decision-making authority
- Implementation timeline
- Ambiguity or contradictory information

The agent returns one of three structured outcomes:

- `qualified`
- `not_qualified`
- `human_review`

It also returns:

- A written reason for the decision
- A confidence score
- A recommended next action

The agent then uses tightly scoped HighLevel API tools to:

1. Create or update the lead as a HighLevel contact
2. Reuse the returned HighLevel contact ID
3. Add the qualification-status tag that matches the agent's decision
4. Preserve any unrelated tags already associated with the contact

The workflow then:

5. Merges the original lead data with the AI-generated decision
6. Logs the complete lead record and qualification result to Google Sheets
7. Routes the lead through deterministic n8n workflow branches
8. Creates or updates the corresponding HighLevel opportunity
9. Moves the opportunity into the appropriate sales pipeline stage
10. Supports human review when the AI should not make an automatic decision
11. Uses native HighLevel workflows for discovery-call follow-up and scheduling

This project demonstrates practical experience with:

- AI agents
- Tool calling
- Structured LLM output
- JSON Schema
- REST APIs
- Bearer-token authentication
- CRM integration
- HighLevel Private Integrations
- Contact management
- Tag management
- Opportunity management
- Sales pipeline automation
- Native CRM workflows
- Human-in-the-loop design
- Conditional routing
- Google Sheets logging
- Google OAuth
- Secure credential handling
- API debugging
- Workflow testing
- Docker
- Git
- GitHub
- VS Code

## System Architecture

```text
n8n Discovery Form
        ↓
Data Normalization
        ↓
Lead Qualification Agent
   ├── OpenAI Chat Model
   ├── Structured Output Parser
   ├── HighLevel Contact Upsert Tool
   └── HighLevel Qualification Tag Tool
        ↓
Merge Original Lead Data + AI Decision
        ↓
Prepare Final Lead Record
        ↓
Google Sheets Logging
        ↓
Decision Routing
   ├── qualified
   ├── not_qualified
   └── human_review
        ↓
HighLevel CRM Actions
   ├── Contact Management
   ├── Qualification Tagging
   ├── Opportunity Creation / Update
   └── Pipeline Stage Management
        ↓
HighLevel Native Workflows
   ├── Discovery-call scheduling email
   ├── Manual qualification follow-up
   └── Discovery Scheduled progression
```

The system deliberately separates:

**AI judgment**

from

**deterministic operational control**

The AI handles contextual reasoning and tightly scoped tool use.

n8n handles normalization, merging, logging, branching, and controlled execution.

HighLevel handles CRM state, opportunity progression, pipeline management, and native follow-up automation.

Google Sheets provides an external audit trail of submitted leads and AI decisions.

## Current Workflow

The current workflow:

1. Accepts an application through an n8n discovery form
2. Collects business, operational, budget, authority, and timeline information
3. Normalizes the form submission into consistent internal field names
4. Sends the application to the Lead Qualification Agent
5. Evaluates the application using business-fit criteria and contextual judgment
6. Returns a structured qualification decision
7. Creates or updates the lead as a contact in HighLevel
8. Uses the returned HighLevel contact ID to add the matching qualification tag
9. Preserves existing HighLevel tags
10. Merges the original lead data with the AI-generated decision
11. Logs the complete lead record and AI decision to Google Sheets
12. Routes the lead into the appropriate deterministic workflow branch

HighLevel opportunity creation, pipeline-stage movement, and discovery-call follow-up are being added as the next layer of the system.

## Qualification Logic

The agent evaluates factors including:

- Whether the business is actively operating or clearly preparing to launch
- Whether the lead describes a real business process or operational problem
- Whether the process could plausibly benefit from automation
- How frequently the problem occurs
- The current business impact
- The desired outcome
- Budget relative to the $3,000 starting investment
- The lead's involvement in approving the investment
- Implementation timeline
- Ambiguity, contradictions, or unusual circumstances

The agent is instructed not to invent information that is not present in the application.

## Qualification Outcomes

### `qualified`

Used when the application represents a strong fit for the offer.

The HighLevel contact receives:

```text
qualified
```

A qualified lead will also enter the HighLevel sales process and trigger the discovery-call follow-up workflow.

### `not_qualified`

Used when the application clearly does not meet the current offer criteria.

The HighLevel contact receives:

```text
not-qualified
```

A not-qualified lead does not progress through the active sales pipeline.

### `human_review`

Used when the application is ambiguous, contradictory, incomplete, or presents an edge case that should not be automatically accepted or rejected.

The HighLevel contact receives:

```text
human-review
```

This creates an explicit human-in-the-loop path rather than forcing the model to make a decision when the available information is insufficient.

A human can later review the lead and manually qualify it if appropriate.

## Structured AI Output

The Lead Qualification Agent returns a strict structured object containing:

```json
{
  "decision": "qualified | not_qualified | human_review",
  "reason": "Explanation of the decision",
  "confidence": 0,
  "recommended_next_action": "Recommended next step"
}
```

The output is constrained with a JSON Schema.

The `decision` field can contain only:

```text
qualified
not_qualified
human_review
```

Confidence is constrained to an integer from 0 to 100.

Structured output makes the AI decision predictable and safe to use in downstream automation.

## Agent Tool Use

The Lead Qualification Agent currently has two tightly scoped HighLevel API tools.

### HighLevel Contact Upsert

The agent can create a new contact or update an existing HighLevel contact using:

- Name
- Email
- Business name
- Fixed source attribution

The agent is instructed to use submitted lead data exactly as provided rather than inventing contact information.

The HighLevel API returns a contact ID that is then used by the second tool.

### HighLevel Qualification Tagging

After receiving the HighLevel contact ID, the agent calls a separate HighLevel Add Tags endpoint.

The tool adds exactly one qualification-status tag:

```text
qualified
human-review
not-qualified
```

The tagging operation intentionally uses HighLevel's dedicated tag endpoint rather than including tags in the contact update.

This preserves unrelated tags already associated with the contact.

Testing confirmed that an existing unrelated tag remained on the contact after the agent added the qualification tag.

## Bounded Agent Design

The agent is intentionally not given unrestricted CRM access.

Its autonomy is bounded to specific approved actions:

- Evaluate a submitted lead
- Select one of three qualification outcomes
- Create or update the corresponding HighLevel contact
- Add one qualification-status tag

It cannot freely perform arbitrary actions inside HighLevel.

This creates a safer pattern for business AI implementation by separating contextual reasoning from controlled operational execution.

## HighLevel Sales Pipeline

The CRM side of the system uses a dedicated HighLevel pipeline representing the actual sales process.

The pipeline stages are:

```text
New Lead
Qualified
Discovery Scheduled
Proposal Sent
```

Qualification status and pipeline stage are intentionally treated as separate concepts.

For example:

- `qualified` is an AI qualification outcome and contact tag
- `Qualified` is a sales pipeline stage
- `human_review` is an evaluation state, not a sales stage
- `not_qualified` is a qualification result, not a sales stage

The system is being designed so that:

- A newly submitted lead can exist in `New Lead`
- A qualified lead moves into `Qualified`
- A lead that books a discovery call moves into `Discovery Scheduled`
- A lead that reaches the proposal stage moves into `Proposal Sent`

Opportunity creation and pipeline-stage movement will be managed through HighLevel CRM actions.

## Automated Discovery-Call Follow-Up

Qualified leads will automatically enter a native HighLevel follow-up workflow.

The planned process is:

```text
Lead Qualification Agent
        ↓
Lead receives "qualified" tag
        ↓
HighLevel Workflow Trigger
        ↓
Discovery-call scheduling email is sent
        ↓
Lead books discovery call
        ↓
Opportunity moves to "Discovery Scheduled"
```

The same HighLevel workflow can also be triggered if a lead is manually qualified later by a human.

This is important because the system supports both:

- Automatic AI qualification
- Human review and manual qualification

while preserving the same downstream sales process.

For example, a lead may initially receive:

```text
human-review
```

A human can later review the application and determine that the lead should move forward.

Once that lead is manually marked as qualified, the same HighLevel scheduling workflow can send the discovery-call invitation and continue the normal pipeline progression.

The AI agent does not send the email directly.

HighLevel handles the scheduling email through a native workflow after the appropriate qualification condition is met.

This keeps email communication and sales follow-up inside the CRM while limiting the permissions granted to the AI agent.

## Human-in-the-Loop Design

The workflow does not force every lead into a binary yes/no decision.

The `human_review` outcome is used when the agent encounters:

- Incomplete information
- Conflicting information
- Ambiguous budget or timing
- Unusual business circumstances
- Edge cases
- Situations where automatic rejection would be too aggressive

This provides a human escalation path.

A reviewer can later:

1. Evaluate the lead manually
2. Decide whether the lead should be qualified
3. Update the lead's HighLevel status
4. Trigger the same downstream scheduling and pipeline process used for automatically qualified leads

This allows the AI to assist with decision-making without becoming the sole authority for every case.

## Google Sheets Logging

Every application is logged before deterministic decision routing.

The Google Sheet stores:

```text
timestamp
name
email
business_name
business_type
operating_status
problem
frequency
business_impact
desired_outcome
budget
decision_authority
timeline
additional_notes
decision
reason
confidence
recommended_next_action
```

Logging before routing ensures that every evaluated lead is recorded regardless of qualification outcome.

This also creates an external decision history that can be reviewed independently of the CRM.

## Discovery Form

The n8n form collects:

- Name
- Email
- Business name
- Business category
- Current operating stage
- Repetitive process or operational challenge
- Frequency of the challenge
- Current business impact
- Desired outcome
- Available investment range
- Decision-making authority
- Desired implementation timeline
- Additional context

The form intentionally captures both structured qualification signals and open-text context so the agent can apply judgment rather than relying only on rigid scoring rules.

## Tech Stack

- **n8n** for orchestration and workflow execution
- **OpenAI API / LLM** for lead evaluation and agent reasoning
- **n8n AI Agent** for tool-using agent behavior
- **Structured Output Parser / JSON Schema** for constrained AI responses
- **HighLevel REST API** for CRM actions
- **HighLevel Private Integration** for scoped API authentication
- **HTTP Request Tools** for controlled agent API actions
- **HighLevel Contacts** for lead records
- **HighLevel Tags** for qualification status
- **HighLevel Opportunities** for sales tracking
- **HighLevel Pipelines** for sales-stage progression
- **HighLevel Workflows** for native follow-up and scheduling automation
- **Google Sheets API** for lead and decision logging
- **Google Drive API**
- **Google OAuth**
- **Docker** for the local n8n environment
- **Git**
- **GitHub**
- **VS Code**

## Testing Completed

The workflow has been tested with multiple qualification scenarios.

Completed testing includes:

- Successful discovery-form submission
- Data normalization
- Qualified lead evaluation
- Not-qualified lead evaluation
- Human-review evaluation
- Structured decision output
- Confidence scoring
- Decision reasoning
- Recommended next action
- Three-way Switch routing
- Google Sheets logging before routing
- HighLevel Private Integration authentication
- HighLevel REST API authentication and troubleshooting
- HighLevel contact creation
- Existing-contact upsert
- Retrieval and reuse of the HighLevel contact ID
- Agent-controlled `qualified` tagging
- Preservation of an unrelated existing HighLevel tag
- Successful sequential agent tool use

Additional qualification-tag scenarios and full end-to-end regression testing are still planned.

## Security

The project is designed so credentials and account-specific data are not stored in the public repository.

Security measures include:

- OpenAI credentials are stored in n8n rather than the workflow file
- Google OAuth credentials are stored outside the repository
- The HighLevel Private Integration token is stored as an n8n Bearer Auth credential
- The HighLevel credential is restricted to the HighLevel API domain
- HighLevel permissions are scoped to the API capabilities required by the project
- The agent receives access only to predefined CRM actions
- n8n credential references are removed from public workflow exports
- n8n instance-specific metadata is removed before publication
- The real Google Sheet ID is replaced with:

```text
YOUR_GOOGLE_SHEET_ID
```

- The real HighLevel Location ID is replaced with:

```text
YOUR_GHL_LOCATION_ID
```

- Workflow exports are manually reviewed before being committed
- Pinned or test data is checked for personal information
- API keys and access tokens are never committed to GitHub

## Version 1 Status

### Working

- n8n discovery application form
- Required input fields
- Business and qualification data collection
- Submission timestamp capture
- Data normalization
- OpenAI-powered Lead Qualification Agent
- Qualification criteria and decision logic
- Structured JSON Schema output
- `qualified` outcome
- `not_qualified` outcome
- `human_review` outcome
- Decision reasoning
- Confidence score
- Recommended next action
- Original lead-data preservation
- AI-output flattening
- Google Sheets integration
- Google OAuth integration
- Automatic column mapping
- Lead and AI-decision logging
- Logging before decision routing
- Three-way deterministic Switch routing
- Qualified action path
- Not-qualified action path
- Human-review action path
- HighLevel Private Integration authentication
- HighLevel Bearer-token credential configuration
- Restricted HighLevel API domain
- Agent-controlled HighLevel contact creation
- Agent-controlled existing-contact update
- HighLevel contact-ID reuse between tools
- Agent-controlled qualification tagging
- Existing-tag preservation
- Successful qualified-tag testing
- Persistent local n8n environment using Docker
- Sanitized GitHub workflow exports
- Git and GitHub version control

### Still to Add

- Test `human-review` HighLevel tagging end to end
- Test `not-qualified` HighLevel tagging end to end
- HighLevel opportunity creation
- HighLevel opportunity update logic
- HighLevel pipeline-stage movement
- Native HighLevel workflow triggered by qualification
- Discovery-call scheduling email
- Manual qualification trigger path
- Automatic movement to `Discovery Scheduled` after booking
- Invalid AI-output handling
- Additional ambiguous and contradictory test cases
- Full end-to-end regression testing
- Final workflow cleanup and naming
- Portfolio workflow screenshot
- Final sanitized workflow export
- Final documentation review
- Setup and reproduction instructions

## Design Decisions

### AI for Judgment, Deterministic Logic for Control

The model handles decisions that require contextual interpretation.

n8n handles predictable operations such as:

- Data normalization
- Logging
- Merging
- Routing
- Controlled downstream execution

### Structured Output Instead of Free-Form Responses

The agent returns a strict schema so downstream nodes do not need to interpret natural-language responses.

### Human Review as a First-Class Outcome

Ambiguous applications are not forced into a binary qualified/unqualified decision.

The workflow explicitly supports escalation to human review.

### Restricted CRM Tool Access

The agent receives narrow tools rather than broad access to HighLevel.

This demonstrates a bounded-autonomy approach to business AI implementation.

### Separate Tag Operation

Qualification tags are added with a dedicated HighLevel API action rather than through the contact-upsert request.

This preserves pre-existing contact tags and reduces unintended CRM data changes.

### Qualification Status and Pipeline Stage Are Separate

The system intentionally distinguishes between:

- The AI's qualification decision
- The lead's current sales stage

This prevents states such as `human_review` and `not_qualified` from being incorrectly modeled as stages in the active sales pipeline.

### CRM-Native Email Automation

The agent does not directly send discovery-call emails.

Instead, qualification triggers a native HighLevel workflow that handles the scheduling communication.

This keeps communication logic inside the CRM and limits the permissions granted to the AI agent.

### AI and Human Qualification Share the Same Downstream Process

A lead can become qualified either:

- Automatically through the AI agent
- Manually after human review

Both paths can enter the same HighLevel workflow for scheduling and pipeline progression.

This avoids duplicating follow-up logic and creates a consistent operational process.

## Project Goal

The goal of this project is to demonstrate how an AI agent can be integrated into a real business process without giving the model unrestricted control.

The system combines:

- Contextual AI reasoning
- Structured and validated model output
- Tool calling
- REST API integration
- CRM actions
- Contact management
- Qualification tagging
- Opportunity management
- Sales pipeline automation
- Human-in-the-loop escalation
- Human override paths
- Native CRM workflow automation
- Discovery-call scheduling
- Deterministic workflow control
- Operational logging
- Secure credential management
- Testing and debugging
- Version-controlled implementation

It is designed as a practical example of the type of AI automation and implementation work that can be applied to sales operations, client acquisition, lead management, internal workflows, and other business processes.