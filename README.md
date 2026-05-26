# API Log Analysis Tool
### A Self-Improving, Knowledge-Graph-Powered Incident Analysis System for GitHub Copilot

---

## Table of Contents

1. [What This Tool Does](#1-what-this-tool-does)
2. [How It Works — The Big Picture](#2-how-it-works--the-big-picture)
3. [Prerequisites](#3-prerequisites)
4. [Folder Structure](#4-folder-structure)
5. [The Four Stages Explained](#5-the-four-stages-explained)
   - [Stage 1 — Knowledge Graph Builder](#stage-1--knowledge-graph-builder)
   - [Stage 2 — Log Analyzer](#stage-2--log-analyzer)
   - [Stage 3 — Self-Improvement Engine](#stage-3--self-improvement-engine)
   - [Stage 4 — Runbook & Troubleshooting Auto-Update](#stage-4--runbook--troubleshooting-auto-update)
6. [Files Reference](#6-files-reference)
7. [First-Time Setup (Step by Step)](#7-first-time-setup-step-by-step)
8. [Day-to-Day Usage (Per Incident)](#8-day-to-day-usage-per-incident)
9. [How the Self-Improvement Loop Works](#9-how-the-self-improvement-loop-works)
10. [Understanding the Output Files](#10-understanding-the-output-files)
11. [Human Review Process](#11-human-review-process)
12. [Tips and Best Practices](#12-tips-and-best-practices)
13. [Troubleshooting the Tool Itself](#13-troubleshooting-the-tool-itself)
14. [Quick Reference Card](#14-quick-reference-card)

---

## 1. What This Tool Does

This tool turns your existing documentation — source code, runbooks, and troubleshooting guides — into a live, self-learning incident analysis engine that runs entirely inside **GitHub Copilot**.

Instead of manually reading logs and cross-referencing documentation during an incident, you paste your logs into Copilot Chat and get back a structured incident report in seconds: what broke, why it broke, which services are at risk, and exactly what to do about it — all sourced from your own runbooks.

**What makes it different from asking Copilot directly:**
- It builds a structured **Knowledge Graph** of your entire system from your actual code and docs — so it knows your services, your flows, your error signatures, not just generic best practices
- It learns from every incident — unknown failures get added to the graphs automatically so the next time the same issue appears, it is caught and diagnosed instantly
- It writes back to your runbooks and troubleshooting guides — documentation stays up to date with what is actually happening in production, not just what was written at release time
- Everything is versioned and traceable — you can see exactly what the system learned and when

---

## 2. How It Works — The Big Picture

The tool is built around four prompts stored as `.md` files, each run in GitHub Copilot at different points in the incident lifecycle. Together they form a closed loop:

```
YOUR DOCUMENTATION                         PRODUCTION INCIDENTS
(code, runbooks, guides)                   (raw logs from Copilot Chat)
        │                                           │
        ▼                                           ▼
  ┌───────────┐    graphs     ┌───────────────────────────┐
  │  STAGE 1  │ ──────────►  │        STAGE 2             │
  │  Build    │              │  Analyze logs against       │
  │  Graphs   │  ◄────────── │  graphs → incident report  │
  └───────────┘  Stage 1     └────────────┬───────────────┘
       ▲         re-run                   │ incident report
       │         triggered                ▼
       │                    ┌───────────────────────────┐
       │     updated        │        STAGE 3             │
       │     docs           │  Update graphs + heuristics│
       │                    │  with what was learned     │
       │                    └────────────┬───────────────┘
       │                                 │ new knowledge
       │                                 ▼
       │                    ┌───────────────────────────┐
       └────────────────────│        STAGE 4             │
         writes back to     │  Write new runbook and     │
         input/             │  troubleshooting entries   │
                            │  back to documentation     │
                            └───────────────────────────┘
                                   (human review)
```

After each full cycle, the system is smarter. Failures it could not recognize before are now in the graphs. Remediations that did not work are flagged. Heuristics that predict cascading failures are learned. The unknown failure rate approaches zero over time.

---

## 3. Prerequisites

| Requirement | Details |
|---|---|
| **GitHub Copilot** | Copilot Chat enabled in VS Code or JetBrains. Both Agent Mode and Chat are used. |
| **VS Code** (recommended) | For Agent Mode (Stages 1, 3, 4). Copilot Chat works in most IDEs. |
| **Your codebase** | The source code of the API or service being monitored, checked out locally |
| **Runbooks** | Operational runbooks as `.md` or `.txt` files. Even rough drafts work. |
| **Troubleshooting guides** | Any existing diagnostic documentation as `.md` or `.txt` files |
| **Raw logs** | Any plaintext log output — from kubectl, Azure Monitor, application logs, Istio, etc. |

No external services, no API keys, no installations required beyond Copilot.

---

## 4. Folder Structure

Set up your project folder exactly like this before starting:

```
project-root/
│
├── input/                              ← SOURCE DOCUMENTATION (you populate this)
│   │
│   ├── codebase/                       ← Your service source code
│   │   ├── src/
│   │   ├── config/
│   │   └── (any code files)
│   │
│   ├── runbooks/                       ← Operational runbooks, one file per runbook
│   │   ├── runbook-qod.md
│   │   ├── runbook-npm.md
│   │   └── (one .md or .txt per runbook)
│   │
│   └── troubleshooting/               ← Troubleshooting guides, one file per guide
│       ├── ts-guide-auth.md
│       ├── ts-guide-timeouts.md
│       └── (one .md or .txt per guide)
│
├── output/                             ← GENERATED FILES (tool writes these)
│   ├── knowledge-graph.yaml            Stage 1 creates → Stage 2 reads
│   ├── context-graph.yaml              Stage 1 creates → Stage 2 reads
│   ├── incident-report-<ts>.md         Stage 2 creates → Stage 3 reads
│   ├── stage2-heuristics-injection.md  Stage 3 creates → Stage 2 reads next run
│   ├── prompt-heuristics.md            Stage 3 maintains
│   └── graph-changelog.md              Stage 3 appends (version history)
│
└── prompts/                            ← THE PROMPT FILES (this tool)
    ├── stage1-graph-builder.md
    ├── stage2-log-analyzer.md
    ├── stage3-self-improvement.md
    └── stage4-docs-update.md
```

**Important rules:**
- The `output/` folder starts empty. The tool creates everything inside it.
- Never manually edit files in `output/` — they are managed by the tool.
- The `input/` folder is where you add and update your documentation.
- After Stage 4 runs, it writes back into `input/runbooks/` and `input/troubleshooting/` — these are the only times `input/` is modified by the tool.

---

## 5. The Four Stages Explained

### Stage 1 — Knowledge Graph Builder

**File:** `prompts/stage1-graph-builder.md`
**Run in:** Copilot Agent Mode
**When:** Once per release, or whenever your runbooks or code change significantly
**Time to run:** 2–5 minutes depending on codebase size

#### What it does

Stage 1 reads every file in your `input/` folder and builds two structured knowledge files that every subsequent stage uses:

**`output/knowledge-graph.yaml` — the "what exists" layer**

This file contains every service, API, endpoint, component, and infrastructure node in your system — extracted from your actual code and documentation. For each entity it records what it does, who owns it, and what it connects to.

It also captures every relationship between entities: which service calls which, what protocol is used, what the criticality of that connection is, and any SLA or timeout configuration found in the code.

Example of what gets extracted:
```yaml
entities:
  - id: qod_api
    name: QoD API
    type: API
    description: Quality on Demand API handling bandwidth allocation requests
    owner: camara-platform-team
    tags: [critical, egress, camara, gsma]

relationships:
  - from: qod_api
    to: istio_gateway
    relation: ROUTES_THROUGH
    protocol: HTTP
    criticality: HIGH
    notes: 30s timeout, 3 retries configured
```

**`output/context-graph.yaml` — the "what happens" layer**

This file captures operational knowledge: the flows that requests follow through your system, every known failure mode with its exact log signatures, and every remediation with its step-by-step fix.

This is the file that makes Stage 2 intelligent. Instead of guessing what an error means, Stage 2 looks up the exact log pattern in this file and finds the root cause and fix that your runbooks documented.

Example:
```yaml
failure_modes:
  - failure_id: istio_upstream_reset
    name: Istio upstream connection reset
    error_signatures:
      - "upstream connect error or disconnect/reset before headers"
      - "503 UC"
    affected_flows: [qod_allocation_flow]
    root_cause: Upstream pod crashed or was evicted before response
    blast_radius: [qod_api, edr_adapter]
    severity: P2
```

#### What it prints

Before writing the files, Stage 1 prints a **File Inventory** table listing every file it read and what it found. Check this to confirm nothing was missed. After writing, it prints a Coverage Summary showing totals and any gaps — things referenced in the docs that could not be matched to a known entity.

---

### Stage 2 — Log Analyzer

**File:** `prompts/stage2-log-analyzer.md`
**Run in:** Copilot Chat
**When:** Any time you have logs to analyze — during or after an incident
**Time to run:** 30–90 seconds

#### How to run it

1. Open Copilot Chat in VS Code
2. In the chat input box, type `#file:` and select `prompts/stage2-log-analyzer.md`
3. In the same message, paste your raw logs
4. Send

That is all. Copilot reads the instructions from the file, reads the graph files from your `output/` folder (which Stage 1 created), and analyzes the logs you pasted.

#### What it does internally (you do not need to manage this)

Stage 2 runs five analysis phases in sequence:

**Phase 1 — Parse** every log line, extracting timestamp, level, source service, message, error codes, and trace/correlation IDs. Lines with the same trace ID are grouped into request chains so you see the full journey of a failing request, not isolated log lines.

**Phase 2 — Match** each error group against the `error_signatures` in `context-graph.yaml`. If a log line contains `"upstream connect error or disconnect/reset before headers"`, it matches it to the `istio_upstream_reset` failure mode and immediately knows the affected flows, blast radius, and severity.

**Phase 3 — Assess impact** by tracing downstream along the relationships in `knowledge-graph.yaml`. If service A is failing and service B depends on A with HIGH criticality, service B is flagged AT-RISK even if it has not logged any errors yet. SLA breaches are detected by comparing log timestamps against the `sla_ms` values in the context graph.

**Phase 4 — Hypothesize** root causes ranked by confidence percentage. It identifies the single earliest log line that marks the onset of the problem — often several minutes before the alert fired.

**Phase 5 — Map remediations** from `context-graph.yaml` to each detected failure, sequenced by priority. If a failure has no remediation in the graph, it is flagged as UNKNOWN — a gap that Stage 3 and 4 will fill.

If `output/stage2-heuristics-injection.md` exists (created by Stage 3 after earlier runs), Stage 2 loads those learned heuristics before starting analysis. This is what makes the system progressively smarter.

#### What it produces

Stage 2 writes `output/incident-report-<timestamp>.md` containing:

| Section | Content |
|---|---|
| Executive Summary | 2–4 sentences: what broke, when, user impact, next action |
| Detected Incidents | One block per incident with severity, SLA breach, matched failure ID, confidence-ranked hypotheses, blast radius |
| Remediation Plan | Prioritized action table with owner and rollback option |
| Unknown Failures | Log patterns with no graph match — gaps to fill |
| Timeline Reconstruction | Full chronological event sequence from the logs |
| Detection Gap Analysis | Silent periods, missing trace IDs, services that produced no logs |
| Graph Feedback | Specific suggestions for what to add or fix in the graphs |

---

### Stage 3 — Self-Improvement Engine

**File:** `prompts/stage3-self-improvement.md`
**Run in:** Copilot Agent Mode
**When:** Immediately after every Stage 2 run — without exception
**Time to run:** 1–3 minutes

#### What it does

Stage 3 reads the incident report that Stage 2 just wrote and applies everything learned back into the system. It runs four phases:

**Phase 1 — Apply Section 7 feedback** from the incident report directly into the graph files. Every checkbox item in Graph Feedback is executed as a live patch. New failure modes are created. Remediations are updated. New entities are added.

**Phase 2 — Cross-report pattern detection.** Stage 3 reads *all* past incident reports and looks for patterns across them:

- If the same failure appears in 3 or more reports, its severity is automatically promoted one level (P3→P2, P2→P1)
- If the same unrecognized log pattern appears in 2 or more reports, it is automatically created as a new `failure_mode` entry (marked DRAFT for human review)
- If a remediation was applied but the same incident recurred in the next report's time window, that remediation is flagged `ineffective: true` and future analyses will escalate immediately instead of retrying it
- If the same entity produces zero logs across 2 or more runs, it is tagged `monitoring_gap: true` — a signal that observability coverage needs improving

**Phase 3 — Heuristic evolution.** Learned rules are written to `output/prompt-heuristics.md`. Each rule records what triggered it, what action to take, and its current confidence level (LOW → MEDIUM → HIGH based on how many times it has correctly predicted an outcome). These heuristics capture platform-specific knowledge that is not in any runbook — patterns that only emerge from operational experience.

**Phase 4 — Rebuild the Stage 2 injection file.** `output/stage2-heuristics-injection.md` is rewritten with the latest set of active heuristic rules. Stage 2 reads this file at the start of its next run, applying learned rules before even consulting the base graphs.

#### What gets versioned

Every Stage 3 run increments the graph version number and appends a changelog entry to `output/graph-changelog.md`:

```
## v0.1.3 — 2024-01-15 14:32:00

Triggered by: output/incident-report-20240115-143000.md

Entities Added (1): istio_sidecar — observed in logs, not in original graph
Failure Modes Added (2): auto_dns_timeout, auto_pod_eviction_cascade
Heuristics Added (1): rule_7 — if dns_timeout seen, check coredns pod health first
Severity Promoted (1): istio_upstream_reset P3 → P2 (seen 3 times)
```

---

### Stage 4 — Runbook & Troubleshooting Auto-Update

**File:** `prompts/stage4-docs-update.md`
**Run in:** Copilot Agent Mode
**When:** After every Stage 3 run
**Time to run:** 2–4 minutes

#### What it does

Stage 4 closes the loop back into your source documentation. It reads the incident report and writes new entries directly into `input/runbooks/` and `input/troubleshooting/`.

For every new finding, Stage 4 makes a routing decision:

| Finding type | Written to | Why |
|---|---|---|
| Clear fix exists, ordered steps, known trigger | `input/runbooks/` | On-call needs procedure, not investigation |
| Ambiguous cause, needs diagnosis first | `input/troubleshooting/` | Engineer needs a guide, not a script |
| Both apply | One entry in each, cross-referenced | Complete coverage |

#### Safety rules built in

Every section Stage 4 writes carries an AUTO-GENERATED marker:

```html
<!--
AUTO-GENERATED: Stage 4 — incident-report-20240115-143000.md
Status         : DRAFT — requires human review and approval
Failure ID     : istio_upstream_reset
Severity       : P2
Reviewed by    : [ ] platform-team
Approved on    : [ ] <date>
-->
```

**Stage 4 never deletes or overwrites existing content.** New sections are appended at the end of matching files. If an existing section appears incorrect based on the incident evidence, Stage 4 adds a `⚠️ AMENDMENT` note beneath it — the original is preserved.

Every new section also embeds the exact log lines that triggered it, so any engineer reading it later can verify the diagnosis independently.

#### After Stage 4 runs

The `input/` folder now contains DRAFT sections that a human needs to review and approve. Once approved, Stage 1 should be re-run so the graphs reflect the updated, human-vetted documentation.

---

## 6. Files Reference

### Prompt Files (you own these, never modified by the tool)

| File | Purpose | Run in |
|---|---|---|
| `prompts/stage1-graph-builder.md` | Builds knowledge and context graphs from docs | Copilot Agent Mode |
| `prompts/stage2-log-analyzer.md` | Analyzes pasted logs against graphs | Copilot Chat |
| `prompts/stage3-self-improvement.md` | Updates graphs and heuristics from incident report | Copilot Agent Mode |
| `prompts/stage4-docs-update.md` | Writes new runbook and TS guide entries | Copilot Agent Mode |

### Input Files (you populate and maintain these)

| Location | What to put here |
|---|---|
| `input/codebase/` | Your service source code, config files, manifests |
| `input/runbooks/` | Operational runbooks as `.md` or `.txt`, one file per runbook |
| `input/troubleshooting/` | Troubleshooting guides as `.md` or `.txt` |

### Output Files (generated and managed by the tool)

| File | Created by | Used by | What it contains |
|---|---|---|---|
| `output/knowledge-graph.yaml` | Stage 1 | Stage 2, Stage 3 | All entities and relationships |
| `output/context-graph.yaml` | Stage 1 | Stage 2, Stage 3 | Flows, failure modes, remediations |
| `output/incident-report-<ts>.md` | Stage 2 | Stage 3, Stage 4 | Full incident analysis report |
| `output/stage2-heuristics-injection.md` | Stage 3 | Stage 2 (next run) | Active learned heuristic rules |
| `output/prompt-heuristics.md` | Stage 3 | Stage 3 (cross-run) | Full heuristics registry with confidence levels |
| `output/graph-changelog.md` | Stage 3 | Humans | Version history of every graph change |

---

## 7. First-Time Setup (Step by Step)

### Step 1 — Create the folder structure

```
mkdir -p project-root/input/codebase
mkdir -p project-root/input/runbooks
mkdir -p project-root/input/troubleshooting
mkdir -p project-root/output
mkdir -p project-root/prompts
```

### Step 2 — Copy the prompt files

Place the four prompt `.md` files from `api-log-analysis-prompts.md` into `project-root/prompts/`:
- `stage1-graph-builder.md`
- `stage2-log-analyzer.md`
- `stage3-self-improvement.md`
- `stage4-docs-update.md`

### Step 3 — Populate your input folder

Copy your service's source code into `input/codebase/`. It does not need to be the entire monorepo — just the service being monitored.

Copy any existing runbooks into `input/runbooks/`. Name them `runbook-<service>.md`. Even a rough draft is better than nothing — Stage 4 will improve them over time.

Copy any troubleshooting documentation into `input/troubleshooting/`. Name them `ts-<topic>.md`.

If you have no runbooks yet, create a minimal starter file:
```markdown
# Runbook — <Service Name>
## Known Issues
(none yet — will be populated by incident analysis)
```

### Step 4 — Open the project in VS Code

Open `project-root/` as your workspace in VS Code. Copilot Agent Mode needs the workspace open to read and write files.

### Step 5 — Run Stage 1

Open Copilot Agent Mode in VS Code (`Ctrl+Shift+P` → `Copilot: Open Agent`).

Type or paste the Stage 1 prompt from `prompts/stage1-graph-builder.md`.

Watch the output. Stage 1 will:
1. Print a File Inventory table listing every file it reads
2. Extract entities, relationships, flows, failure modes, and remediations
3. Write `output/knowledge-graph.yaml` and `output/context-graph.yaml`
4. Print a Coverage Summary with totals and any gaps found

Check the coverage gaps. If something important is missing, add it to your input files and re-run Stage 1.

**Stage 1 is now complete. The system is ready to analyze logs.**

---

## 8. Day-to-Day Usage (Per Incident)

### When you have logs to analyze

**Step 1 — Open Copilot Chat** in VS Code.

**Step 2 — Attach the Stage 2 prompt file.** In the chat input box, type `#file:` and select `prompts/stage2-log-analyzer.md`. The file name will appear as a reference in your message.

**Step 3 — Paste your raw logs** in the same chat message. These can be:
- Copy-pasted from `kubectl logs`
- Exported from Azure Monitor or Log Analytics
- Copied from an Istio proxy log
- Any plaintext log output

You do not need to clean or format the logs. Paste them as-is.

**Step 4 — Send the message.** Copilot will:
1. Read the instructions from the attached file
2. Read `output/knowledge-graph.yaml` and `output/context-graph.yaml` from your workspace
3. Read any heuristics from `output/stage2-heuristics-injection.md` if it exists
4. Analyze the logs
5. Write the incident report to `output/incident-report-<timestamp>.md`
6. Print a summary in chat

**Step 5 — Run Stage 3** in Copilot Agent Mode immediately after Stage 2 finishes.

Open Copilot Agent Mode and run the Stage 3 prompt from `prompts/stage3-self-improvement.md`. This updates the graphs with what was learned. Do not skip this step — it is what makes the system improve.

**Step 6 — Run Stage 4** in Copilot Agent Mode immediately after Stage 3.

Run the Stage 4 prompt from `prompts/stage4-docs-update.md`. This writes new runbook and troubleshooting entries back to your `input/` files.

**Step 7 — Review the DRAFT sections** added to `input/runbooks/` and `input/troubleshooting/`. See the [Human Review Process](#11-human-review-process) section.

**Step 8 — Run Stage 1 again** after you have approved the DRAFT sections and committed the changes. This rebuilds the graphs from the updated documentation so future analyses reflect what was just learned.

---

## 9. How the Self-Improvement Loop Works

Every time you complete a full cycle (Stage 2 → 3 → 4 → human review → Stage 1), the system becomes more accurate. Here is what happens concretely:

### Run 1 (first incident)

Stage 2 sees logs it has never analyzed before. It matches known failure modes from your original runbooks. Anything it cannot match appears in Section 4 as UNKNOWN FAILURES. Section 7 lists specific suggestions to add to the graphs. Stage 3 applies those suggestions and creates new failure mode entries (marked DRAFT). Stage 4 writes new runbook sections.

**Result:** Graph version goes from `v0.0.0` to `v0.0.1`. A handful of new failure modes added. First heuristic rules created at LOW confidence.

### Run 3 (same failure seen again)

Stage 2 now recognizes the failure that was UNKNOWN in Run 1 — it is in the graph. It matches instantly, produces a higher-confidence hypothesis, and maps to the remediation that Stage 4 wrote. The heuristic rule created in Run 1 has now been confirmed twice and is upgraded to MEDIUM confidence.

**Result:** Time to diagnosis drops from minutes to seconds for known failures.

### Run 5+ (mature system)

Three or more occurrences of the same failure trigger automatic severity promotion. The heuristic file contains platform-specific rules like "if service X returns 503, check coredns health first before checking X itself." Stage 2 applies these before even consulting the base graph — it catches cascading failures earlier.

**Result:** Unknown failure rate approaches zero. MTTD (mean time to detect) measurably shorter.

### What "learned" means in practice

The system learns your platform's specific failure fingerprints. Not generic Kubernetes failure modes — your actual error signatures, your actual blast radius patterns, your actual remediation sequences. This knowledge is stored in versioned YAML files under `output/` and in updated `.md` files under `input/`. It persists across sessions, across team members, and across releases.

---

## 10. Understanding the Output Files

### The Incident Report (`output/incident-report-<timestamp>.md`)

The most important output for an on-call engineer. Open this immediately after Stage 2 runs.

**Section 1 — Executive Summary** is the first thing to read during an incident. It tells you what broke, when, the user-visible impact, and the single most important action to take right now.

**Section 2 — Detected Incidents** gives you one detailed block per distinct incident. The `Matched Failure ID` tells you which entry in `context-graph.yaml` this maps to. `Heuristic Fired` tells you if a learned rule made the match (more confident) or if it was a base graph match. `SLA Breached: YES` means a user-facing SLA was exceeded based on log timestamps.

**Section 3 — Remediation Plan** is the action table for the on-call engineer. Steps are from your own runbooks, sequenced by priority. Always check the Rollback Option before starting any remediation.

**Section 4 — Unknown Failures** is for the post-incident review. These are log patterns the system could not match. After the incident is resolved, these become the inputs for Stage 3 and 4.

**Section 6 — Detection Gap Analysis** is the observability health check. If entities from the knowledge graph appear here as blind spots — meaning they were expected to produce logs but did not — that is a signal to add monitoring before the next incident.

**Section 7 — Graph Feedback** is the instruction set for Stage 3. It lists every improvement to apply. Stage 3 reads this section and executes it automatically.

### The Graph Changelog (`output/graph-changelog.md`)

A running history of every change made to the graphs. Useful for:
- Understanding what the system knows and when it learned it
- Reverting a bad auto-generated entry (find it by version, remove from the YAML)
- Demonstrating to management that the system is improving over time

### The Knowledge Graph (`output/knowledge-graph.yaml`)

A YAML representation of your system's architecture as the tool understands it. Review this after Stage 1 to verify entity extraction was correct. If you see missing services or wrong relationships, add documentation to `input/` and re-run Stage 1. Do not edit this file manually — it will be overwritten.

---

## 11. Human Review Process

Stage 4 marks all auto-generated documentation as DRAFT. A human engineer must review and approve each section before it is trusted.

### Finding DRAFT sections

After Stage 4 runs, look for the AUTO-GENERATED comment block in `input/runbooks/` and `input/troubleshooting/`:

```html
<!--
AUTO-GENERATED: Stage 4 — incident-report-20240115-143000.md
Status         : DRAFT — requires human review and approval
Failure ID     : istio_upstream_reset
Severity       : P2
Reviewed by    : [ ] platform-team
Approved on    : [ ] <date>
-->
```

### How to approve

1. Read the section carefully. Verify the symptoms, diagnostic steps, and remediation commands match your actual system.
2. Edit any steps that are incorrect or incomplete.
3. Update the severity if it does not match your experience with this issue.
4. Remove the entire `<!-- AUTO-GENERATED ... -->` comment block.
5. Fill in the `Reviewed by` and `Approved on` fields (outside the comment).
6. Commit to the repository.

### How to reject

If the auto-generated section is completely wrong or not useful:
1. Delete the entire section from the file.
2. Note the `failure_id` and manually add the correct content if needed.
3. Commit.

### When to run Stage 1 after review

Run Stage 1 only after approval and commit. This ensures the graphs are rebuilt from human-vetted content. If you run Stage 1 before review, the graphs will incorporate unverified content.

---

## 12. Tips and Best Practices

**Run one API at a time through Stage 1.** If you have multiple APIs (QOD, NPM, MTAD, etc.), run Stage 1 separately for each with its own `output/` folder. This keeps entity resolution clean and prevents cross-contamination between services. You can merge graphs later once each is mature.

**The quality of Stage 2 is directly proportional to the quality of Stage 1's inputs.** Vague runbooks produce vague graphs. If Stage 2 is producing too many UNKNOWN FAILURES on the first run, add more specific error signatures and remediation steps to your runbooks and re-run Stage 1.

**Do not skip Stage 3.** It is tempting to skip Stage 3 when under incident pressure. Run it after the incident is resolved. Every skipped Stage 3 run is a missed learning opportunity.

**Paste logs in full, without trimming.** Stage 2 needs the full log context to reconstruct request chains via trace IDs. Trimmed logs lose the timeline and the correlation context.

**Use the Detection Gap Analysis section proactively.** If Stage 2 consistently lists the same entity as a blind spot across multiple runs, add logging or monitoring to that service before the next incident.

**Review the graph-changelog regularly.** After 10 or more runs, review the changelog to see which failures are recurring most often. These are your highest-priority reliability investments.

**For new team members:** The incident reports and graph changelog together tell the story of every production incident. New engineers can read them to understand the system's failure history before they ever handle an on-call shift.

---

## 13. Troubleshooting the Tool Itself

**Stage 1 says it cannot find files in `input/`**
Confirm you have opened `project-root/` as the workspace root in VS Code, not a subfolder. Copilot Agent Mode reads files relative to the workspace root.

**Stage 2 says "No logs found in this message"**
Make sure your log content is in the chat message itself, not in a separate attached file. The stage 2 `.md` file is the instruction — your typed/pasted message content is the log data.

**Stage 2 is producing only UNKNOWN FAILURES**
Your `input/runbooks/` and `input/troubleshooting/` files may not contain enough specific error signatures. Add exact log error strings from past incidents to your troubleshooting guides and re-run Stage 1.

**Stage 3 or 4 is not writing files**
Verify that Copilot has write access to the workspace. In some configurations, Agent Mode needs to be explicitly allowed to create files. Check the Copilot Agent permissions in VS Code settings.

**The graphs seem out of date after a code change**
Run Stage 1 again. The graphs are static snapshots — they do not auto-update when code changes. The recommended trigger is any release or significant runbook update.

**Stage 4 is writing content to the wrong file**
Stage 4 uses entity IDs from the knowledge graph to match new findings to existing files. If entity IDs are wrong or missing, the matching will be incorrect. Review `output/knowledge-graph.yaml` for the affected entity and correct the source documentation in `input/`, then re-run Stage 1.

---

## 14. Quick Reference Card

### What to run and when

| Situation | Action |
|---|---|
| First time setup | Run Stage 1 |
| New release deployed | Run Stage 1 |
| Runbook updated | Run Stage 1 |
| Got logs to analyze | Stage 2 in Copilot Chat |
| After every Stage 2 | Run Stage 3, then Stage 4 |
| After Stage 4 | Review DRAFTs → Approve → Run Stage 1 |

### Stage 2 — How to run in Copilot Chat

```
1. Open Copilot Chat in VS Code
2. Type:  #file:prompts/stage2-log-analyzer.md
3. Paste your raw logs in the same message
4. Send
```

### Stages 1, 3, 4 — How to run in Copilot Agent Mode

```
1. Open Copilot Agent Mode  (Ctrl+Shift+P → "Copilot: Open Agent")
2. Paste the content of the relevant stage .md file
3. Send
```

### Output files location

```
output/knowledge-graph.yaml           ← system architecture
output/context-graph.yaml             ← failure modes + remediations
output/incident-report-<ts>.md        ← latest incident analysis
output/graph-changelog.md             ← version history
output/stage2-heuristics-injection.md ← learned rules (auto-managed)
```

### Per-incident full sequence

```
1. Stage 2  (Copilot Chat)   — paste logs → get incident report
2. Stage 3  (Copilot Agent)  — update graphs + heuristics
3. Stage 4  (Copilot Agent)  — update runbooks + TS guides
4. Human review              — approve DRAFT sections, commit
5. Stage 1  (Copilot Agent)  — rebuild graphs from updated docs
```

---

*This tool runs entirely within GitHub Copilot using your local workspace files. No external services, no data leaves your environment.*
