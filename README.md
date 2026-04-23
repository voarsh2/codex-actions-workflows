# codex-actions-workflows

Reusable GitHub Actions workflows that wrap [`openai/codex-action`](https://github.com/openai/codex-action) into a comment-driven actor flow.

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
- `CODEX_RESPONSES_API_ENDPOINT`: full Responses API URL, for example `https://example.com/openai/v1/responses`.

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
- `issue_number`

Context flags:

- `is_pull_request`
- `pull_request_number`

Optional:

- `bot_branch_prefix`
- `model`
- `effort`
- `dry_run`
- `allow_users`
- `deny_untrusted`

Defaults:

- `bot_branch_prefix`: `codex/issue`
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
   - `CODEX_RESPONSES_API_ENDPOINT`
3. Copy [`examples/use-in-your-repo.issue_comment.yml`](./examples/use-in-your-repo.issue_comment.yml) into the target repo as `.github/workflows/codex.yml`.
4. Replace the `uses:` line with your actual reusable workflow repo and ref.
5. Commit the workflow in the target repo.
6. Summon Codex in an issue or PR comment with `@codex ...` or `/codex ...`.

The example workflow handles both normal issue comments and PR comments through the `issue_comment` event.

## Behavior notes

- PR writeback is intentionally limited to same-repo branches in v1.
- Fork PRs get a safe refusal comment instead of an unreliable push attempt.
- Issue-triggered runs reuse `codex/issue-<number>` by default so repeated commands do not create duplicate PRs.
- The workflow expects the upstream endpoint to be compatible with `openai/codex-action` and its `responses-api-endpoint` override.
