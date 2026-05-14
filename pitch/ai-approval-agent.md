# AI Approval Agent — Hackathon Design

**Status:** Iterating. Last updated 2026-05-14.
**Format:** Internal Harness hackathon, 1–3 days, mixed judges (demo + storytelling weighted).
**Team:** 2–3 people, mostly backend.
**Source pitch:** https://harness.atlassian.net/wiki/spaces/~5eea78d11b849f0ac0a32960/pages/23761191076/AI+Approval+Agent

---

## 1. Reframe (LOCKED)

**Old pitch:** "AI auto-approves safe deploys."
- Problem: enterprise approvers are *accountable*. They re-read everything anyway. Auto-approve becomes glorified summarization that adds a second artifact to read.

**New pitch:** "AI Approval Agent turns every approval into a 20-second, calibrated, defendable decision — by computing blast radius, predicting rollback risk from your own deploy history, and producing evidence the policy required but nobody filled in."

The human still clicks approve. The agent removes the *investigation* step, not the *accountability* step.

**Why this framing wins:** survives the "would a real customer turn this on?" judge question. Auto-approve stays as an opt-in for trivial cases — not the headline.

---

## 2. Three Pillars (LOCKED)

| Pillar | Demo proof | Why nobody else can do this |
|---|---|---|
| **Defendable** | Side-by-side blast radius map: services, regions, customers, dependencies | Needs the deploy fleet graph — only Harness has it |
| **Calibrated** | Live calibration plot: predicted vs. actual rollback rate on seeded history | Needs deploy outcome telemetry — only Harness has it |
| **Active** | Generates missing evidence (rollback verification, canary plan) on demand | Needs to *act* inside the pipeline, not just read context |

These three together = the moat. PR-approval AIs (GitHub, Atlassian Rovo, ChatGPT-MCP) cannot replicate any of them because they don't own deploy outcomes or fleet state.

---

## 3. Differentiation Decisions (LOCKED)

- **Headline moat:** Closed-Loop / Calibration (pillar 2)
- **Demo wow:** Risk-calibrated blast radius map (pillar 1)
- **V2 vision (one slide only):** Fleet Coordinator — cross-pipeline collision detection
- **Calibration data strategy:** Seeded history (50–200 synthetic past deploys with outcomes) + live-updating verdict during demo

---

## 4. Architecture (LOCKED — top-level)

### 4.1 Runtime
- **Custom Harness pipeline step** ("AI Approval Agent" plugin) inserted before/replacing a manual approval step.
- Reads pipeline execution context (service, env, artifact, stage) from Harness automatically.
- Outputs the verdict + evidence packet to the approval step's UI panel.

### 4.2 LLM strategy
- **Agent loop with MCP tool calls.** LLM iteratively calls tools until it has enough evidence, then emits a structured verdict.
- **Stage-reliability mitigations** (mandatory for live demo):
  - All MCP tool responses cached and replayable (`DEMO_MODE=record|replay`).
  - Hard timeout per tool call; agent falls back to "NEEDS_HUMAN: tool X timed out."
  - Pre-warm the model + tools 60s before demo to dodge cold starts.
  - "Re-run" button on the demo UI in case of nondeterminism.

### 4.3 Blast radius graph
- Source: **Harness CD service/env metadata** (services, envs, deployments) + **hand-curated `depends-on` edges** for the demo's 8–12 services.
- Stored as a small JSON next to the agent for the demo; in production this would be derived from CD + service mesh.
- Map renders: services touched, envs affected, regions, est. customer count (computed from a seeded customer-per-service table).

### 4.4 Calibrated risk model
- **Inputs (locked):**
  - Change size: files touched, lines added/deleted
  - Service rollback history: rollback rate over last N deploys for this service
  - Test/scan signals: tests pass/fail, critical/high vuln count, coverage delta
  - Env health: current error rate / saturation of target env (synthetic prom metric)
- **Model:** simple logistic regression (4–6 features) trained on **seeded synthetic history of 100–200 past deploys with outcomes**. Easy to re-train live during the demo when a new outcome is appended.
- **Output:** rollback probability + 95% CI + top 3 contributing features (SHAP-style attribution, even if just feature × coefficient).
- **Calibration plot** (the killer slide): predicted-vs-actual rollback rate bucketed across seeded history. Shows the model is honest, not just confident.

### 4.5 Verdict schema (evidence packet)
```
{
  "verdict": "APPROVE | APPROVE_WITH_CONDITIONS | REJECT | NEEDS_HUMAN",
  "confidence": 0.0-1.0,
  "rollback_probability": 0.0-1.0,
  "rollback_ci": [low, high],
  "top_risk_factors": [{ "name", "weight", "value" }],
  "blast_radius": {
    "services": [...], "envs": [...], "regions": [...],
    "est_customers_affected": int,
    "graph_image_url": "..."
  },
  "evidence_collected": [{ "source", "summary", "link" }],
  "missing_evidence": [{ "name", "auto_fillable": bool }],
  "actions_taken": [{ "action", "result", "artifact_url" }],
  "policy_blockers": [...],
  "recommended_conditions": [...],
  "human_summary_md": "..."
}
```

### 4.6 MCP connectors (demo set)
- **Harness CD MCP** — pipeline/service/env/artifact/deploy-history (most of these we'll mock with a thin adapter for demo)
- **Git MCP** — PR diff, files changed, author
- **Jira MCP** — linked ticket status, approvals
- **Security scan MCP** — vuln summary (mocked output)
- **Observability MCP** — current env health (mocked Prometheus)
- **"Active" MCP (the wow)** — `compute_blast_radius` (locked for demo). `verify_rollback_artifact` and `generate_canary_plan` listed as v2 in the deck.

The "Active" MCP is the differentiator. Other AI approval agents only have read-tools.

### 4.7 Risk model implementation note
- **Logistic regression** with 4–6 features, trained on seeded synthetic history.
- Attribution computed as `feature_value * coefficient` per feature, ranked top-3 — SHAP-style without the SHAP library.
- Re-trainable in <1s on the seeded dataset so we can show the model updating live during demo.

### 4.8 Slack notification output (V2 / optional for demo)
- **Closes the loop:** agent *reads* Slack context as input (mentioned in original pitch), now *writes* verdict summaries back to Slack as output.
- **Config:** opt-in per approval step. Example:
  ```yaml
  approval_step:
    ai_agent: true
    notify_slack:
      channel: "#prod-deploys"
      on: ["APPROVE_WITH_CONDITIONS", "REJECT", "NEEDS_HUMAN"]  # skip routine approves
  ```
- **Payload:** Slack Block Kit message with verdict badge, rollback probability, top risk factors, "View full evidence →" link to Harness UI.
- **Value props:**
  - Async visibility — team members see what happened without switching to Harness
  - Audit trail in team space — searchable history of verdicts where incidents are discussed
  - Reduces "what did the AI decide?" follow-up questions
- **Build cost:** ~2 hours (Slack webhook + Block Kit formatter). Only build if Day 2 goes well.
- **Demo treatment:** skip in live demo (4-min script is tight), but show a **screenshot on Slide 8 (V2/vision)** with 10-second mention: *"And the verdict posts to your team channel automatically — no one needs to ask 'did it approve?'"*

---

## 5. Demo Script (LOCKED — 4 min target)

**Setup on stage:** two browser tabs side-by-side. Left = today's Harness pipeline at a manual approval step. Right = same pipeline with the AI Approval Agent step.

### Beat 1 — The pain (0:00–0:30)
- Click into the manual approval. Show the wall: PR link, Jira link, scan dashboard, Grafana, runbook, Slack thread.
- Voiceover: "An approver answers the same eight questions every time. On a busy day, this is 40 minutes per approval, and the answer is almost always 'looks fine, approved.'"

### Beat 2 — The reveal (0:30–1:00)
- Switch to the AI Approval Agent step. Same pipeline, paused at the same point.
- Panel renders: verdict badge, rollback probability with CI, blast radius map, top-3 risk factors, evidence list.
- "Same pipeline. Same data. Twenty seconds instead of twenty minutes."

### Beat 3 — Defendable: blast radius (1:00–1:45)
- Hover the blast radius graph: 7 services, 2 envs, 3 regions, ~120k customers in path.
- Highlight the agent *computed* this — it called the `compute_blast_radius` MCP tool live. Not summarization. **Action.**
- Contrast slide: "GitHub Copilot, Atlassian Rovo, ChatGPT-MCP — none of them can do this. They don't see the deploy fleet."

### Beat 4 — Calibrated: the moat (1:45–2:45)
- Show the rollback probability: 3.2% [CI 1.8–5.4%]. Top factors: small diff (-2.1pp), service has clean rollback history (-0.8pp), target env healthy (-0.3pp).
- Click "calibration" tab. Show the **calibration plot**: predicted-vs-actual rollback rate across 200 seeded historical deploys, near-perfect diagonal.
- "This number is honest. It's anchored in *your* deploys. The model retrains every time a deploy succeeds or rolls back."
- Append a new outcome live → calibration plot updates.

### Beat 5 — The contrast deploy (2:45–3:30)
- Re-run pipeline with a riskier change: large diff, service has 18% recent rollback rate, env error rate climbing.
- Verdict flips to `NEEDS_HUMAN`. Risk: 34% [CI 24–46%]. Top factors: service rollback history (+12pp), large diff (+9pp), env error rate climbing (+5pp).
- Recommended conditions: "Wait for env error rate to settle below 0.5%, or run with canary at 5% for 10 min first."
- "The agent doesn't refuse — it tells you exactly what would change its mind."

### Beat 6 — Close (3:30–4:00)
- One slide: Defendable, Calibrated, Active. Three pillars. One moat: only Harness has deploy outcome telemetry + fleet graph.
- "This is not a chatbot. This is a calibrated decision system that earns trust by being right over time."

### Demo-mode safeguards (operational)
- Both pipelines pre-recorded with cached MCP responses.
- "Re-run" button in case of LLM nondeterminism.
- Backup video recording of the full demo cued up — if the live agent falters, switch to video mid-sentence and keep narrating.
- Calibration plot, risk number, and blast radius all server-rendered as fallback PNGs in case the live JS chart hiccups.

---

## 6. Build Plan (LOCKED — 3-day, 3-engineer)

### Real vs. faked

| Component | Real? | Why |
|---|---|---|
| Custom Harness pipeline step plugin | **Real** | Core "this is a Harness feature" credibility |
| Agent loop + LLM tool calls | **Real** | Headline architecture; cached responses for stage |
| `compute_blast_radius` MCP tool | **Real** | The wow demo moment; must work live |
| Logistic regression risk model + calibration plot | **Real** | The moat slide; cannot be faked |
| Seeded historical deploy dataset (100–200 rows) | **Real** | Pre-generated JSON, fed to the model |
| Harness CD context (services, envs, artifacts) | **Real for one demo pipeline**, mocked otherwise | Need one real pipeline to be believable |
| Git / Jira / scanner / observability MCPs | **Mocked thin adapters** | Stage reliability + time budget |
| Approval step UI panel | **Real React component** | This is what judges see; cannot be faked |

### Day-by-day, 3 engineers

**Day 1**
- **Eng A (platform):** scaffold Harness pipeline step plugin + agent runtime + MCP tool plumbing + `DEMO_MODE=record|replay` cache layer.
- **Eng B (data/ML):** generate seeded historical deploy dataset (100–200 rows with realistic features + outcomes); train + serialize logistic regression; build calibration plot generator.
- **Eng C (UI/active tools):** build the approval step UI panel (verdict, blast radius, risk, evidence) + `compute_blast_radius` tool with hand-curated `depends-on` JSON.

**Day 2**
- **Eng A:** wire up real LLM agent loop with mocked MCP adapters; lock the structured-output schema; record the demo's tool responses to cache.
- **Eng B:** SHAP-style attribution; live-retrain endpoint; calibration plot integration in UI.
- **Eng C:** polish UI, render blast radius graph (vis.js or react-flow), build the "contrast deploy" toggle.

**Day 3 (morning)**
- All hands: dress rehearsal, fix demo-mode reliability, record fallback video, lock the deck.
- 1pm: full run-through.
- 3pm: dress rehearsal in front of one external reviewer (not on team) for fresh-eyes feedback.

### Out of scope (V2 deck slides only)
- `verify_rollback_artifact` (sandbox rollback rehearsal)
- `generate_canary_plan` (auto canary)
- Fleet Coordinator (cross-pipeline collision)
- Real OTel-driven blast graph
- Real security scanner / observability integrations
- Slack notification output (only build if Day 2 schedule allows; otherwise screenshot-only on Slide 8)

---

## 7. Pitch Deck Outline (DRAFT v1)

**Target:** 8–10 slides, ~3 minutes of slide time around a 4-minute live demo. Mixed judge panel, demo + storytelling weighted.

**Narrative arc:** Pain → Why nobody fixed it → Our reframe → Demo → Why this is a moat → Roadmap → Ask.

---

### Slide 1 — Title (0:00–0:15)
- **AI Approval Agent**
- One-line tagline: *"From 20-minute investigation to 20-second decision — calibrated to your own deploy history."*
- Team names. Module: Harness CD.

### Slide 2 — The pain, in one number (0:15–0:45)
- "**8 questions × every approval × N approvals/week = the most expensive minutes in your pipeline.**"
- Show a real-looking screenshot of a Harness manual approval with 7 tabs open behind it.
- Imply (don't promise) data: "internal pulse — engineers report 15–40 min per prod approval, 70%+ of which is gathering context already in our systems."

### Slide 3 — Why nobody has solved it yet (0:45–1:15)
- The competitors slide. 3 columns.
  - **GitHub Copilot / Atlassian Rovo:** can read PRs and tickets. Cannot see deploy fleet or outcomes.
  - **ChatGPT + MCPs:** generic agent. No calibration, no fleet graph, no pipeline integration.
  - **Custom in-house bots:** brittle scripts; no learning loop.
- Bottom row: "Approval is a *deploy* problem. The deploy platform is the only place to solve it correctly."
- This is the slide that earns the moat.

### Slide 4 — The reframe (1:15–1:45)
- Headline: **"We don't replace the click. We replace the investigation."**
- Three pillars (icons):
  - **Defendable** — blast radius computed, not summarized
  - **Calibrated** — rollback risk anchored to your history
  - **Active** — generates evidence the policy required
- Note: "Auto-approve is opt-in for trivial cases. Headline value is *speed and trust at the human-in-the-loop step.*"

### Slide 5 — Demo (1:45–5:45) — LIVE
- Hand off to live demo (4 min). See § 5.
- Slide on screen during demo: minimal — just the three-pillar bug at the corner so the narrative anchors are visible.

### Slide 6 — How it works (5:45–6:15)
- One architecture diagram:
  - Harness pipeline → Approval step plugin → Agent loop → MCP tools (Harness CD, Git, Jira, scanners, observability, **Active tools**) → Verdict + Evidence packet → UI panel + audit log
- Highlight the two boxes that are *only possible at Harness*: deploy outcome history (calibration), service+env graph (blast radius).

### Slide 7 — The moat, in one chart (6:15–6:45)
- The calibration plot.
- Caption: "200 seeded historical deploys. Predicted-vs-actual rollback rate. The diagonal is the win — *we are right about how often we are right*."
- Sub-caption: "GitHub/Atlassian/ChatGPT cannot draw this chart. They don't have the data."

### Slide 8 — V2 / vision (6:45–7:15)
- Four items teased:
  - **Active gap-fillers** (rollback rehearsal, canary plan generation)
  - **Fleet Coordinator** (cross-pipeline collision detection)
  - **Org-wide trust score** (services earn auto-approve eligibility through track record)
  - **Slack loop-closure** — screenshot of a Slack Block Kit message showing verdict posted to `#prod-deploys`. Caption: *"The verdict posts to your team channel automatically — no one needs to ask 'did it approve?'"*
- "The agent gets smarter every deploy. The platform gets safer every approval."

### Slide 9 — Why this matters to a Harness customer (7:15–7:45)
- Three customer-language bullets:
  - **Faster deploys without lower trust** — eliminate the 20-minute approval queue, keep accountability with humans.
  - **Audit-ready by default** — every decision ships with the evidence packet that compliance asked for.
  - **A reason to choose Harness over GitHub/GitLab CI** — this feature is impossible without our deploy graph.

### Slide 10 — The ask (7:45–8:00)
- "Pick us, and we ship a private beta of the pipeline step within 6 weeks."
- Team contact + Slack channel + "thanks."

---

**Slide-design notes:**
- Live demo slot is the longest single beat — keep slide chrome out of the way during it.
- The calibration plot slide (#7) is the screenshot that will end up in re-tells; design it to read in 5 seconds. One chart, two labels, one caption.
- Slide 3 (competitor matrix) is what closes the "but why isn't this just an LLM wrapper?" question; do not skip it.

## 8. Open Questions Log

- **Calibration data realism:** how synthetic can the seeded history look before judges call it out? Plan: generate from a plausible joint distribution (correlate diff size, env health, rollback) so the calibration plot's diagonal is *earned*, not hand-drawn.
- **Pipeline plugin productionability:** is custom-step the right packaging long-term, or should this be a first-class step type in Harness CD? (Punt to V2 conversation.)
- **Auto-approve threshold story:** policy says "auto-approve only when rollback prob < X% AND blast radius < Y customers AND no critical vulns." Need to land this on a slide vs. live demo — probably mention in voiceover during Beat 2.
- **Privacy/security of seeded data:** make sure no real internal Harness deploy data leaks into the demo dataset — fully synthetic.

---

## 8. Open Questions Log

- (none yet — will accumulate as we iterate)
