# research-to-package

[![Claude Code skill](https://img.shields.io/badge/Claude_Code-skill-d97757)](https://claude.com/claude-code)
[![License: Unlicense](https://img.shields.io/badge/license-Unlicense-blue)](LICENSE)
[![Last commit](https://img.shields.io/github/last-commit/odinokov/research-to-package)](https://github.com/odinokov/research-to-package/commits/main)

A Claude Code skill that turns a messy research/prototype codebase (loose scripts, notebooks, a one-off repo) into a production-grade, installable Python package — clean, typed, tested, documented, with an importable API (and a CLI when it's a file-in/file-out tool). Its governing rule is **faithfulness**: the science/output is preserved exactly while structure, typing, tests, and docs are brought to production standard.

## Requirements

- [Claude Code](https://claude.com/claude-code). Skills are auto-discovered from `SKILL.md` files in known locations — no build step.

## Install

Clone the skill into a skills directory — keep the folder name `research-to-package` (it must match the skill's `name:`, which `git clone` preserves).

**Personal** — available in every project on this machine:

```bash
git clone https://github.com/odinokov/research-to-package.git ~/.claude/skills/research-to-package
```

**Project** — shared with a repo, committed to source control:

```bash
git clone https://github.com/odinokov/research-to-package.git <repo>/.claude/skills/research-to-package
```

Update later with:

```bash
git -C ~/.claude/skills/research-to-package pull
```

## Verify

Start (or restart) a Claude Code session in any project and confirm the skill is listed / invokable:

```
/research-to-package
```

It also triggers automatically when you ask Claude to "productionize", "port", "package", "harden", "clean up", or "make installable" research code.

## Usage

In a Claude Code session inside your research code:

```
> productionize scripts/ into an installable package
```

The skill drives the work through phased gates (Recon → Scaffold → Port → Tests+docs → optional Optimize), stopping for your approval at each gate. You get a `src/`-layout package that installs with `pip`/`uv`, typed and documented, with a regression test asserting the new output matches the original's baseline exactly, and a `CHANGELOG.md` recording every change and why. Behavior-changing work (optimization, dependency upgrades) is opt-in and gated behind that regression test.

## License

[Unlicense](LICENSE) — public domain.
