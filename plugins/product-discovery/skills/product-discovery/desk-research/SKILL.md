---
name: desk-research
description: "Cited secondary research via two parallel sub-agents (competitive scan + scholar scan), both forced to surface disconfirming evidence. Use when the user wants desk research or a literature review, a competitive analysis of how others solved a problem, or asks 'is there evidence for' a hypothesis."
metadata:
  ready_to_publish: "true"
---

# Desk Research

Frame the question with the user, design queries together, dispatch two parallel sub-agents (competitive + scholar), synthesize, and produce a report with mandatory disconfirming evidence.

Respond and produce all artifacts (including the report) in the user's conversation language.

This skill runs on live web search. Confirm web search tools are available before proceeding; if they are not, tell the user and stop - the report must come from live sources, not memory.

## Reference load map

Load each file only at the stage indicated:

| Stage | Load |
|-------|------|
| 0 Setup | `assets/state-template.json` |
| 3 Dispatch | `assets/shared-subagent-base.md` + `assets/competitive-subagent-prompt.md` / `assets/scholar-subagent-prompt.md` |
| 4 Synthesis | `references/quality-rubric.md` (re-check confidence, synthesis checks) |
| 5 Report | `references/report-template.md` |

## Hard rules (always active)

1. **Confront the hypothesis, don't confirm it.** At least 1/3 of the report must address disconfirming evidence. This is the heart of the skill.
2. **Every finding needs a URL.** No URL = not a finding.
3. **Minimum 3 independent sources for High confidence.** Three articles citing the same report = 1 source.
4. **Vendor sources flagged, not hidden.** Confidence capped at Low, tagged `[VENDOR]` in bibliography.
5. **Never modify `ost-state.json`.**

Full scoring criteria live in `references/quality-rubric.md`.

## Lazy user mode

If the user signals disengagement ("skip", "", "." or equivalent): auto-decide, state your pick and why in one sentence, mark `auto_decided: true` in state, continue. Exception: if the disconfirmation prompt (1.4) is skipped, mark `disconfirmation_skipped: true` and strengthen the sub-agents' falsification mandate.

## Workflow

```
Setup → Brief gate → Query design gate → Sub-agent dispatch (||) → Synthesis gate → Report
```

If invoked on existing state, jump to the stage in `current_stage` (see Resume routing).

## Stage 0 - Setup

1. **Detect state.** Look for `desk-research-state-*.json` in the project folder or `research/`. If found, offer: resume / jump to stage / start new (archive old).
2. **Ingest context.** Scan the project folder: `brief.md` and `*-brief.md`; first 80 lines of every `.md` in `research/` and `analyses/`; `ost-state.json` read-only (outcome statement + top 2-3 opportunities by PRES); note `*.csv`/`*.xlsx` filenames without loading them. Extract `industry_or_domain` from the brief; if the folder is empty, ask directly.
3. **Confirm scope** (single-choice): single problem (default) / broader literature scan (scale=deep) / competitive only / scholar only. Initialize state from `assets/state-template.json`.

## Stage 1 - Research brief gate

1. From context, formulate: **problem statement** (one falsifiable sentence), **working hypothesis** (what the user already assumes, often unstated), **decision at stake** (what gets approved/rejected). Ask directly for anything you can't extract - don't invent.
2. Propose 3-5 falsifiable research questions ("Does X correlate with Y?", "Under what conditions..."). Show a numbered list; loop with the user until approved.
3. Set scale from question count: 1-3 → quick (~5-7 sources per sub-agent, 1-2 page report); 4-5 → deep (~15+ sources, full report). Tell the user explicitly, offer override.
4. **Disconfirmation prompt (soft gate).** Ask what evidence would prove the hypothesis WRONG. Accept any answer; if skipped, mark `disconfirmation_skipped: true`.
5. Show the full brief (problem, hypothesis, decision, questions, scale, falsification criterion). Gate: OK / edit / stop and discuss.

## Stage 2 - Query design gate

1. **Competitive queries.** Per research question: 5-10 companies/cases (leaders + niche + adjacent industry), 3-5 English search keywords, recency window (default 5 years), source heuristics (prioritize trade press, analyst reports, case studies; flag vendor whitepapers).
2. **Scholar queries.** Map the user's business language to academic language. Show the mapping table (`USER LANGUAGE → ACADEMIC LANGUAGE`, 1-2 query strings each) - this is where users often add a key term.
3. Confirm all queries with the user (edit/add/remove), save the final set to state.

## Stage 3 - Sub-agent dispatch (parallel when available)

Dispatch both scans with the host's sub-agent or delegation capability. Start them in parallel when the host supports parallel delegation. In Claude Code, a general-purpose `Agent` is suitable; in Codex, use the available sub-agent/collaboration mechanism. Do not depend on one provider's tool name. If the host has no delegation capability, run the scans sequentially in the main context and keep their findings in separate files.

- **A - Competitive scan:** prompt = `assets/shared-subagent-base.md` + `assets/competitive-subagent-prompt.md`.
- **B - Scholar scan:** prompt = `assets/shared-subagent-base.md` + `assets/scholar-subagent-prompt.md`.

Substitute every `{{placeholder}}` from state (placeholder names match state fields). Computed: `{{recency_start_year}}`, `{{current_year}}`. Special: `{{disconfirmation_criterion_OR_skipped_message}}` - the criterion verbatim, or if skipped: "User skipped the falsification criterion. Your mandate for the disconfirming section is stronger: minimum 4 disconfirming findings at confidence Medium+."

Each sub-agent saves findings to `{project_path}/research/.desk-tmp/<stream>-findings-<timestamp>.md` and returns a 200-word summary.

Failure handling: competitive empty → queries too narrow, back to 2.1. Scholar empty → may be genuinely under-researched (a legitimate finding), flag in Gaps. Both empty → ask the user: bad queries / under-researched / network issue?

## Stage 4 - Synthesis gate

1. Read both findings files in full.
2. Cluster evidence per research question and confront it with the hypothesis (supports/refutes/mixed); collect the strongest quotes and numbers.
3. Surface cross-source contradictions (competitive vs scholar disagreements) - the most valuable synthesis fragments. Patterns in `references/quality-rubric.md` (Synthesis checks).
4. **Disconfirming evidence.** Dedicate a section to everything contradicting the hypothesis. If fewer than 3 points at confidence Medium+, send sub-agents back for more.
5. Show the user the strongest conclusion + strongest counter-evidence. Gate: write report / more counter-evidence / more on topic X / stop and discuss.

## Stage 5 - Report

1. Generate following `references/report-template.md` exactly.
2. Save to `<project>/research/<YYYY-MM-DD>-desk-<short-topic>.md`. Set state to `complete`. Clean up `.desk-tmp/` unless the user wants full logs.
3. Wrap up: report link, one-line TL;DR, strongest counter-evidence to watch, next steps.
4. The report's evidence is ready-made source material for an Opportunity Solution Tree - when the user's goal is structuring discovery, suggest running the ost-builder skill on it.

## Quality gates - stop and fix

- Empty or missing disconfirming section → back to 4.4.
- Bibliography >50% vendor sources → back to 2.1.
- "Supported" with <3 independent High-confidence sources → note the limitation in the TL;DR.
- Sub-agent citations without URLs → send back to fix.
- Disconfirmation skipped AND disconfirming section <3 points → escalate to the user.
- >50% of competitive sources older than 5 years → flag recency drift.

## Iteration mode

When the user returns to an existing report wanting more: load state + report, repeat the relevant stage (typically 3 with new queries, or 2 with better mappings), dispatch sub-agents with differential queries only, append `## Iteration [date]` before the bibliography. If the disposition changes: keep the original TL;DR as a blockquote, write a new one, update frontmatter.

## Resume routing

| `current_stage` | Action |
|---|---|
| `setup` | Start from 0.1 |
| `brief` | Show existing brief for approval/edit |
| `queries` | Show existing queries for approval |
| `dispatch` | Both complete → Stage 4; one complete → re-dispatch missing; pending/failed → re-dispatch; in_progress → ask: re-run or recover from `.desk-tmp/`? |
| `synthesis` | Stage 4 with existing findings |
| `report` | Stage 5.1 (synthesis approved, just needs the file) |
| `complete` | Offer iteration mode |

## Style and file conventions

- One question at a time, max 4 options + open-ended. Always show citations with suggestions.
- Push back on thin evidence; be honest about gaps. Mirror the user's vocabulary from their project files.
- State: `<project>/research/desk-research-state.json`. Temp: `<project>/research/.desk-tmp/`. Report: `<project>/research/<YYYY-MM-DD>-desk-<short-topic>.md`. Never write to `4-Knowledge/`.
