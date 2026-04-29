# codex-actions-workflows

Reusable GitHub Actions workflows that wrap direct `codex exec` usage into a comment-driven actor flow.

## What v1 does

- Responds to `@codex ...` or `/codex ...` on issue comments and PR comments.
- Ignores comments that do not summon Codex.
- Ignores bot-authored comments to avoid loops.
- Only acts for trusted collaborators by default: `OWNER`, `MEMBER`, or `COLLABORATOR`.
- On same-repo PRs, checks out the PR branch, runs Codex, and commits/pushes changes back to that branch.
- On normal issues, creates or reuses a deterministic bot branch, runs Codex, commits/pushes changes, and opens or updates a PR.
- Posts a normal top-level comment with the result.

## Repo layout

- [`.github/workflows/codex_actor.yml`](./.github/workflows/codex_actor.yml): reusable workflow invoked with `workflow_call`.
- [`templates/issue_comment_caller.yml`](./templates/issue_comment_caller.yml): thin caller workflow for `issue_comment`.
- [`templates/review_comment_caller.yml`](./templates/review_comment_caller.yml): thin caller workflow for `pull_request_review_comment`.

## Required caller secrets

The reusable workflow expects these secret names in the target project:

- `CODEX_API_KEY`: bearer key for your Claw Bay or other Responses-compatible endpoint.
- `CODEX_BASE_URL`: preferred provider base URL, for example `https://api.example.com/backend-api/codex`.

Use repository secrets or environment secrets in the caller repo. The reusable workflow reads `${{ secrets.* }}` from the calling workflow context.

## Required caller permissions

Caller workflows should grant:

- `contents: write`
- `issues: write`
- `pull-requests: write`

## Reusable workflow inputs

Required:

- `event_name`
- `repository_owner`
- `repository_name`
- `repository_full_name`
- `default_branch`
- `actor`
- `sender_type`
- `author_association`
- `comment_body`
- `comment_id`
- `issue_number`

Context flags:

- `is_pull_request`
- `pull_request_number`

Optional:

- `bot_branch_prefix`
- `model_provider`
- `model`
- `effort`
- `codex_version`
- `history_comment_limit`
- `history_max_comment_chars`
- `dry_run`
- `allow_users`
- `deny_untrusted`

Defaults:

- `bot_branch_prefix`: `codex/issue`
- `model_provider`: `theclawbay`
- `model`: `gpt-5.5`
- `effort`: `medium`
- `codex_version`: `latest`
- `history_comment_limit`: `20`
- `history_max_comment_chars`: `2000`
- `dry_run`: `false`
- `deny_untrusted`: `true`

## Example caller setup

Copy one of the templates in [`templates`](./templates) into a target repo under `.github/workflows/`, then replace:

- `YOUR_ORG`
- `YOUR_REUSABLE_REF`
- the secret names if your target repo uses different secret names

If you want a more literal copy-paste starting point, use [`examples/use-in-your-repo.issue_comment.yml`](./examples/use-in-your-repo.issue_comment.yml) in the target repo and only change the `uses:` repo/ref plus any optional inputs.

## Use in a target repo

1. Push this repo to GitHub so other repos can call the reusable workflow by `owner/repo/.github/workflows/...@ref`.
2. In the target repo, add repository or environment secrets:
   - `CODEX_API_KEY`
   - `CODEX_BASE_URL`
3. Copy [`examples/use-in-your-repo.issue_comment.yml`](./examples/use-in-your-repo.issue_comment.yml) into the target repo as `.github/workflows/codex.yml`.
4. Replace the `uses:` line with your actual reusable workflow repo and ref.
5. Commit the workflow in the target repo.
6. Summon Codex in an issue or PR comment with `@codex ...` or `/codex ...`.

The example workflow handles both normal issue comments and PR comments through the `issue_comment` event.

## Behavior notes

- PR writeback is intentionally limited to same-repo branches in v1.
- Fork PRs get a safe refusal comment instead of an unreliable push attempt.
- Issue-triggered runs reuse `codex/issue-<number>` by default so repeated commands do not create duplicate PRs.
- The workflow writes a minimal temporary `config.toml` for `codex exec` at runtime instead of relying on `openai/codex-action`.
- On Linux runners, the workflow prepares the modern Codex sandbox path by enabling unprivileged user namespaces and clearing Ubuntu's AppArmor user-namespace restriction when present. This follows the same general hosted-runner fix used by `openai/codex-action` and avoids relying on Codex's deprecated legacy Landlock fallback.
- Prompt context includes bounded prior conversation history.
- Human comments are included by default; bot comments are only re-ingested if they contain this workflow's hidden marker.
- Historical human comments have leading `@codex` or `/codex` summon syntax stripped before they are added to prompt context.
- Codex returns structured output so the workflow can use a model-authored commit subject while still owning `git commit` and `git push`.
- Each run creates one progress comment that the workflow updates in place until the final result replaces it.
- While `codex exec` is running, the workflow can harvest early `todo_list`, command, file-change, and agent-message updates from `codex exec --json` and surface a live task checklist or latest status in that per-run progress comment.
- Final result comments include workflow metadata such as duration and a run link.
- Codex can optionally return a short structured `task_plan` as a fallback or final plan summary, but the workflow still owns GitHub comment updates.
- The workflow registers repo-local runtime artifacts such as `.codex` and Python bytecode caches in `.git/info/exclude` so normal `git status` and `git add` honor the repository's own ignore rules plus workflow-local junk without mutating tracked `.gitignore` files.
