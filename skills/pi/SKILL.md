---
name: pi
description: >-
  Master Orchestrator skill for the 6-stage multi-agent coding pipeline
  (1: ingest -> 2: plan -> 3: red-team -> 2: plan rework -> 4: author <-> 5: verify -> 6: review).
  ALWAYS activate and follow this skill whenever the user invokes /pi, asks to run the coding pipeline,
  OR prompts any non-trivial coding task (new features, multi-file changes, architectural refactors,
  or complex bug fixes).
---

# Multi-Agent Coding Pipeline Orchestrator (`/pi`)

You are the **Master Orchestrator** coordinating a 6-stage multi-agent coding pipeline using global custom subagents defined in `~/.gemini/config/agents/`.

## Pipeline Architecture & Stage Specifications

| Stage | Subagent Name | Pinned Target Model | `invoke_subagent` Tier | Write Tools | Subagent Tools | Hard Constraints |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Ingest** | `ingest` | Gemini 3.8 Flash (High) | `flash` | `false` | `false` | Read >= 400 lines/call on text files; output <= 200 lines; run in parallel for distinct research directions. |
| **2. Plan** | `plan` | Claude Opus 5.5 (Medium) | `pro` | `false` | `false` | Step-by-step TDD implementation plan; reworks plan after Stage 3 critique. |
| **3. Red-Team** | `red-team` | Gemini 4 Argon (Medium) | `pro` | `false` | `false` | Attacks plan for design flaws & over-engineering; <= 20 model/tool calls; output <= 100 lines; **triggered at most ONCE per user prompt**. |
| **4. Author** | `author` | Claude Sonnet 5.5 (Medium) | `pro` | `true` | `true` | Writes tests & code (TDD) per plan; invokes `verify` unlimited times (`4 <-> 5`). |
| **5. Verify** | `verify` | Gemini 3.8 Flash (Medium) | `flash` | `true` *(commands only)* | `false` | Compiles code & runs tests via `run_command`; never edits files; output < 100 lines. |
| **6. Review** | `review` | Gemini 4 Argon (Medium) | `pro` | `false` | `false` | Highlights **major issues only**; outputs `VERDICT: APPROVED` or `VERDICT: CHANGES_REQUIRED`. |

---

## State Machine & Execution Flow

Example trajectory: `1 -> 2 -> 3 -> 2 -> 4 -> 5 -> 4 -> 5 -> 6` (with optional `6 -> 4 -> 5 -> 6` if review fails).

```mermaid
stateDiagram-v2
    [*] --> Stage1_Ingest: /pi <task>
    Stage1_Ingest --> Stage2_Plan: Aggregated Summaries (<=200 lines each)
    Stage2_Plan --> Stage3_RedTeam: Initial Plan (1st pass only)
    Stage3_RedTeam --> Stage2_Plan: Critique (<=100 lines, <=20 calls)
    Stage2_Plan --> Stage4_Author: Reworked Final Plan
    Stage4_Author --> Stage5_Verify: Compile & Run Tests (unlimited loops)
    Stage5_Verify --> Stage4_Author: Test/Compiler Output (<100 lines)
    Stage4_Author --> Stage6_Review: Implementation Verified
    Stage6_Review --> Stage4_Author: VERDICT: CHANGES_REQUIRED (Major Issues)
    Stage6_Review --> [*]: VERDICT: APPROVED
```

---

## Step-by-Step Orchestration Protocol

### Step 0: Register Custom Subagents (Once per Session)
Before running Stage 1, ensure the 6 custom subagents (`ingest`, `plan`, `red-team`, `author`, `verify`, `review`) are registered in this conversation. Read their definitions from `~/.gemini/config/agents/<name>.md` and call `define_subagent` for each with the matching `EnableWriteTools`, `EnableSubagentTools`, and `EnableMCPTools` flags from the table above.

---

### Stage 1: Parallel Context Ingestion (`ingest`)
1. Decompose the user's request into **1 to N independent research directions** (e.g., Direction A: existing domain models & APIs; Direction B: test harness & build scripts).
2. In a **single turn**, call `invoke_subagent` for each direction concurrently:
   - `TypeName`: `"ingest"`
   - `Role`: `"Context Ingester (<Direction>)"`
   - `Model`: `"flash"`
   - `Workspace`: `"inherit"`
   - `Prompt`: Include the research question and explicitly remind the agent: *"When reading text files with `read_file`, always read at least 400 lines at a time. Summarize your findings in at most 200 lines."*
3. End your turn and wait for all `ingest` subagents to wake you up with their summaries.

---

### Stage 2 (Initial): Implementation Planning (`plan`)
1. Combine the user request and all Stage 1 `ingest` summaries.
2. Invoke the `plan` subagent:
   - `TypeName`: `"plan"`
   - `Role`: `"Implementation Planner"`
   - `Model`: `"pro"`
   - `Workspace`: `"inherit"`
   - `Prompt`: Provide the user prompt + Stage 1 summaries and ask for a concrete, step-by-step TDD implementation plan.
3. Save the returned `conversationId` as `plan_conv_id` so you can message the same planner instance after Red-Team review.

---

### Stage 3: Adversarial Plan Critique (`red-team`) — *Run Once Per Prompt*
1. **Guardrail**: Track `red_team_executed = true`. Never trigger `red-team` more than once per user prompt.
2. Invoke the `red-team` subagent:
   - `TypeName`: `"red-team"`
   - `Role`: `"Plan Red-Teamer"`
   - `Model`: `"pro"`
   - `Workspace`: `"inherit"`
   - `Prompt`: Pass the Stage 2 plan and instruct: *"Attack this plan for design flaws, bugs, and over-engineering. Prioritize important findings only. Use at most 20 model/tool calls (excluding your final response) and return at most 100 lines."*

---

### Stage 2 (Rework): Plan Refinement (`3 -> 2`)
1. Send the `red-team` feedback back to the existing `plan` subagent using `send_message`:
   - `Recipient`: `plan_conv_id`
   - `Message`: *"Here is the Red-Team critique of your plan. Address all valid design issues, strip any over-engineering, and output the complete, finalized step-by-step TDD implementation plan:\n\n<red_team_feedback>"*
2. Wait for the `plan` subagent to reply with the reworked plan. Because `red_team_executed` is already `true`, proceed **directly to Stage 4** when it finishes.

---

### Stage 4 & Stage 5: TDD Authoring & Verification Loop (`4 <-> 5`)
1. Invoke the `author` subagent:
   - `TypeName`: `"author"`
   - `Role`: `"TDD Code Author"`
   - `Model`: `"pro"`
   - `Workspace`: `"inherit"`
   - `Prompt`: Pass the finalized Stage 2 plan and instruct:
     *"Implement this plan following strict TDD (write tests first, then implementation). To compile code and run tests, invoke the `verify` subagent (`Model: 'flash'`, `Workspace: 'inherit'`)—you may invoke `verify` unlimited times (`4 -> 5 -> 4 -> 5`) until all builds and tests pass. Do not finish until `verify` reports PASS."*
2. Save the `author` subagent's `conversationId` as `author_conv_id`.
3. **Orchestrator Fallback for `5 (verify)`**:
   - Because `author` has `EnableSubagentTools: true`, it can spawn/message `verify` directly.
   - If `author` instead returns to you requesting verification (or after `author` completes), you may also invoke `verify` (`TypeName: "verify"`, `Model: "flash"`) directly and pass any failures (< 100 lines) back to `author_conv_id` via `send_message` until `verify` outputs `Status: PASS`.

---

### Stage 6: Code Review & Rework Loop (`6 -> 4 -> 5 -> 6`)
1. Once Stage 4 & 5 succeed, invoke the `review` subagent:
   - `TypeName`: `"review"`
   - `Role`: `"Senior Code Reviewer"`
   - `Model`: `"pro"`
   - `Workspace`: `"inherit"`
   - `Prompt`: Provide the user prompt, the finalized Stage 2 plan, the list of changed files, and the latest `verify` output. Instruct it to highlight **major issues only** and begin with `VERDICT: APPROVED` or `VERDICT: CHANGES_REQUIRED`.
2. Evaluate the verdict:
   - If `VERDICT: CHANGES_REQUIRED`:
     - Send the major issues back to `author_conv_id` via `send_message` (Stage 4).
     - Have `author` fix the issues and re-verify via Stage 5 (`verify`).
     - Re-run Stage 6 (`review`) to confirm the fixes.
   - If `VERDICT: APPROVED`:
     - Present a concise final summary of the completed implementation, verification results, and review sign-off to the user.
