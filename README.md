# pipedrive-hot-deal-detector

A Claude Code skill that evaluates sales leads for urgency signals, syncs deal probability with Pipedrive via MCP, and keeps a local `HOT_DEALS.md` file sorted by priority.

## What it does

- Searches Pipedrive for the deal first, then branches on status:
  - **Open / new** → scores the lead by timeframe signals (base 40%, +45% high urgency, +25% short term).
  - **Won / Lost** → skips scoring and removes the deal from the local list.
- Only deals scoring **≥ 60%** are written to Pipedrive and tracked locally.
- Maintains `HOT_DEALS.md` in the working directory, sorted highest-to-lowest probability.

See [`SKILL.md`](./SKILL.md) for the full instructions Claude follows.

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed.
- A Pipedrive MCP server configured in Claude Code, exposing search/create/update tools for deals.

## Install

Clone this repo into your Claude Code skills directory:

```sh
# User-level (available across all projects)
git clone <repo-url> ~/.claude/skills/pipedrive-hot-deal-detector

# Or project-level (only in this project)
git clone <repo-url> .claude/skills/pipedrive-hot-deal-detector
```

Restart Claude Code (or start a new session) and the skill will be auto-loaded.

## Usage

Trigger it by describing a customer interaction, for example:

> "Acme Corp emailed asking to roll this out ASAP before their month-end close."

Claude will look up Acme in Pipedrive, score the urgency, update or create the deal if it qualifies, and sync `HOT_DEALS.md`.

## Output format

`HOT_DEALS.md` is overwritten on each run with this structure:

```markdown
# High Priority Pipeline

| Probability | Deal ID | Company / Lead | Timeframe Signal | Context Summary |
| :--- | :--- | :--- | :--- | :--- |
| 85% | 1045 | Acme Corp | High Urgency (ASAP) | Needs implementation before the end-of-month accounting close. |
| 65% | 1089 | Globex Inc | Short Term (This quarter) | Budget is allocated for Q3 deployment. |
```
