# Changelog

## Unreleased

- A failing `post-checkout` hook no longer aborts `git wt` after the worktree is created
- A gem with no `bin/setup` is provisioned with `bundle install`
- The `bin/dev` hint only shows when the repo has a `bin/dev`

## [v0.2.0] Oct 21, 2026

- `git wt --remove [<type> [<name>]]` removes one worktree, every worktree of a type, or every worktree under `.worktrees/`, then deletes their merged branches

## [v0.1.1] Oct 19, 2026

- `git wt --version`
- Releases run from GitHub Actions

## [v0.1.0] Oct 19, 2026

- Worktrees live at `<main checkout>/.worktrees/<type>/<name>`; a branch already checked out elsewhere is reused
- New branches start from the remote's default branch, or `--from <ref>`
- New worktrees get the gitignored Rails secrets and env files, a clone of `storage/`, and a per-branch `PORT`
- `--init zsh|bash` prints the `wt` function that `cd`s into the worktree

[v0.2.0]: https://github.com/OutlawAndy/git-wt/releases/tag/v0.2.0
[v0.1.1]: https://github.com/OutlawAndy/git-wt/releases/tag/v0.1.1
[v0.1.0]: https://github.com/OutlawAndy/git-wt/releases/tag/v0.1.0
