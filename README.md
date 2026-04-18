# synozur-BoD
Synozur Board of Directors

# Board of Directors — Personas and Instructions

A nine-member virtual board of directors built on Claude. This repository
contains the instruction sets and grounding documents for each board
member, plus the master orchestrator instructions that coordinate them.

---

## What this is

A multi-agent advisory board designed to give consequential decisions the
benefit of nine distinct, well-defined perspectives — without flattening
them into consensus.

The board operates in three modes:

- **ADVISORY** — exploration and reframing, no decision
- **BOARD** — formal vote with rationale per member, governance flags
  surfaced
- **REVIEW** — post-decision retrospective with explicit naming of what
  was right and wrong

---

## The nine board members

| Member | Seat |
|--------|------|
| **Warren Buffett** | Capital allocation, moats, downside protection |
| **Paul Krugman** | Macroeconomics, distributional consequences, mechanism analysis |
| **Mark Cuban** | Revenue realism, customer traction, founder execution |
| **Bill Belichick** | Execution discipline, preparation, role clarity |
| **Barack Obama** | Strategic deliberation, coalition-building, surfacing dissent |
| **Satya Nadella** | Platform strategy, ecosystem, cultural transformation |
| **Steven Spielberg** | Narrative coherence, audience empathy, creative courage |
| **Sheryl Sandberg** | Operational scaling, prioritization, people systems |
| **Dr. John Hillen** | Strategy as discipline, board altitude, definitional rigor |

Each member occupies a structurally distinct seat. No two members share a
lens.

---

## Repository contents

For each persona:

- **`grounding.md`** — Loaded as a knowledge source on the persona's
  child agent. Contains verified biographical material, frameworks,
  quotes, real cases, and acknowledged blind spots.
- **`instructions.md`** — Loaded as the persona's child agent
  instructions. Governs behavior, voice, length budgets, and vote format.

The master orchestrator instructions live in
`master/master-instructions.md` and are loaded into the master agent
that coordinates the nine children.

---

## Deployment in Copilot Studio

### Architecture

This system uses a **master agent + nine child agents** pattern:

- **Master agent** — orchestrates the board. Routes questions to relevant
  members, establishes facts once, assembles attributed responses,
  enforces timeout and citation discipline, runs the vote.
- **Nine child agents** — one per persona. Each is invoked by the master
  with established facts and the question, returns its in-voice
  contribution within strict word budgets.

Child agents (rather than connected agents) are recommended for this use
case because they share the master's environment and have lower
invocation latency. The tradeoff is that child agents do not have
independent telemetry streams in Application Insights — see the
Observability section below.

### Setup steps

1. **Create the master agent** in your Copilot Studio environment.
   - Recommended model: **Claude Opus 4.1**
   - Paste the contents of `master/master-instructions.md` into the
     agent's instructions field
   - Disable web search and remove all tools (the master orchestrates
     children, it doesn't reason from external sources)

2. **Create nine child agents** under the master, one per persona.
   - Recommended model: **Claude Sonnet 4.5** for each (or Haiku 4.5
     for cost-sensitive deployments — voice fidelity holds well at the
     compressed length budgets)
   - For each child:
     - Paste the contents of `personas/{name}/instructions.md` into the
       child agent's instructions field
     - Upload `personas/{name}/grounding.md` as the child agent's
       knowledge source
     - Disable web search at the tool level (not just in instructions)
     - Strip tool list to zero — children reason from grounding only

3. **Configure routing.** The master agent's instructions handle
   routing internally. Verify that the master agent's child agent list
   includes all nine personas with descriptive names matching the
   routing rules in the master instructions.

4. **Connect Application Insights** for observability. See the
   Observability section.

5. **Test in dev** with a known question before promoting to production.
   Suggested test: a BOARD-mode vote on a non-trivial business
   question. Verify all nine personas vote, each contribution is
   attributed with a header, and no auto-citations appear.

### Recommended Copilot Studio settings

- **Generative AI → Content moderation:** Lower from default if you see
  refusals from personas like Cuban, Krugman, Spielberg on substantive
  business discussion. Default settings are sometimes too aggressive
  for direct analytical voice.
- **Generative AI → Citations:** Disable citations at the master agent
  level if available. Auto-generated citation markers create unreadable
  noise in board synthesis output.
- **Application Insights:** Enable "Include sensitive properties" and
  "Include activities" toggles to capture child agent invocations and
  content for diagnosis.

---

## Observability

Because child agents share the master's telemetry stream, you cannot
view per-persona performance in a separate Application Insights
resource. Three approaches work:

1. **Application Insights → Agents (Preview) view** — surfaces
   per-child-agent invocations, latency, and errors when available in
   your tenant.

2. **Application Insights → Workbooks → Copilot Studio Dashboard** —
   prebuilt workbook against the master agent's telemetry. Customize
   tiles to break down activity by child agent name.

3. **Custom KQL queries** against the master's `customEvents` table.
   Filter by `customDimensions['ChildAgentName']` (the exact field
   name varies by Copilot Studio version) to isolate per-persona
   behavior.

For diagnosing silent dropouts, refusals, or timeouts, the master
agent's pre-response validation logic (built into
`master-instructions.md`) surfaces these explicitly in the response —
making failures visible without requiring telemetry deep-dives.

---

## Design principles

A few non-obvious decisions shaped this work.

**Disagreement is the product.** The system preserves disagreement
rather than resolving it. The convergence note names where members
aligned AND where they diverged. Unanimous votes are treated as a
signal to scrutinize, not to celebrate.

**Selective routing for analysis, all-nine for votes.** Not every
question needs every voice. But a vote is a governance act and
requires the full board, regardless of topical relevance. The master
enforces this distinction.

**Honest persona blind spots.** Each grounding document includes
acknowledged failures — Buffett on Dexter Shoe, Krugman on the
2021-23 inflation miss, Belichick on the Butler benching, Obama on
the Syria red line, Sandberg on the Cambridge Analytica period.
Personas that defend themselves into infallibility lose credibility
quickly.

**Verified material only.** Every quote, framework, and case in the
grounding documents has been verified against published sources.
Frameworks attributed to other thinkers (Lafley/Martin's Playing to
Win, Collis/Rukstad's 35-word challenge, etc.) are credited to them.

**Failures must be visible.** The master agent treats silent dropouts,
refusals, and content-filter hits as visible operational signals to
surface explicitly — never as results to silently work around.

---

## Operating notes

- **Length budgets matter.** Per-persona contributions are capped at
  150-250 words in BOARD mode. This is enforced in both the master and
  child instructions. If a persona consistently exceeds the budget,
  check that the length discipline rules are present in that persona's
  instructions.
- **Citations should be invisible.** If you see "cite:1 Citation-1" or
  "[synozur.sh...epoint.com]" appearing in output, citation suppression
  isn't taking effect. Check the master agent's CITATION AND SOURCE
  DISCIPLINE section and verify the platform-level citation toggle is
  off.
- **Vote completeness is non-negotiable.** A vote with fewer than nine
  voices is a failure to be diagnosed, not a result to be reported.
  The master agent's pre-response validation should surface any
  missing personas explicitly.
- **Solution import gotcha.** When promoting from dev to prod via
  managed solution, verify that knowledge source connection references
  remap correctly to the prod environment. Stale connection references
  are the most common cause of silent persona dropouts in production.

---

## Limitations

- This is a structured way to surface considerations and pressure-test
  reasoning. The output is input to a decision, not the decision
  itself.
- The personas are reconstructions, not the actual people. They reason
  from documented frameworks and stated principles.
- Quality of output depends heavily on quality of input. A poorly
  framed question produces poorly framed analysis.
- Cost scales with fan-out. A nine-member BOARD mode session with Opus
  on the master and Sonnet on members consumes meaningful tokens.

---

## License and use

[ADD LICENSE INFORMATION HERE]

The persona grounding documents reference real public figures and draw
on their published material. Using these personas commercially or in
publication may have additional considerations beyond software
licensing.

---

## Acknowledgments

Persona grounding draws from the published work and documented decisions
of the individuals represented. The Hillen persona benefits from Dr.
John Hillen's *The Strategy Dialogues* (2025); other personas draw on
their respective subjects' books, letters, interviews, and public
records as cited within each grounding document.
