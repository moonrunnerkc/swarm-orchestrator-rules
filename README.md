# swarm-orchestrator-rules

Community rule pack registry for [swarm-orchestrator](https://github.com/moonrunnerkc/swarm-orchestrator).
Holds three artifact types the orchestrator's falsification battery and
related layers consume: cheat-detection rules (Semgrep-compatible plus
orchestrator metadata), property-test templates, and regression fixtures.

The orchestrator ships a small set of built-in rules in its own repository
under `config/built-in-rules/`. This registry is for everything else:
forks, alternative rule sets, language-specific cheat rules,
property-template families, and the eventual regression-fixture corpus.

## Layout

The orchestrator's rule loader (`src/rules/loader.ts` in the main repo)
walks `<rules-dir>/<author>/<name>/<rule-type>/` looking for `.yaml`,
`.yml`, or `.json` files. This repo's top-level mirrors that layout
exactly so cloning the repo into `~/.swarm/rules/` is the entire install:

```
.
├── LICENSE
├── README.md
├── CONTRIBUTING.md
├── swarm-orchestrator/         # author
│   └── cheat-defaults/         # pack name
│       ├── pack.yaml           # manifest (name, version, ruleCount, …)
│       └── cheat-rules/        # rule type (one of: cheat-rules, property-templates, regression-fixtures)
│           ├── complexity-mismatch.yaml
│           ├── exception-swallowing.yaml
│           ├── hardcoded-answer.yaml
│           ├── mock-mutation.yaml
│           └── test-modification.yaml
└── test-corpus/                # labeled-patch corpus for the CI gate (see below)
```

When you fork to add a pack, add a sibling directory under your own
GitHub handle as `<your-handle>/<pack-name>/`. The author segment must
match where the pack will be hosted so the loader can attribute findings
back to a known source.

## Quick Start

### Install

Clone the registry into the orchestrator's default rules directory:

```bash
git clone https://github.com/moonrunnerkc/swarm-orchestrator-rules.git ~/.swarm/rules
```

Cloning into `~/.swarm/rules` directly works only when the directory
does not yet exist. If you already have a `~/.swarm/rules/` from prior
contributions, clone into a sibling and merge:

```bash
git clone https://github.com/moonrunnerkc/swarm-orchestrator-rules.git /tmp/registry
cp -R /tmp/registry/swarm-orchestrator ~/.swarm/rules/
cp -R /tmp/registry/test-corpus ~/.swarm/rules/  # optional
```

### Opt in

Packs in the community location load only when you list them explicitly.
Add a `.swarm/config.yaml` to your project:

```yaml
rule_packs:
  - swarm-orchestrator/cheat-defaults
```

`rules_dir` defaults to `~/.swarm/rules`. Override it in the same file
if you keep packs elsewhere:

```yaml
rules_dir: /opt/team-rules
rule_packs:
  - swarm-orchestrator/cheat-defaults
  - your-team/your-pack
```

### Verify

Run any rule-consuming orchestrator command (for example
`swarm gates .`) and check the startup line:

```text
Loaded 5 rules from 1 packs: swarm-orchestrator/cheat-defaults
```

A configured pack that the loader cannot find on disk produces an error
line that names the missing pack and the install hint, but does not
crash the run.

## Pack Catalog

| Pack | Rules | Description |
|---|---:|---|
| `swarm-orchestrator/cheat-defaults` | 5 cheat rules | Default cheat-detection patterns shipped with the orchestrator. Mirrors `config/built-in-rules/swarm-orchestrator/cheat-defaults/` in the main repo so contributors can fork it as a worked example. Loading it from this registry while the same pack is also active built-in produces duplicate rules; pick one location, not both. |

The catalog grows by PR. See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the
authoring guide.

## Status Notes

- **CI quality gate is not live yet.** The plan is for new and changed
  rules to clear false-positive ≤ 10% / true-positive ≥ 90% on a
  labeled-patch corpus. Until that lands, contributions are reviewed
  manually with the same criteria in mind. See `test-corpus/README.md`
  for the expected corpus layout.
- **Property templates: schema stable, consumer not data-driven yet.**
  The orchestrator's property gate currently uses inline TypeScript
  builders. Property templates contributed here will be picked up once
  the gate refactor lands in a future release.
- **Regression fixtures: forward work.** The regression falsifier that
  consumes these fixtures is part of v8 of the orchestrator
  (`v8-overhaul-plan.md`). The schema is in place so contributors can
  start authoring fixtures today.

## Contributing

Read [`CONTRIBUTING.md`](CONTRIBUTING.md). It walks through adding one
rule end-to-end, lists schema fields for each artifact type, and
explains what reviewers look for during the manual-review phase before
the CI gate ships.

## License

ISC, matching the orchestrator. See [`LICENSE`](LICENSE).
