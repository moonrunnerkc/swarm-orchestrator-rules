# Contributing

Thanks for considering a rule pack contribution. This guide walks
through adding a single rule end-to-end, lists the schema fields for
each artifact type, and describes what reviewers look at before the
automated quality gate ships.

If you have not used the orchestrator's rule loader before, read the
**Layout** section of the [README](README.md) first. The repo layout
mirrors what the loader walks; an extra directory layer is the most
common source of "my rule loaded silently as nothing".

## Quick Start

Five minutes from "I want to add a rule" to a draft PR:

1. **Fork** `moonrunnerkc/swarm-orchestrator-rules` on GitHub.

2. **Clone your fork** and create a working branch:

   ```bash
   git clone https://github.com/<your-handle>/swarm-orchestrator-rules.git
   cd swarm-orchestrator-rules
   git checkout -b cheat-rule-hardcoded-api-key
   ```

3. **Pick a path under `<your-handle>/<pack-name>/<rule-type>/`.**
   The author segment is your GitHub handle; the pack name is yours
   to choose; rule type is one of `cheat-rules`, `property-templates`,
   or `regression-fixtures`. Add a `pack.yaml` at the pack root so the
   loader's manifest read picks up your version string.

   Example:

   ```
   alex-doe/
   └── security-extras/
       ├── pack.yaml
       └── cheat-rules/
           └── hardcoded-api-key.yaml
   ```

4. **Validate locally** by pointing the orchestrator's loader at your
   working tree and running a rule-consuming command. From a checkout
   of `moonrunnerkc/swarm-orchestrator`:

   ```bash
   cd /path/to/swarm-orchestrator
   npm run build
   mkdir -p /tmp/local-rules-test/.swarm
   cat > /tmp/local-rules-test/.swarm/config.yaml <<EOF
   rules_dir: /path/to/your/swarm-orchestrator-rules
   rule_packs:
     - alex-doe/security-extras
   EOF
   (cd /tmp/local-rules-test && /path/to/swarm-orchestrator/dist/src/cli.js gates .)
   ```

   The startup line should read
   `Loaded N rules from M packs: ..., alex-doe/security-extras`.
   If your pack does not appear or N is lower than expected, the
   loader emits an error line naming the file and the schema field
   that failed.

5. **Commit and open a PR.** Title format:
   `<rule-type>: short description (alex-doe/security-extras)`.
   Use the PR description to include the rationale described in
   **What reviewers look for** below.

## Worked Example: a cheat rule

Suppose you want the cheat detector to flag hardcoded credentials in
test files. Tests often use real-looking secrets that an LLM agent
might paste into production code by mistake; flagging them at the
test layer catches the leak before the production path even comes up.

### The rule file

`alex-doe/security-extras/cheat-rules/hardcoded-api-key-in-test.yaml`:

```yaml
ruleId: hardcoded-api-key-in-test
version: 1.0.0
severity: high
message: Possible hardcoded credential in a test file. If this is a real key, rotate it; if it is a fixture, prefix the value with "fake-" so this rule does not flag it.
languages: [javascript, typescript, python]
paths:
  include:
    - "**/*test*"
    - "**/__tests__/**"
    - "**/tests/**"
    - "**/*.spec.*"
patterns:
  - pattern-regex: '(?i)(api[_-]?key|secret|token)\s*[:=]\s*["''][A-Za-z0-9_\-]{20,}["'']'
  - pattern-not-regex: '["'']fake-[A-Za-z0-9_\-]+["'']'
```

What this rule does and does not do:

- **Scope is narrow.** `paths.include` restricts firing to files
  whose path matches a test naming convention. A production file with
  the same shape lives or dies under a different rule.
- **The negative pattern (`pattern-not-regex`) carves out the
  documented escape hatch.** A fixture string prefixed with `fake-`
  passes. Without that opt-out, the rule's false-positive rate on a
  realistic test corpus is high enough to be noise.
- **The error message tells the reader what to do.** "Rotate it" is
  the action when the value is real; "prefix with fake-" is the
  action when it is a fixture. A rule whose message is "hardcoded
  secret" without the next step gets dismissed.

### The pack manifest

`alex-doe/security-extras/pack.yaml`:

```yaml
name: security-extras
author: alex-doe
version: 1.0.0
description: Cheat rules focused on credentials, secrets, and dangerous defaults that LLM agents tend to copy across files.
ruleCount: 1
maintainer: alex-doe
homepage: https://github.com/alex-doe/swarm-orchestrator-rules
```

### Validating

Point the loader at your working copy and run a rule-consuming
command. The startup line is the source of truth:

```text
Loaded 6 rules from 2 packs: swarm-orchestrator/cheat-defaults, alex-doe/security-extras
```

If the count is lower, the loader emits a per-file error explaining
why a file failed to load. Common errors:

- `schema validation failed: /ruleId: must match pattern` — the
  `ruleId` is not lowercase-kebab-case.
- `rule rejected: <file> — top-level is not an object (got string)`
  — your YAML parses as a scalar, usually because of an unquoted
  colon in `message`.
- `configured rule pack 'alex-doe/security-extras' not found at
  <path>` — the `rules_dir` in your `.swarm/config.yaml` does not
  point at the parent of `alex-doe/`.

### What a passing PR looks like

The diff is the rule file plus the pack manifest. The PR description
lists, in this order:

1. **One sentence on the bug class.** "Hardcoded credentials in test
   files leak into production via LLM copy-paste."
2. **One known-bad code sample** the rule should flag, copyable into
   a test file.
3. **One clean code sample** the rule must not flag, ideally one that
   is structurally close to the bad sample (a test using a `fake-`
   prefixed fixture works).
4. **The startup-line output** confirming local validation passed.

That is enough for a reviewer to decide. The CI gate will replace
points 2-4 with corpus-based numbers when it ships; until then, this
is the floor.

## Rule Type Reference

### Cheat rules

`<author>/<pack>/cheat-rules/<rule>.yaml`. Schema id:
`https://swarm-orchestrator.dev/schemas/cheat-rule.schema.json`.

Required fields:

| Field | Type | Notes |
|---|---|---|
| `ruleId` | string | kebab-case, lowercase, must start with a letter |
| `version` | string | semver (`MAJOR.MINOR.PATCH`); pre-release and build metadata accepted |
| `severity` | enum | `high`, `medium`, `low`. Maps to Semgrep ERROR / WARNING / INFO |
| `message` | string (≤ 200 chars) | Finding message; `{{var}}` placeholders are passed through to Semgrep |
| `languages` | array of strings | Semgrep language list (e.g. `[javascript, typescript, python]`) |
| One pattern field | one of: `pattern`, `pattern-regex`, `pattern-either`, `patterns` | The Semgrep matcher |

`additionalProperties: true` on the schema, so any other Semgrep
field (`paths`, `metadata`, `fix`, `pattern-not`, `pattern-inside`,
…) is accepted. The loader does not interpret them; Semgrep does.

What makes a good cheat rule:

- **Specific to a class of cheating, not "code smell."** Cheating
  is "the agent gamed the test" — hardcoded answers, exception
  swallowing in implementation, mock mutation in tests, test files
  edited to match a buggy implementation. A rule that flags long
  functions is a code-style rule and belongs elsewhere.
- **Low false-positive rate on realistic code.** A rule that fires
  on legitimate idiomatic code in 1 of 10 random files is noise; the
  battery already runs many checks and noise crowds out signal. The
  CI gate (when live) targets ≤ 10% FP. Manual reviewers eyeball for
  obviously over-broad patterns.
- **Actionable message.** `{name} is a hardcoded answer; either
  derive it from input or move the literal into a test fixture` is
  actionable. `Hardcoded value detected` is not.
- **Severity matches blast radius.** `high` for things that change
  production behaviour silently (exception swallowing, hardcoded
  credentials). `medium` for cheating likely to be caught
  downstream. `low` for stylistic cheating where a human would
  notice on review.

Reference: Semgrep pattern syntax at
[semgrep.dev/docs/writing-rules/pattern-syntax](https://semgrep.dev/docs/writing-rules/pattern-syntax).

### Property templates

`<author>/<pack>/property-templates/<template>.yaml`. Schema id:
`https://swarm-orchestrator.dev/schemas/property-template.schema.json`.

Required fields:

| Field | Type | Notes |
|---|---|---|
| `ruleId` | string | kebab-case |
| `version` | string | semver |
| `languages` | array | subset of `[typescript, javascript, python, go, java]` |
| `targetType` | enum | `pure-function`, `mutation`, or `invariant` |
| `generators` | object | parameter-type-name → generator-symbol-path |
| `invariant` | string | template body; `{{fn}}`, `{{args}}` placeholders substituted by the gate |
| `severity` | enum | `high`, `medium`, `low` |
| `message` | string (≤ 200 chars) | finding message; `{{fn}}` and `{{counterexample}}` available |

Generator symbol paths are language-specific. Examples:

- Python: `hypothesis.strategies.integers`,
  `hypothesis.strategies.text(min_size=1, max_size=100)`
- TypeScript / JavaScript: `fast-check.integer`,
  `fast-check.string`, `fast-check.array(fast-check.integer)`

The `targetType` answers "what shape does the template apply to":

- **`pure-function`**: input → output, deterministic, no side
  effects. Invariant is typically a roundtrip property
  (`f(g(x)) === x`) or an algebraic identity.
- **`mutation`**: in-place modification of an argument. Invariant
  asserts a post-condition on the mutated argument and the return
  value.
- **`invariant`**: a property that must hold across a sequence of
  operations on a stateful object. Less common; useful for
  collections and parsers.

**Status note: the consumer is not data-driven yet.** The
orchestrator's property gate today uses inline TypeScript builders
to emit Hypothesis (Python) and fast-check (TypeScript) harnesses.
Templates contributed to this registry will be picked up by the gate
once the data-driven refactor ships in a future swarm-orchestrator
release. The schema is stable; what is missing is the consumer
loop. Please contribute templates anyway — the schema does not
change between now and the consumer landing, so today's templates
work tomorrow.

### Regression fixtures

`<author>/<pack>/regression-fixtures/<fixture>.yaml`. Schema id:
`https://swarm-orchestrator.dev/schemas/regression-fixture.schema.json`.

Required fields:

| Field | Type | Notes |
|---|---|---|
| `ruleId` | string | kebab-case; convention: mirror the bug's short name or issue id |
| `version` | string | semver |
| `originalBugCommit` | string | 40-character lowercase hex SHA of the commit that last exhibited the bug, before the fix |
| `testFilePath` | string | repo-relative path to the test that proves the fix; must not be absolute or contain `..` |
| `affectedFiles` | array | repo-relative paths the fix touched; same restrictions; minimum 1 entry |
| `severity` | enum | `high`, `medium`, `low` |
| `message` | string (≤ 200 chars) | one-sentence bug description |

**Status note: the consumer is v8 forward work.** The regression
falsifier that consumes these fixtures is on the
`v8-overhaul-plan.md` roadmap, not in the current orchestrator
release. Writing fixtures today is forward work: the schema and
registry support it; the orchestrator does not yet act on them.
Contributors who want their fixtures to land before the consumer
ships should still open PRs — the registry is intended to absorb
that backlog so the consumer ships against a populated corpus on
day one.

## Versioning

- **Per-rule semver.** Bump the rule's `version` on any behavioural
  change: pattern, severity, message wording, language list, target
  type. Adding a `paths.include` that narrows scope is a behavioural
  change. Fixing a typo in a comment is not.
- **Pack version is separate from rule versions.** The `version`
  field in `pack.yaml` describes the pack as a unit and follows its
  own semver. A pack version bump is appropriate when adding,
  removing, or replacing a rule; not necessary for in-place rule
  edits whose own versions bumped.
- **Breaking changes get a major bump.** Renaming a `ruleId`,
  changing `targetType` for a property template, or changing
  `originalBugCommit` for a regression fixture are breaking changes.
  Downstream pinning is the consumer's recourse.

## CI Gate (when it ships)

The plan: every PR runs the new and changed rules against a
labeled-patch corpus stored under `test-corpus/`. The gate fails the
PR if false-positive rate exceeds 10% or true-positive rate falls
below 90%. The corpus build, the runner, and the thresholds live in
the swarm-orchestrator v7 plan under the rule-pack workstream.

Until that ships, contributions go through manual review. Reviewers
look for the four PR-description points listed in **What a passing
PR looks like** above. The numeric thresholds are still the target
the manual reviewer estimates against; the gate just automates the
estimate.

## What reviewers look for

- The PR description includes the four points from **What a passing
  PR looks like**.
- The pattern is structurally narrow. A maintainer should be able to
  look at the regex or Semgrep pattern and predict its FP behaviour
  without running it.
- The message is actionable.
- Schema validation passes locally; reviewers do not re-run it.
- The `version` and `pack.yaml` `version` are sane (a new rule
  starts at `1.0.0`; an in-place edit bumps the `MINOR` or `PATCH`
  per semver).

## Code of Conduct

The orchestrator's code of conduct applies. If a CoC is added to
[`moonrunnerkc/swarm-orchestrator`](https://github.com/moonrunnerkc/swarm-orchestrator),
this repo follows it. Until then: the contribution process is
factual and technical. Reviewers reject for reasons that map to one
of the items in **What reviewers look for**, not for taste.
