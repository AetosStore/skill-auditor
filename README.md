[README.md]
# Skill Auditor for Claude Code

A read-only static auditor for Claude Code skills.

Skill Auditor scans installed or supplied Claude Code skills for configuration problems, weak trigger definitions, overlapping skills, broken references, maintainability issues, and potentially risky bundled capabilities such as shell execution, network access, credential access, hooks, dependency installation, and destructive filesystem operations.

It is designed for **triage and review**, not security certification.

## What it does

Skill Auditor checks four main areas:

- **Discovery and configuration** — frontmatter, invocation controls, trigger guidance, tool policies, and unreachable skills.
- **Trigger overlap and precedence** — competing slash commands, similar auto-invocation metadata, nested variants, and override situations.
- **Security and capability inventory** — shell execution, networking, credential access, remote-code patterns, hooks, binaries, symlinks, destructive operations, and dangerous capability combinations.
- **Maintainability** — dead references, oversized skill files, long reference files without navigation, and missing evaluation material.

The audit engine is intentionally **read-only** and uses only the Python standard library.

## Why this exists

Claude Code skills can contain more than Markdown instructions. Depending on the skill, they may also include scripts, hooks, tools, dependencies, dynamic shell context, network calls, or access to local credentials.

That makes reviewing a skill manually useful, but increasingly tedious as the number of installed skills grows.

Skill Auditor provides a consistent first pass:

1. discover skills;
2. inspect their configuration and bundled files;
3. identify suspicious or fragile patterns;
4. generate structured findings with stable IDs;
5. produce a human-readable Markdown report;
6. leave all changes to an explicit approval step.

A finding indicates **evidence or capability**, not malicious intent. A clean report does **not** prove that a skill is safe.

## Example

Run the scanner:

```bash
python3 scripts/audit.py --json-out /tmp/skill-audit-findings.json
```

Generate a Markdown report:

```bash
python3 scripts/report.py \
  /tmp/skill-audit-findings.json \
  --out skill-audit-report.md
```

To scan one or more explicit skill locations:

```bash
python3 scripts/audit.py \
  --paths ~/.claude/skills ./some-skill-directory \
  --json-out /tmp/skill-audit-findings.json
```

The report includes:

- overall status;
- findings by severity;
- per-skill status;
- stable finding IDs;
- evidence;
- suggested fixes;
- confidence where relevant;
- scanned locations;
- audit coverage limits.

## Severity levels

| Severity | Meaning |
|---|---|
| **CRITICAL** | Pattern may directly execute untrusted remote code or strongly resembles instruction override/deception. Manual review before trust. |
| **HIGH** | Broken/unreachable configuration or a dangerous capability combination, such as credential access together with networking. |
| **MEDIUM** | Meaningful reliability or security concern requiring context, such as shell execution, hooks, dependency installation, or dead references. |
| **LOW** | Portability, quality, or maintainability issue that normally does not block use. |
| **INFO** | Context or inventory that affects interpretation but is not itself a defect. |

Severity and confidence are separate. A high-confidence finding means the pattern was identified with high confidence; it does not automatically mean the behavior is unsafe.

## Checks

Examples of checks currently included:

### Configuration and quality

- malformed or missing frontmatter;
- unknown frontmatter fields;
- weak or missing discovery text;
- discovery text truncation;
- weak trigger guidance;
- unreachable skills;
- conflicting tool policy;
- invalid or non-portable names;
- oversized or empty skill bodies;
- broken local references;
- long reference documents without navigation;
- missing evaluation material.

### Triggering and overlap

- precedence shadowing;
- duplicate non-plugin command variants;
- bundled-skill overrides;
- similar auto-invocation metadata.

Plugin skills are treated as namespaced for command invocation, so identical short names do not automatically count as bare slash-command shadowing.

### Security and capabilities

- remote content piped to a shell;
- suspicious instruction-override patterns;
- possible exfiltration instructions;
- dynamic `eval` / `exec` execution;
- credential or environment-secret access;
- destructive filesystem operations;
- encoded payload execution;
- credential access combined with networking;
- external symlinks;
- local shell/process execution;
- network access;
- dependency installation;
- Claude Code dynamic `!` shell context;
- skill hooks;
- directive-like hidden HTML comments;
- unscanned binaries or unsupported executables;
- writes to the user's home directory;
- pre-approved consequential tools.

See [`references/checks.md`](references/checks.md) for the complete check catalog and interpretation notes.

## Installation

### Option 1: Install as a personal Claude Code skill

Clone the repository:

```bash
git clone https://github.com/AetosStore/skill-auditor)
cd skill-auditor
```

Copy the skill into your personal Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills
cp -R skill-auditor ~/.claude/skills/skill-auditor
```

The resulting structure should contain:

```text
~/.claude/skills/skill-auditor/
├── SKILL.md
├── scripts/
│   ├── audit.py
│   └── report.py
├── references/
│   └── checks.md
└── evals/
    └── cases.md
```

### Option 2: Use the scanner directly

You can also run the Python scripts from the repository without installing the skill:

```bash
python3 skill-auditor/scripts/audit.py \
  --paths ~/.claude/skills \
  --json-out /tmp/skill-audit-findings.json

python3 skill-auditor/scripts/report.py \
  /tmp/skill-audit-findings.json \
  --out skill-audit-report.md
```

## Requirements

- Python 3
- Claude Code skills to inspect
- No third-party Python packages

The static engine uses the Python standard library only.

## Repository structure

```text
.
├── README.md
└── skill-auditor/
    ├── SKILL.md
    ├── scripts/
    │   ├── audit.py
    │   └── report.py
    ├── references/
    │   └── checks.md
    └── evals/
        └── cases.md
```

### `SKILL.md`

Defines the Claude Code skill workflow: audit first, interpret the evidence, present findings, and make changes only after approval.

### `scripts/audit.py`

Read-only static audit engine. Discovers skill directories, parses configuration, analyzes trigger overlap and bundled capabilities, and emits structured JSON findings.

### `scripts/report.py`

Converts the JSON findings into a Markdown audit report.

### `references/checks.md`

Documents the severity model, individual checks, and known coverage limitations.

### `evals/cases.md`

Contains manual/model-based evaluation cases for triggering behavior, namespace handling, suspicious-skill review, and approval boundaries.

## Safety model

Skill Auditor follows a deliberately conservative workflow:

**Audit first. Interpret second. Change only after approval.**

The scanner does not execute the scripts contained in audited skills.

When a finding is legitimate behavior, it should be documented rather than automatically removed. For example, network access may be expected in a deployment skill. The important questions then become:

- Which destination can it contact?
- What data can it transmit?
- Does it also have credential access?
- How are commands constructed?
- Can untrusted input reach those commands?

For consequential changes, the skill workflow uses stable finding IDs and calls for backups and re-auditing after modifications.

## Known limitations

Static analysis has important blind spots. Skill Auditor does not prove runtime behavior and cannot fully inspect:

- runtime-only data flow;
- content fetched after execution;
- dependency or package integrity;
- vulnerabilities in third-party packages;
- compiled binary internals;
- MCP server or tool implementations;
- enterprise-managed skills that are not exposed through a scan path;
- active `--add-dir` / `/add-dir` locations unless explicitly supplied;
- built-in Claude Code skill internals;
- legacy `.claude/commands/` files;
- external Claude Code settings and permission state such as skill overrides.

Use the report as a **triage layer before manual review and runtime isolation**, not as a certification.

## Self-auditing

The auditor is designed to be able to scan its own package.

Because its source contains regex patterns describing dangerous strings, the engine masks only its own known regex-definition catalog during self-scanning. That prevents the detection rules themselves from being reported as security findings while leaving the rest of the package subject to normal checks.

The masking is surfaced as an informational finding rather than hidden.

## Development and evaluation

Evaluation cases live in:

```text
skill-auditor/evals/cases.md
```

Current scenarios include:

- full installed-skill audit;
- review of one suspicious skill;
- automatic-trigger troubleshooting;
- negative triggering for skill-authoring requests;
- plugin namespace behavior;
- approval boundaries for fixes.

When adding a new check, add or update an evaluation case that demonstrates the behavior you expect.

## Contributing

Contributions are welcome, especially for:

- improved trigger-overlap heuristics;
- new Claude Code frontmatter/configuration checks;
- lower false-positive rates;
- additional static security checks;
- reproducible test fixtures;
- coverage for newly introduced Claude Code skill features.

When proposing a security check, prefer evidence-based findings over claims about intent.

Good:

> This skill reads credentials and also makes network requests. Trace whether credential data can reach the network call.

Bad:

> This skill steals credentials.

## Responsible use

Do not treat static findings as proof that another developer's skill is malicious.

Security findings should be reviewed in context and reproduced where possible before making public accusations.

## Status

Skill Auditor v2 is an independent community tool and is not an official Anthropic or Claude Code security product.
