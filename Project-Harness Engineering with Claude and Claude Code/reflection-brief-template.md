# Reflection Brief — Harness Engineering Capstone

## Environment

* Name: \(REDDIPALLI SAI CHARISH\)

* Date: October 9, 2026

* Model(s): `claude-haiku-4-5-20251001` (System 1, per `summary.md`); recorded-response client with no live model calls in System 4.

* OS / Python: Linux, Python 3.13.0.

* Approx. API spend: Approximately $0.12 for System 1's 8-claim run (`summary.md`). System 2 used compression calls over approximately 12k and 11k token histories (`budget.json`). System 4 ran offline using `--recorded-response`, with effectively $0 API cost.

## Part 1 — Per-system

### System 1 — Agentic loop

1. Loop control

The trace for `claim_01_kitchen_fire` showed the sequence `tool_use → tool_use → end_turn`. On turn 1, the model called `lookup_policy`; on turn 2, it issued several `record_claim_fact` tool calls; and on turn 3, it returned `end_turn`. Loop termination is implemented in `claims_intake/loop.py`, inside the `run()` function. After each `client.messages.create(...)` call, the code checks `response.stop_reason`: `tool_use` executes tools and continues the loop, while `end_turn` returns a `FinalState`. The test `test_stop_reason_is_loop_control` in `test_antipatterns.py` verifies that termination is based on `stop_reason`.

2. Anti-pattern

One anti-pattern checked by `test_antipatterns.py` is the use of hardcoded iteration caps, such as `for _ in range(3)` or `while turns < 5`. The test `test_no_integer_literal_iteration_cap_in_loop` uses AST analysis to detect these constructs in `loop.py`. If a fixed cap were used, claims such as `claim_02_stolen_bike` and `claim_03_water_damage`, which required roughly five turns, could terminate before reaching `classify_claim` or `route_to_adjuster`. This would leave some claims incompletely processed. The test ensures the loop follows the model's actual `stop_reason` instead of an arbitrary turn limit.

3. Tool design

`route_to_adjuster` and `escalate_to_human` accept similar claim-summary inputs, but their descriptions distinguish when each should be used. Claims with classification confidence of at least `0.6` are routed automatically, while those below `0.6` require escalation. Structured errors from `_t_lookup_policy` include `is_error`, `error_category`, and `is_retryable`. For example, `is_retryable: false` tells the agent that repeating the same request is inappropriate, allowing it to request a corrected policy ID or escalate instead of blindly retrying. A generic error string would not reliably provide this machine-readable recovery guidance.

4. Your numbers

According to `summary.md`, `claim_01_kitchen_fire` completed in 3 turns, using approximately 9,600 input tokens, 580 output tokens, and $0.0125. The more ambiguous `claim_03_water_damage` required 5 turns and cost approximately $0.021, including a clarification cycle. The additional `request_clarification` exchange increased the number of model interactions and token usage. The exact README sample figures are not available in the supplied evidence, so the numerical difference from that sample cannot be verified without the README.

### System 2 — Context strategy

5. The reduction

`budget.json` reports 38,708 baseline tokens and 16,794 assembled tokens, producing a 56.61% reduction. The `active` section dominates the assembled context at 15,789 tokens. By comparison, `case_facts` uses 204 tokens, `resolved_refund` uses 405 tokens, and `resolved_subscription` uses 414 tokens. The active section is preserved verbatim because it contains unresolved information needed for immediate reasoning. Summarizing it could remove details that affect the next decision.

6. Summarize vs preserve

The system summarizes resolved historical issues and preserves active issues exactly. According to `budget.json`, the refund case was compressed from 12,334 tokens to 392 tokens, while the subscription case was compressed from 11,475 tokens to 401 tokens. Meanwhile, the active section remained intact at 15,789 tokens. This strategy reduces historical context while retaining the operational details needed for ongoing work. The evidence is in the per-section token counts in `budget.json`.

7. Facts block

Comparing `eval.jsonl` with `eval_control.jsonl`, Q6 regressed. In `eval.jsonl`, the model correctly returned `payment_update_status: in_progress`, and the evaluation passed. In `eval_control.jsonl`, the model stated that no structured status token existed, and the evaluation failed. Q1 passed in both evaluations and correctly identified the refund amount as $22.14. This demonstrates that summarization can lose structured operational facts even when straightforward narrative information remains available.

### System 3 — Claude Code config

8. Path-scoped rules

The file `.claude/rules/tests.md` contains this path-scoped frontmatter:

YAML

```
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
```

These patterns apply the rules to matching test files throughout the repository, including files in `src/components`, `src/pages`, `src/api`, and `src/db`. A directory-level `CLAUDE.md` would require more duplication to enforce the same convention across unrelated directories. The test `test_ac_02_06_test_file_matches_react_and_tests` verifies that a test file can inherit multiple rule sets simultaneously. This makes path-scoped rules useful for cross-cutting conventions.

9. Forked skill

The deploy-check skill contains the following configuration:

YAML

```
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
```

The `context: fork` setting isolates intermediate inspection output from the primary conversation, while the allowlist restricts the skill to reading and repository inspection. This reduces context pollution and limits the risk of accidental modifications during verification. Without forking, verbose diagnostic output could accumulate in the main context. Without the read-only tool restrictions, the skill could have access to actions beyond its verification purpose.

10. Scope

The validator completed successfully with exit code 0, according to the supplied validation summary. A project-level example is `./CLAUDE.md` and `.claude/rules/*.md`, which are committed to Git and shared with the team. A user-level example is `~/.claude/skills/deploy-check-strict/`, which exists in an individual developer's environment rather than the repository. This separates team-wide conventions from personal configuration. The validator output is the artifact supporting the successful configuration check.

### System 4 — Orchestration

11. Push work down

The shift-monitor output reported 3 high-severity and 2 medium-severity defects associated with capacitor bank `C-7`, all from lot `2026-0430-B`, plus one low-severity VP-4 vent squeal. The recommendation was to quarantine the lot. A subsequent execution reported `shift C: 0 new defects`. The indexed query in `shift_monitor/warm.py`, `WarmStore.defects_since()`, executes:

SQL

```
SELECT * FROM defects
WHERE ts > ?
ORDER BY ts DESC
LIMIT ?
```

Filtering happens in SQLite before prompt construction, so the model receives only the relevant subset rather than the entire historical defect dataset. The test `test_gather_new_defects_has_no_python_side_filtering` verifies that filtering remains in the database layer.

12. Crash recovery

`recovery.py` defines `STALE_RESUME_THRESHOLD_MINUTES = 30`. Its `decide()` function resumes work only when incomplete tasks exist and the last work occurred within the threshold; otherwise, it chooses a fresh start. The tests `test_recovery_decide_truth_table[30-False-resume]` and `test_recovery_decide_truth_table[31-False-fresh]` verify behavior at the 30- and 31-minute boundaries. A fresh start with an injected summary can be more reliable because it preserves earlier findings without blindly continuing stale reasoning. This reduces the risk of acting on outdated assumptions after a long interruption.

13. Small state

The file `data/hot_state.json` measured approximately 643 bytes, according to the supplied run summary, against a reported budget of approximately 5 KB. The test `test_hotstate_rejects_more_than_20_hashes` checks that state remains bounded. Since the monitor runs once per shift indefinitely, unbounded state could increase storage use, prompt size, and recovery time over months of operation. Keeping a small snapshot makes resource usage more predictable. The exact file size and the test name provide the supporting artifacts.

## Part 2 — Synthesis

14. Three layers

* Model: `summary.md` records the System 1 model as `claude-haiku-4-5-20251001`. The trace for `claim_01_kitchen_fire` shows the model selecting tools and eventually returning `end_turn`.

* Harness: `claims_intake/loop.py` and `test_antipatterns.py` define and test the execution loop. The harness interprets `stop_reason`, executes tools, and prevents unsafe control-flow shortcuts such as hardcoded iteration caps.

* Orchestration: `shift_monitor/warm.py`, `recovery.py`, and `data/hot_state.json` manage retrieval, recovery, and persistent state. The indexed `defects_since()` query limits retrieved history, while the 30-minute threshold determines whether work resumes or restarts.

Together, these artifacts show three separate responsibilities: the model decides what to do next, the harness controls how tool calls execute, and orchestration manages work and state across runs.

15. Deterministic vs prompt

A deterministic behavior is the recovery decision in `recovery.py`: the 30-minute threshold is enforced by code and tested at the boundary. Another is the read-only `allowed-tools` list in `.claude/skills/deploy-check/SKILL.md`, which restricts verification actions. By contrast, tool descriptions in System 1 guide the model toward `route_to_adjuster` or `escalate_to_human` according to claim confidence. Code enforcement is appropriate for safety-critical boundaries and invariants; prompts and descriptions are appropriate for semantic decisions that require interpreting context. The distinction is supported by `test_recovery_decide_truth_table[30-False-resume]` and the deploy-check skill configuration.

16. Context, two faces

System 2 manages context within a session: `budget.json` reduces the baseline from 38,708 to 16,794 tokens, a 56.61% reduction, while preserving the active section at 15,789 tokens. System 4 manages context across shifts: `data/hot_state.json` is approximately 643 bytes, and `recovery.py` uses a 30-minute staleness threshold to choose between resuming and starting fresh. Both systems preserve essential information while avoiding unnecessary history. System 2 uses summarization and selective verbatim preservation; System 4 uses indexed retrieval, compact persistent state, and recovery rules. These mechanisms address different time scales but share the same principle: retain relevant state, not the entire history.

17. Reliability you can't see in one run

The test `test_no_integer_literal_iteration_cap_in_loop` checks that `claims_intake/loop.py` does not use hardcoded iteration limits. A single successful claim run would not prove that the loop can handle longer or more complex claims without being cut off by an arbitrary cap. Similarly, `test_recovery_decide_truth_table[31-False-fresh]` verifies that work older than the staleness threshold starts fresh. These tests matter before shipping because edge cases and interrupted runs may not appear in a normal demonstration. The named tests provide evidence that these behaviors are checked explicitly.

18. Blast radius

For System 4, an orchestration failure could affect defect reporting and lot-quarantine recommendations for a production shift. The enforcement points include the indexed query in `shift_monitor/warm.py`, the resume-versus-fresh decision in `recovery.py`, and the bounded state in `data/hot_state.json`. If retrieval included excessive history or recovery reused stale work, the monitor could produce outdated or misleading recommendations. A safe operational kill switch would be to disable scheduled monitor execution and require manual review until the issue is resolved. The supplied artifacts do not identify a built-in kill-switch command, so this is a proposed operational safeguard rather than a verified existing feature.

## Part 3 — Honest assessment

19. What broke

The supplied setup notes report an `httpx` compatibility error: `Client.__init__() got an unexpected keyword argument 'proxies'`. A separate environment also required a newer Anthropic SDK to support `messages.count_tokens`. These issues were resolved by selecting compatible package versions. The experience indicates that dependency compatibility can break a project before its application logic is exercised. The recorded exception and the dependency-version fix are the available evidence; the supplied notes do not include the exact package lockfile or installation command.

20. What you'd change

I would introduce fully pinned dependency versions or lockfiles for each system rather than relying on transitive dependency resolution. The setup failures included `Client.__init__() got an unexpected keyword argument 'proxies'` and an Anthropic SDK version that did not support `messages.count_tokens`. Both issues required selecting compatible package versions. Separate lockfiles and documented setup commands would make the environment reproducible and reduce onboarding time. The evidence comes from the recorded setup failures in the supplied project notes.

