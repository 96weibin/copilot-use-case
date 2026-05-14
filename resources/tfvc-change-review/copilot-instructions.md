# Project Guidelines

## TFVC Review Routing

- When the user asks for code review and provides TFVC or TFS local paths, folder paths, or change hints such as add, edit, delete, prefer the tfvc-change-review skill.
- For TFVC review requests, use tf diff as the source of truth for pending changes and review the changed hunks plus minimal local context, not the entire local file.
- If the user provides a folder path, enumerate pending changes under that folder before reviewing individual files.
- In local TFVC workspaces, do not rely on tf status /user:* unless a specific workspace is also supplied.

## TFVC Sync Routing

- When the user asks to pull latest code, update code, sync code, run tf get, or says phrases such as 拉最新代码, 拉取最新代码, 更新代码, prefer the tfvc-get-latest skill.
- For TFVC sync requests without an explicit path, prefer `C:\source\Main` only when that path exists locally; otherwise ask the user to choose or provide the mapped TFVC path.
- For TFVC sync requests without an explicit branch, assume the user wants the local workspace mapping for main.
- Use tf get for TFVC sync requests, not Git pull.

## Review Output

- Present findings first, ordered by severity, and keep the review focused on risks introduced by the diff.
- If no issues are found in the change, state that explicitly and mention any residual testing gaps.