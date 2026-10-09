# Reflection Brief — Harness Engineering Capstone

## Environment

* Name: REDDIPALLI SAI CHARISH

* Date: October 9, 2026

* System 1 model: `claude-haiku-4-5-20251001`

* Environment: Linux, Python 3.13.0

* System 4: Recorded-response/offline mode

## Part 1 — Per-System Reflections

### System 1 — Agentic Loop

Q1. How does your loop know when to stop?

The `run()` function in `claims_intake/loop.py` checks `response.stop_reason`. When it is `tool_use`, the harness executes the requested tools and continues. When it is `end_turn`, the loop returns the final state. The `claim_01_kitchen_fire` trace shows 2 turns, ending with `end_turn`. Evidence: `summary.md`, the `claim_01_kitchen_fire` trace, and `test_stop_reason_is_loop_control`.

Q2. Why is a hardcoded iteration limit an anti-pattern?

A fixed limit such as `for _ in range(3)` can terminate a task before the model finishes its tool calls. The run summary shows `claim_01_kitchen_fire` took 2 turns, while `claim_03_water_damage` took 4 turns. This variation demonstrates why the loop should follow `stop_reason` instead of an arbitrary cap. Evidence: `test_antipatterns.py`, `test_no_integer_literal_iteration_cap_in_loop`, and `summary.md`.

Q3. How do your tool descriptions help the model?

The `route_to_adjuster` and `escalate_to_human` tools distinguish automatic routing from human escalation. The guidance uses a confidence threshold of 0.6: higher-confidence claims can be routed, while lower-confidence claims should be escalated. The `_t_lookup_policy` tool returns structured error fields such as `is_error`, `error_category`, and `is_retryable`, helping distinguish retryable failures from other errors. Evidence: System 1 tool definitions and tests.

Q4. What were the actual run metrics?

According to `summary.md`:

* `claim_01_kitchen_fire`: 2 turns, 6,372 input tokens, $0.009.

* `claim_03_water_damage`: 4 turns, $0.0193, with one clarification.

These are the submitted run's figures and replace the earlier incorrect estimates.

### System 2 — Context Strategy

Q5. How much context did you reduce?

`budget.json` reports a baseline of 38,708 tokens and assembled context of 16,794 tokens, a 56.61% reduction. The active section remains verbatim at 15,789 tokens because it contains information needed for the current decision.

Q6. What did you summarize and what did you preserve?

Resolved history was compressed while the active issue was preserved:

* Refund history: 12,334 → 392 tokens.

* Subscription history: 11,475 → 401 tokens.

* Active section: 15,789 tokens, kept verbatim.

Evidence: System 2 `budget.json`.

Q7. What happens when the persistent facts block is removed?

The full evaluation, `eval.jsonl`, reports 6/6 passing. Q1 returns the refund amount 22.14, and Q6 returns `payment_update_status: in_progress`.

The submitted `eval_control.jsonl` was only a single byte and contained no usable evaluation results. Therefore, the Q6 regression is not yet verified by the submitted evidence. To complete this answer, rerun the control variant with the persistent facts block removed, then compare its generated results against the full evaluation.

### System 3 — Claude Code Configuration

Q8. Why use path-scoped rules?

`.claude/rules/tests.md` scopes its rules to `**/*.test.tsx` and `**/*.test.ts`. This applies testing guidance to matching files without repeating it in every directory. Evidence: `.claude/rules/tests.md` and the System 3 test output.

Q9. What does the forked deploy-check skill do?

The deploy-check skill sets `context: fork` and limits `allowed-tools` to reading/searching files and selected Git/GitHub inspection commands. This keeps intermediate investigation separate from the main conversation and restricts the tools available to the skill. Evidence: the deploy-check skill configuration.

Q10. What is the difference between project-level and user-level configuration?

Project-level files such as `./CLAUDE.md` and `.claude/rules/*.md` can be versioned and shared with the team. A user-level skill such as `~/.claude/skills/deploy-check-strict/` belongs to an individual developer's environment. Evidence: System 3 configuration files and `validator_output.txt`.

### System 4 — Orchestration

Q11. How does the SQL-filtered defect slice work?

The updated `shift-output.txt` now confirms `shift C: 17 new defects`. The updated `shift_scratchpad.jsonl` records the analysis window as starting at `2026-04-01T00:00:00Z`. It describes 3 high- and 2 medium-severity defects on capacitor bank C-7, associated with lot `2026-0430-B`, plus one repeat low-severity VP-4 vent squeal. The recommendation was to quarantine the affected lot.

`shift_monitor/warm.py` performs the filtered retrieval in SQL:

SQL

```
SELECT * FROM defects
WHERE ts > ?
ORDER BY ts DESC
LIMIT ?
```

Evidence: `shift-output.txt`, `shift_scratchpad.jsonl`, and `shift_monitor/warm.py`.

Q12. How does crash recovery decide whether to resume?

`recovery.py` defines `STALE_RESUME_THRESHOLD_MINUTES = 30`. The `decide()` function resumes incomplete work if it is recent enough; stale work starts fresh. The 30- and 31-minute truth-table tests verify the boundary. Evidence: `recovery.py` and `pytest_S4.log`.

Q13. Why keep hot state small?

`data/hot_state.json` measures 643 bytes, according to `hot_state_size.txt`, well below the approximately 5 KB budget. The test `test_hotstate_rejects_more_than_20_hashes` checks the retained-hash limit. A small hot state avoids carrying the entire defect history into each shift. Evidence: `hot_state_size.txt` and System 4 tests.

Q14. How does a forked investigation stay isolated?

`shift_monitor/fork.py` creates a separate working state for each hypothesis. Each fork has its own scratchpad and can investigate without changing the base `hot_state.json`. The `merge_findings` operation brings results back by appending findings to the main scratchpad rather than replacing the base state with the fork's copy. This isolates exploratory work while allowing useful findings to be shared. Evidence: `shift_monitor/fork.py` and the System 4 tests.

## Part 2 — Cross-System Synthesis

Q15. Where are the Model, Harness, and Orchestration layers?

* Model: System 1 uses `claude-haiku-4-5-20251001`, recorded in `summary.md`.

* Harness: `claims_intake/loop.py` interprets `stop_reason` and executes tool calls.

* Orchestration: System 4 uses `warm.py`, `recovery.py`, and `fork.py` to manage filtered retrieval, recovery, and isolated investigations.

The model chooses actions, the harness executes the interaction, and orchestration manages state across runs.

Q16. What is the difference between deterministic enforcement and prompt guidance?

System 4's 30-minute recovery threshold is deterministic code behavior tested at the 30- and 31-minute boundaries. System 1's tool descriptions guide the model in choosing between adjuster routing and human escalation. I would enforce hard safety boundaries in code and use prompt/tool descriptions for decisions requiring contextual interpretation. Evidence: `recovery.py`, System 4 tests, and System 1 tool definitions.

Q17. How does context management differ between Systems 2 and 4?

System 2 reduces context from 38,708 to 16,794 tokens, a 56.61% reduction. System 4 keeps hot state to 643 bytes and uses a 30-minute threshold to decide whether to resume. System 2 compresses long conversation history; System 4 uses SQL-filtered retrieval and compact persisted state across shifts. Evidence: `budget.json`, `hot_state_size.txt`, and `recovery.py`.

Q18. Why are tests important beyond a successful run?

A successful trace only demonstrates one execution path. `test_no_integer_literal_iteration_cap_in_loop` checks that the agent loop does not rely on a hardcoded iteration limit. System 4's 30- and 31-minute recovery tests exercise the boundary between resuming and starting fresh. These tests cover failure conditions that may not appear in a normal run. Evidence: System 1 anti-pattern tests and `pytest_S4.log`.

Q19. What is the potential blast radius of an orchestration failure?

An incorrect SQL time window could omit defects and lead to an incomplete shift summary. Resuming stale work could produce decisions based on outdated state. SQL filtering in `warm.py`, the recovery threshold in `recovery.py`, compact hot state, and fork isolation reduce these risks. A further safeguard would be to pause scheduled runs and request manual review if state validation or retrieval checks fail.

## Part 3 — Honest Assessment

Q20. What broke, and what would you improve?

The initial shift output reported `0 new defects`, even though the recorded response described a defect cluster. After rerunning, the updated `shift-output.txt` reported 17 new defects, and the scratchpad recorded the investigation window and findings. The submitted `eval_control.jsonl` also contained no usable evaluation results, so that control comparison still needs to be regenerated.

I would automate evidence generation so the workflow seeds fixtures, runs the shift, executes both full and control evaluations, validates JSONL contents, and saves the actual outputs. I would also pin dependencies and document setup commands to make the environment reproducible.
