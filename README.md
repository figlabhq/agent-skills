# FIGLAB Agent Skills

A collection of Agent Skills for the FIGLAB team.

## Installation

```bash
npx skills add figlabhq/agent-skills
```

## Skills

| Skill | What it does |
|-------|--------------|
| `code-convention` | Battle-tested conventions for Laravel/PHP (with React/TypeScript frontend). Applied while writing or refactoring code. |
| `owasp-security` | Security standards for review and implementation — OWASP Top 10:2025, ASVS 5.0, Agentic AI security. |
| `phpcs-check-fix` | Fix PHP code style with PHPCS/PHPCBF using the FigLab Coding Standard. |
| `refine-tests` | Review and prune tests from a coding session — lean, high-confidence suites over max count. |
| `research` | Deep research before planning. Launches parallel agents across docs, web, and codebase, then synthesizes findings. |

## Usage

Once installed, Claude loads each skill automatically when its triggers match. Invoke one explicitly with `/<skill-name>`.
