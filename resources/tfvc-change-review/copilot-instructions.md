# Project Overlay: tfvc-change-review (Aura)

This file is a project-specific overlay. The global review logic should live in:
`C:\Users\ZHAOWE\.copilot\skills\tfvc-change-review\SKILL.md`

## Routing

- When user asks TFVC/TFS code review with local paths, folders, or changeset ID, route to global `tfvc-change-review`.
- Keep review scoped to TFVC real diff hunks + minimal local context.
- Do not scan entire file unless changed hunks require one-hop dependency checks.

## Aura-Specific Context

- If `.github/kb/README.md` exists, use it as KB index first.
- Prefer Aura pages under `.github/kb/projects/aura.md` and `.github/kb/pages/*.md` when diff touches UI, data-grid, or form flows.
- If ADO work item is available, verify diff against AC/repro and Aura KB conventions before final findings.

## TFVC Defaults

- If `.github/tfvc-defaults.json` exists, use `MainRoot` from it for TFVC commands.
- If missing, ask user for mapped local root instead of guessing alternate roots.

## Output Style

- Findings first, severity ordered.
- Mention where Aura KB rule is applied (page path) when giving convention-related findings.
- If no issue found, state no findings and list residual test gaps.