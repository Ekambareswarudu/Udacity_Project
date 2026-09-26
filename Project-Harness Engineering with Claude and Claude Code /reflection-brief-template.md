# Reflection Brief — Harness Engineering Capstone

**Name**: Ekambareswarudu Vutla
**Date**: 2026-09-25

**Environment**
Model(s): claude-haiku-4-5-20251001
OS / Python: Linux workspace environment; Python 3.13 virtual environments.
Approx. API spend: System 1 reported $0.1151 USD estimated total cost. System 2 used the Claude API for context compression/evaluation; its budget.json records the token usage but not a total dollar spend.
Part 1 — Per-system
**System 1 — Agentic loop**

**1. Loop control**
In evidence/system1_agentic_loop/claim_04_neighbor_injury.jsonl, the stop_reason sequence is tool_use → tool_use → tool_use → tool_use → end_turn. The first four turns perform policy lookup, fact recording, classification/severity assessment, and routing; the fifth turn has no tool calls and ends with end_turn. The loop control is implemented in claims_intake/loop.py: it continues when stop_reason == "tool_use", returns when stop_reason == "end_turn", and raises an error for unexpected values.

**2. Anti-pattern**

One anti-pattern checked by tests/test_antipatterns.py is using an integer-literal iteration cap such as for _ in range(5) or while ... < 5 as the primary stopping mechanism. The test requires a budget sourced from configuration instead of a hard-coded iteration limit. A fixed iteration cap could stop a claim before all required tool calls were completed, resulting in incomplete processing. The same test file also verifies that stop_reason is referenced as the loop-breaking signal.

**3. Tool design**

Two tools with overlapping inputs are classify_claim and request_clarification, because both deal with determining the claim type. Their descriptions separate their responsibilities: classify_claim commits to a claim type with a confidence score and rationale, while request_clarification is used only when the claim type is genuinely ambiguous between two or more types. The tool executor also returns structured errors with categories such as "permanent" or "transient", a retryable flag, and a specific message. This lets the agent distinguish a recoverable problem from a permanent validation or sequencing error instead of treating every failure as a generic string.

**4. Your numbers**

For claim_04_neighbor_injury, the actual run took 5 turns and had an estimated cost of $0.0226, ending with a routed outcome. The README's reference run states 7 routed and 1 escalated across eight fixtures, with an estimated cost of approximately $0.05 on Haiku 4.5. My run also processed all eight fixtures, but the model's runtime trajectory and cost differed from the reference. The System 1 test suite passed all 29 tests, confirming the required behaviors even though the exact model path varied.

**System 2 — Context strategy**

**5. Reduction**

budget.json reports a baseline of 38,708 tokens and an assembled context of 16,946 tokens, giving a 56.22% reduction. The active section dominates the assembled context at 15,789 tokens, compared with 204 for case_facts, 421 for resolved_refund, and 550 for resolved_subscription. The active section is kept largely verbatim because it represents the current unresolved conversation state needed for the copilot's immediate task.

**6. Summarize vs preserve**

The context strategy summarizes older resolved information while preserving information that needs to remain available for the current task. The budget shows resolved_refund at 421 tokens, resolved_subscription at 550, case_facts at 204, and active at 15,789. The compression API reduced the refund section from 12,334 input tokens to 408 output tokens and the subscription section from 11,475 input tokens to 537 output tokens. The approach is to compress resolved historical material while preserving active information required for the next interaction.

**7. Facts block**

The normal eval.jsonl evaluation passed all six questions. In eval_control.jsonl, Q1 unexpectedly passed because the refund answer remained in context, while Q6 failed as expected because the structured status information was absent after the case-facts content was stripped. This demonstrates that context reduction needs explicit preservation rules and control evaluations. A successful answer alone does not prove that the intended context structure is responsible for the answer.

**System 3 — Claude Code config**
**8. Path-scoped rules**

The React rule contains the following path globs:

paths:
  - "src/components/**/*"
  - "src/pages/**/*"

These rules load when editing files under the specified React component and page paths. This is preferable to putting React-specific conventions in a broad directory-level CLAUDE.md because the instructions are applied only where they are relevant. It prevents React-specific rules from unnecessarily affecting unrelated parts of the monorepo.

**9. Forked skill**

**The deploy-check skill specifies:**

context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git status:*)
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(git rev-parse:*)
  - Bash(git ls-files:*)
  - Bash(gh pr view:*)
  - Bash(gh pr checks:*)

The skill is explicitly read-only and runs in a fork so verbose repository-discovery output stays out of the main session. Only the structured pass/fail summary is returned to the calling session. The restricted tool list prevents the skill from modifying files, pushing changes, or deploying.

**10. Scope**

The validator output was OK, confirming that the project-level configuration passed validation. Project-level scope is represented by repository files such as CLAUDE.md, .claude/rules/, .claude/commands/, and .claude/skills/. User-level scope would instead be personal Claude Code configuration outside the project repository. The project therefore demonstrates configuration that is committed with the codebase and shared with the team.

**System 4 — Orchestration**

**11. Push work down**

The warm tier retrieves recent defects using the defects_since query:

SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?

The database defines idx_defects_ts ON defects(ts) and idx_defects_shift_ts ON defects(shift, ts). This allows the database to filter recent defects before they are passed into the orchestration/model layer instead of sending the complete historical database to the model. In my shift run, the resulting information was summarized into a compact shift-level result rather than exposing the full history.

**12. Crash recovery**

The recovery logic in shift_monitor/recovery.py uses a 30-minute staleness threshold. If there are no previous steps or the previous state is already complete, the decision is fresh; otherwise, the system chooses resume when the last step is no more than 30 minutes old. If the last step is older than 30 minutes, it chooses fresh. Starting fresh with the findings already captured in the manifest injected as a summary avoids continuing from stale partial state while retaining useful information.

**13. Small state**

The recorded hot_state.json size was 643 bytes. Keeping hot state small matters because the monitoring system runs repeatedly across shifts, so state size can accumulate over time. A bounded state reduces the amount of information that must be loaded and processed on each shift. The pipeline explicitly trims state to the configured byte budget before writing it.

**Part 2 — Synthesis**
**14. Three layers**

The Model layer is represented by artifacts such as the System 1 system prompt and Claude's tool decisions. The Harness layer is represented by deterministic control code such as claims_intake/loop.py, the System 2 context assembly, and the System 3 Claude Code configuration. The Orchestration layer is represented by the System 4 pipeline, warm database, hot state, scratchpad, and shift execution. Together these layers separate model reasoning from deterministic execution and longer-running state management.

**15. Deterministic vs prompt**

A code-guaranteed behavior is the System 1 loop rule that continues on stop_reason == "tool_use" and terminates on end_turn. This belongs in code because execution control must remain deterministic. Prompt-guided behavior is more appropriate for semantic decisions such as whether a claim needs clarification or how the model should classify a claim. The general principle is to enforce structural and safety invariants in code while using prompts to guide model reasoning.

**16. Context, two faces**

System 2 handled context within a session by reducing 38,708 tokens to 16,946 tokens, a 56.22% reduction, while preserving the active conversation. System 4 handles context across shifts through persisted state, including the 643-byte hot_state.json. Both systems apply the same principle of retaining information needed for the next decision while avoiding unnecessary historical context. System 2 achieves this through context assembly and compression, while System 4 achieves it through bounded persisted operational state.

**17. Reliability you can't see in one run**

A single successful run cannot prove that failure modes are handled correctly. System 1's anti-pattern tests verify that execution is controlled by stop_reason rather than string matching or a hard-coded iteration limit. System 2's control evaluation deliberately removes information and checks whether answers fail when the required context is unavailable. These tests reveal reliability properties that would not necessarily be visible in one successful execution.

**18. Blast radius**

System 4 limits blast radius through its bounded hot-state mechanism. The pipeline calls _trim_to_budget() and removes active alerts while the serialized state exceeds HOT_STATE_BYTE_BUDGET, then persists the result using write_atomic(). This keeps state growth bounded and prevents unexpectedly large state from propagating indefinitely across shifts. Because the limit is enforced in code, it does not depend solely on the model following a prompt instruction.

**Part 3 — Honest assessment**
**19. What broke**

One first-try issue occurred during System 2 when the API credentials were not correctly configured, causing the initial API request to fail. After correcting the API configuration, System 2 completed successfully and produced the expected budget and evaluation artifacts. During evidence collection, I also discovered that the first evidence ZIP had been created before all required evidence files were copied into the final evidence tree. These issues showed that both environment configuration and final evidence validation need to be checked explicitly.

**20. What you'd change**

I would make final evidence validation an explicit stage before creating the deliverable ZIP. The project guide requires specific artifacts for each system, including pytest logs and system-specific evidence. Checking the complete evidence tree against that requirement before zipping would have prevented missing files from being discovered afterward. The main implementation work was successful, but the evidence-packaging process could be made more deterministic and less error-prone.
