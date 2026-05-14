# AI Approval Agent Build And Iteration Plan

## Goal

Build a marketplace-friendly Harness Agent Step that runs before a real approval or deployment gate, gathers evidence through MCP connectors, and outputs approval variables that downstream approval/gate steps can consume.

The v1 agent is stateless and evidence-based. It does not require an organizational memory corpus.

## Agent Contract

The agent produces one of four decisions:

- `APPROVE`
- `APPROVE_WITH_CONDITIONS`
- `REJECT`
- `NEEDS_HUMAN`

It writes these output variables to `$DRONE_OUTPUT`:

- `AI_APPROVAL_DECISION`
- `AI_APPROVAL_ALLOWED`
- `AI_APPROVAL_REQUIRES_HUMAN`
- `AI_APPROVAL_CONFIDENCE`
- `AI_APPROVAL_CONDITIONS`
- `AI_APPROVAL_SUMMARY`
- `AI_APPROVAL_PACKET_B64`

The downstream approval or gate step should primarily use `AI_APPROVAL_ALLOWED` and `AI_APPROVAL_DECISION`.

## MCP Connector Plan

Start with a minimal connector set, then add optional context providers.

### Required

- Harness MCP connector: execution, pipeline, service, environment, artifact, prior step status.

### Strongly Recommended

- SCM MCP connector: PR, commits, changed files, reviewers, code owners.
- Jira or work-tracking MCP connector: ticket, change intent, priority, approval notes.

### Optional

- Slack MCP connector: explicit linked Slack thread or message context only.
- Security MCP connector: scan status, critical vulnerabilities, SBOM or artifact findings.
- Observability MCP connector: current service health, active incidents, error rate.

## System Prompt Tuning Plan

Tune the prompt around reliability, not creativity.

1. **Context resolution**
   Prefer template inputs, then Harness runtime env vars, then MCP execution metadata. Never invent missing values.

2. **Evidence packet discipline**
   Force the agent to build a structured evidence packet before deciding.

3. **Hard blockers**
   Deterministic blockers must reject or escalate before LLM judgment:
   - failed required tests
   - critical vulnerabilities
   - missing ticket when policy requires it
   - missing rollback plan for prod
   - active change freeze
   - red environment health
   - human approval required by policy

4. **Confidence thresholds**
   Use input thresholds:
   - default auto approve threshold: `0.85`
   - default human review below threshold: `0.65`

5. **Escalation behavior**
   Missing evidence should produce `NEEDS_HUMAN`, not a guessed approval.

6. **Output stability**
   The final action must write all outputs to `$DRONE_OUTPUT` as single-line values.

## Iteration Strategy

### Iteration 1: Static Inputs

Run with only user-provided inputs:

- approval policy
- service
- environment
- change summary
- PR URL
- Jira URL
- rollback plan

Expected result: stable output variables and an explainable decision without relying on many tools.

### Iteration 2: Harness Context

Add Harness MCP and validate:

- execution metadata is discovered
- previous steps are inspected
- failed or skipped required checks are treated as blockers
- output variables remain stable

### Iteration 3: SCM And Jira Context

Add SCM + Jira MCP connectors:

- changed files influence risk
- PR reviewers/code owners are captured
- Jira ticket intent is summarized
- missing PR/Jira evidence escalates when policy requires it

### Iteration 4: Collaboration Context

Add explicit Slack thread URL support:

- agent only reads linked Slack context
- it does not browse unrelated DMs or private conversations
- Slack context influences intent and constraints, not hard approval by itself

### Iteration 5: Risk Providers

Add security and observability:

- critical findings reject or escalate
- red service health rejects or escalates
- warnings can produce `APPROVE_WITH_CONDITIONS`

## Test Matrix

Create representative test runs:

| Scenario | Expected Decision |
|---|---|
| Small change, tests passed, Jira linked, rollback present | `APPROVE` |
| Prod deploy with rollback missing | `NEEDS_HUMAN` |
| Critical security finding | `REJECT` |
| Tests unknown and policy requires tests | `NEEDS_HUMAN` |
| Risky change but canary and monitor present | `APPROVE_WITH_CONDITIONS` |
| Slack says urgent but no Jira or rollback owner | `NEEDS_HUMAN` |

## Downstream Approval Usage

The actual approval step can use the agent output as an input or condition. Example references:

```text
<+pipeline.stages.<stage_id>.steps.runAiApprovalAgent.output.outputVariables.AI_APPROVAL_ALLOWED>
<+pipeline.stages.<stage_id>.steps.runAiApprovalAgent.output.outputVariables.AI_APPROVAL_DECISION>
<+pipeline.stages.<stage_id>.steps.runAiApprovalAgent.output.outputVariables.AI_APPROVAL_SUMMARY>
```

For the hackathon demo, use:

- auto proceed when `AI_APPROVAL_ALLOWED == "true"`
- require manual approval when `AI_APPROVAL_REQUIRES_HUMAN == "true"`
- fail/block when `AI_APPROVAL_DECISION == "REJECT"`

## Success Criteria

- The agent always writes all output variables.
- The agent never approves when required evidence is missing.
- The agent explains every decision with cited evidence.
- The downstream approval step can branch on the agent decision.
- The demo shows at least three outcomes: approve, reject, needs human.
