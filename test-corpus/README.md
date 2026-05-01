# test-corpus

This directory holds the labeled-patch test corpus that the rule-quality CI
gate will run new and changed rules against. The corpus and the gate that
consumes it land in a follow-up swarm-orchestrator release; until then, this
directory is intentionally empty and contributions go through manual review.

The plan, briefly: each entry under `test-corpus/` is a small repository
snapshot plus a label (`clean` or `bug-<id>`). When a PR adds or modifies a
rule, CI runs the rule against every snapshot and records true positives,
false positives, true negatives, and false negatives. The gate fails the PR
if false-positive rate exceeds 10% or true-positive rate falls below 90%.
The exact thresholds and the corpus build live in
[swarm-orchestrator's v7 plan](https://github.com/moonrunnerkc/swarm-orchestrator)
under the rule-pack workstream.

For now, contributors should include a brief test-case rationale in their
PR description (a known-bad example the rule should catch and a clean
example it must not flag for cheat rules; for property templates, a
function with known edge cases the harness exercises). Reviewers use that
rationale plus a manual eyeball of the rule pattern.
