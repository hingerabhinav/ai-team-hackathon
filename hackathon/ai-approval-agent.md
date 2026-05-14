# AI Approval Agent

## Pitch

Pipelines spend too much time waiting for manual approvals because approvers need to gather context from Harness, Git, Jira, Slack, security scans, deployment health, and team knowledge before making a decision. The AI Approval Agent turns an approval step into an evidence-backed decision point: it gathers the relevant context, checks hard blockers, explains its reasoning, and either approves, rejects, approves with conditions, or escalates to a human.

**Tagline:** Stop waiting on approvals. Let an agent approve low-risk changes, block unsafe ones, and escalate ambiguous releases with evidence.

## Product Use Case

The agent runs inside a Harness pipeline before a high-risk step such as production deploy, infrastructure change, feature flag rollout, or rollback execution. It has an LLM connector, an MCP connector, and a system prompt that defines the approval policy.

Instead of a human manually jumping across tools, the agent creates an approval evidence packet from available MCP tools:

- Harness execution context: pipeline, stage, service, environment, artifact, inputs, prior step results.
- Git context: PR, commits, changed files, reviewers, code owners.
- Work tracking context: Jira ticket, change request, incident or support ticket.
- Collaboration context: linked Slack thread or user-provided approval notes.
- Risk context: test results, security scans, rollback readiness, current environment health.
- Governance context: required approvals, freeze windows, policy checks, ownership.

The agent then emits one of four decisions:

- `APPROVE`: evidence is complete, risk is low, hard gates passed.
- `APPROVE_WITH_CONDITIONS`: deployment can proceed only with guardrails such as canary, extra monitor, or limited environment scope.
- `REJECT`: a deterministic blocker or severe risk is present.
- `NEEDS_HUMAN`: required evidence is missing, ambiguous, or confidence is too low.

## Why This Matters

Manual approvals are often slow because the context is fragmented across enterprise systems. Existing approval steps capture a human decision, but not the work required to reach that decision. The AI Approval Agent makes the approval process faster and more auditable by producing a structured explanation every time.

The agent is not a black-box approver. It is an approval assistant with a strict decision contract. It can only approve when required evidence is present and deterministic blockers pass. Otherwise, it escalates.

## Marketplace Fit

This is a strong marketplace agent because it does not require a customer-specific memory corpus to be useful. Customers can install it, attach their Harness MCP connector and LLM provider, and configure the approval policy in the system prompt or pipeline inputs.

A persistent organizational memory layer can be a future enhancement, but v1 should stay stateless and evidence-based. This keeps onboarding simple and avoids the trust and data-governance complexity of indexing enterprise history.

## Slack And Collaboration Context

Many deployment decisions start in Slack or coworker notes. For marketplace v1, the agent should not assume broad access to all DMs or private conversations. Instead, it should consume explicitly linked or supplied context:

- a Slack thread URL pasted into a pipeline input
- a Jira ticket with copied discussion context
- a manual approval note
- a channel message selected by the user
- a generated change request summary

If a Slack MCP connector has the required permissions, it may be able to read DMs or private channels, but that should be opt-in and permission-scoped. The safer product default is: the agent reads collaboration context only when the user explicitly links or provides it.

## Decision Model

The agent should separate deterministic gates from LLM judgment.

Hard blockers should always reject or escalate:

- required tests failed
- critical security vulnerability exists
- production health is red
- change freeze is active
- rollback plan is missing
- required Jira or PR linkage is missing
- requester is unauthorized
- policy requires a human approval

LLM reasoning should only happen after hard gates pass. It should evaluate soft signals:

- change size and changed files
- environment criticality
- blast radius
- current service health
- PR and ticket context
- rollback readiness
- release intent
- risk mitigations

Suggested confidence bands:

- `>= 0.85`: approve or approve with clear conditions
- `0.65 - 0.85`: approve with conditions or request human review
- `< 0.65`: request human review
- missing required evidence: request human review

## Example Demo Flow

### Safe Change

1. Pipeline reaches the AI approval step before production deploy.
2. Agent gathers PR, Jira, tests, artifact, scan, and environment health.
3. All hard gates pass; the change is low risk.
4. Agent approves and attaches the evidence packet to the execution.

### Risky Change

1. Pipeline reaches approval for a production config change.
2. Agent sees tests passed, but security scan has a critical issue or rollback plan is missing.
3. Agent rejects or escalates with exact evidence.

### Ambiguous Change

1. Slack note says the deploy is urgent, but no Jira ticket or rollback owner is linked.
2. Agent returns `NEEDS_HUMAN`.
3. It asks for the missing evidence instead of guessing.

## Output Artifact

Every run should produce an approval evidence packet:

```json
{
  "decision": "APPROVE_WITH_CONDITIONS",
  "confidence": 0.82,
  "conditions": ["canary first", "monitor checkout error rate for 30 minutes"],
  "hard_gates": {
    "tests": "passed",
    "security": "no critical findings",
    "rollback_plan": "present",
    "change_freeze": "not active"
  },
  "evidence": [
    {
      "type": "pull_request",
      "summary": "Small timeout config change",
      "source": "PR-123"
    },
    {
      "type": "jira",
      "summary": "Customer-impacting timeout fix",
      "source": "PROJ-456"
    }
  ],
  "reasoning_summary": "Approval allowed because required gates passed and blast radius is limited, but canary is required due to production impact."
}
```

## Future Extension

After v1, add organizational memory as an optional extension. The memory layer can learn from previous approvals, incidents, and deployment outcomes, then contribute cited evidence to future decisions. It should remain advisory, scoped, and auditable rather than hidden prompt context.
