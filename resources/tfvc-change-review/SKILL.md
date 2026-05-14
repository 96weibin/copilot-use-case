---
name: tfvc-change-review
description: 'Review TFS or TFVC changes using tf commands. Use when the user asks for code review and provides local paths, folders, pending-change hints such as add, edit, delete, tf diff, TFVC, or TFS, or references a specific changeset such as Changeset 240833, changeset id 240833, cs240833, or 查看 changeset. Focus the review on real TFVC change hunks plus minimal nearby local context, not the entire local file.'
argument-hint: 'Provide one or more TFVC local paths, folders, or a changeset ID to review'
user-invocable: true
---

# TFVC Change Review

Use this skill when the user wants a review of TFVC or TFS changes and provides file paths, folder paths, pending-change hints, or a specific changeset ID.

中文说明：当用户提供 TFVC 或 TFS 工作区中的文件路径、目录路径，明确给出 `add`、`edit`、`delete` 这类变更提示，或直接说 `Changeset 240833`、`查看 changeset 240833`、`review cs240833` 这类请求时，优先基于 TF 命令获取真实变更，再围绕变更块做审查。

## Goal

Review only the actual pending changes, using TFVC diff output as the source of truth, and compare those changed hunks against the smallest amount of local code needed for context.

中文说明：只审查挂起变更本身，不把整个本地文件当作 review 对象。`tf diff` 是变更事实来源，本地代码只读取最小必要上下文。

When the user starts from a changeset number instead of local paths, first resolve the items that belong to that changeset, then review those changed files with the same narrow-scope approach.

中文说明：如果用户是从 changeset 编号出发，而不是先给本地路径，先解析该 changeset 包含哪些文件，再用同样的窄范围方式审查这些改动。

## When to Use

中文说明：以下场景适合触发本 skill。

- The user says `code review` and gives Windows paths from a TFVC workspace.
- The user lists changed files with hints like `add`, `edit`, or `delete`.
- The user asks to review TFS or TFVC changes instead of reviewing the entire local file.
- The repository is not Git-based for the requested change, or TFVC is the intended source of truth.
- The user asks to inspect or review a specific changeset, for example `帮我查看 Changeset 240833`, `review changeset 240833`, `看一下 cs240833`, or `帮我分析 changeset id 240833`.

## Procedure

中文说明：执行顺序应当先锁定用户给的路径，再取 diff，再补最小本地上下文，确认是否有关联 ADO 工作项，最后输出 findings。

0. Before diving into findings, ask the user once whether there is a related ADO work item (User Story or Bug) link or ID associated with this change. Use the `ask_user` tool with a short yes/no/skip style question. Do not block the review if the user says no or skips.
1. Normalize the user input as either a local path, a local folder, or a changeset ID.
2. Resolve `<MainRoot>` from `.github/tfvc-defaults.json` when that file exists, and prefer running TFVC commands from `<MainRoot>` or the nearest mapped ancestor directory of the requested path.
3. If the user supplied file or folder paths, determine pending items with TFVC commands before reading local files broadly.
4. If the user supplied a changeset ID, run `tf changeset <id> /noprompt` first to get the metadata and changed item list.
5. For path-based review, use `tf diff <path> /noprompt` for each changed file to obtain the actual change content.
6. If a direct `tf diff <absolute-path>` fails for a path under `<MainRoot>`, retry from `<MainRoot>` before concluding the file is not mapped or not in source control.
7. For changeset-based review, use the item list from `tf changeset <id> /noprompt`, map each item to the relevant local workspace path when local context is needed, and compare the file at `C<id>` to its previous version with `tf diff <path> /version:C<id>~C<id-1> /format:Unified /noprompt` or another equivalent TFVC diff that isolates that changeset.
8. If the user gives a folder, enumerate pending changes inside that folder first, then diff the concrete files.
9. Read only narrow local context around changed hunks when needed to understand behavior, types, or nearby call sites.
10. Review the diff, not the entire file. Untouched regions are out of scope unless they directly control the changed behavior.
11. Return findings first, ordered by severity, with file and line references pointing to the changed local file when available, or to the TFVC server path when local mapping is unavailable.

中文说明：如果用户给的是目录，先列出该目录下的 pending changes，再逐个文件看 diff；如果用户直接给的是具体文件，直接进入 `tf diff`；如果用户给的是 changeset 编号，先用 `tf changeset <id>` 找出文件清单，再针对该 changeset 的文件逐项看差异。

## Command Rules

中文说明：命令选择遵循下面规则，避免把本地工作区的 TFVC 状态读错。

- Prefer `tf diff <path> /noprompt` for any file explicitly marked as changed.
- Before path-based TFVC commands, prefer setting the working directory to `<MainRoot>` from `.github/tfvc-defaults.json` when it exists locally, or to the nearest mapped ancestor directory of the requested path.
- Prefer `tf status <path> /recursive /format:detailed` to enumerate pending changes under a folder.
- Prefer `tf changeset <id> /noprompt` when the user references a changeset number and you need the authoritative list of changed items.
- For changeset-based review, prefer a TFVC version diff that isolates that changeset for each changed file, such as `tf diff <path> /version:C<id>~C<id-1> /format:Unified /noprompt`, after confirming the local path that maps to the changed item.
- Do not rely on `tf status /user:*` for local workspaces unless a specific workspace is also supplied. In local workspaces, that often reports no pending changes.
- If `tf diff <path>` fails for a path that is under `<MainRoot>`, retry from `<MainRoot>` before treating that as a mapping failure.
- Do not infer alternate local roots such as `C:\source\Main` unless the user explicitly provides them or TFVC mapping output proves that alternate root.
- If `tf diff` reports an `add`, treat the added file content as the entire change set for review.
- If `tf diff` reports an `edit`, review only the modified hunks and fetch minimal local context around those hunks.
- If `tf diff` reports a `delete`, review the removal impact against the surrounding code that referenced the deleted file or symbol.

If `tf changeset <id>` returns server paths outside the currently opened workspace, keep the review scoped to the changed items you can resolve locally and explicitly call out any files that could not be mapped for deeper context.

中文说明：`add` 表示新增文件全文就是变更范围；`edit` 只看 diff 块；`delete` 重点看删除后对引用点或依赖方的影响。按 changeset 审查时，先用 `tf changeset <id>` 拿到权威文件列表，再对每个文件做该 changeset 对应的版本差异。

## ADO Work Item Context

中文说明：在拿到 diff 之后、给出 findings 之前，先确认是否有关联的 ADO 工作项；如果用户提供了 ADO 号或链接，就把工作项的 Description、Acceptance Criteria、Repro Steps 等信息拉进来，与 `.github/kb` 中的项目知识结合，作为 review 的判断依据。

- After collecting the TFVC diff, ask the user with `ask_user`: "Is there a related ADO work item (User Story / Bug) for this change? If yes, please paste the ADO ID or URL; otherwise reply `no` or `skip`." Ask this only once per session.
- If the user provides an ADO ID or URL:
  1. Parse the numeric work item ID from the input.
  2. Default to project `AspenTech SAFe` when the user did not name a different project.
  3. Use `ado-wit_get_work_item` with `expand: "all"` (or at minimum fetch `System.Title`, `System.WorkItemType`, `System.State`, `System.Description`, `Microsoft.VSTS.Common.AcceptanceCriteria`, `Microsoft.VSTS.TCM.ReproSteps`, `System.Tags`, `System.AreaPath`) to read the item.
  4. If the work item type is `User Story` or `Feature`, prioritize Description + Acceptance Criteria. If the type is `Bug`, prioritize Repro Steps + System Info + Acceptance Criteria.
  5. Use `ado-wit_list_work_item_comments` only when the description or AC is too thin to judge the diff intent.
  6. If the work item links to a Parent, briefly fetch the Parent title and description so the review understands the larger story.
- Combine the ADO context with the relevant knowledge base pages under `.github/kb`:
  - Always consult `.github/kb/README.md` as the index.
  - For Aura-related changes, prefer `.github/kb/projects/aura.md` and any matching `.github/kb/pages/*.md` (such as `common-data-grid.md`, `common-flowsheet.md`, `common-form-controls.md`, `common-test-routes.md`).
  - For shared infra changes, prefer `.github/kb/projects/common.md` and the matching shared pages.
- Cross-check the diff against ADO intent:
  - Verify the changed code actually implements every Acceptance Criterion or fully addresses the bug repro.
  - Call out missing AC coverage, partial fixes, or behavior that contradicts the work item.
  - Flag deviations from KB-documented patterns (naming, file layout, data grid usage, form controls, flowsheet conventions, test routes, etc.) and cite the KB page.
- If the user replies `no` or `skip`, proceed with a pure diff-based review and note in the output that no ADO context was used.
- If the work item cannot be fetched (permission, wrong project, deleted), report that explicitly and continue with the diff-only review rather than aborting.

中文说明：拉取到 ADO 信息后，逐条 AC / repro 与 diff 对照，确认实现是否完整、是否偏离 KB 中已沉淀的项目约定；KB 引用请在 findings 里点名具体页面，便于后续追溯。

## Review Scope

中文说明：只评估这次变更直接引入的风险，不顺手扩大成整文件或整模块体检。

- In scope: changed logic, changed templates, changed styles that affect behavior or UI correctness, compatibility with surrounding code, missing validation, regressions introduced by the diff.
- Out of scope: unrelated pre-existing issues elsewhere in the file, untouched sections with no bearing on the diff, broad architecture commentary unless the diff creates the risk.

## Recommended Workflow

中文说明：下面是推荐的落地步骤，可以直接作为实际 review 的操作顺序。

1. Validate each provided path exists locally.
2. Resolve `<MainRoot>` from `.github/tfvc-defaults.json` when available, and run TFVC commands from that root or the nearest mapped ancestor directory.
3. Run TFVC diff on each changed file path.
4. If the diff is ambiguous, read the smallest local slice needed around the changed lines.
5. If a changed symbol interacts with another file, read only that nearby dependency.
6. Produce findings with severity, rationale, and the concrete changed location.
7. If no issues are found, say so explicitly and mention any residual testing gaps.

For changeset-driven requests:

1. Parse the numeric changeset ID from the user request.
2. Run `tf changeset <id> /noprompt`.
3. Use the returned item list as the review scope.
4. For each resolvable local file, inspect the changeset-specific diff and only then read narrow local context.
5. Report any files that could not be mapped locally before finalizing the review.

中文说明：如果某个改动涉及类型、模板绑定、构造参数或调用方，只补一跳相邻上下文，不做大范围扫描。

## Output Expectations

中文说明：输出格式优先是 code review 结论，而不是过程描述。

- Findings first, not a general summary.
- Each finding should explain the concrete regression risk introduced by the diff.
- Reference the changed file path and local line numbers when available.
- If the user asked only whether TFVC diff can be read, answer that directly before expanding into review.
- If the user asked for a changeset by ID, confirm the changeset metadata briefly, then review the changed items rather than asking the user to restate file paths.
- When ADO context was used, add a short `ADO Context` block before the findings that lists the work item ID, type (User Story / Bug), title, and a compact bullet list of ACs or repro steps that the review checked against.
- When findings cite a KB rule, include the KB page path (for example `.github/kb/pages/common-data-grid.md`) so the user can jump back to the source.

## Notes

中文说明：这里记录一些针对当前前端仓库常见改动类型的补充规则。

- For added Aurelia component files, review the new HTML, TS, and LESS together as one feature slice.
- For infrastructure edits, prefer the directly changed class or constructor and nearby call sites instead of scanning the whole file.
- If the user says only `Changeset 240833`, treat that as enough input to start TFVC inspection. Do not ask for local file paths first unless local mapping is required and cannot be inferred.