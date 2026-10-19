# Changelog

## [v0.1.1] Oct 19, 2026

- `git wt --version`
- Releases run from GitHub Actions

## [v0.1.0] Oct 19, 2026

- Worktrees live at `<main checkout>/.worktrees/<type>/<name>`; a branch already checked out elsewhere is reused
- New branches start from the remote's default branch, or `--from <ref>`
- New worktrees get the gitignored Rails secrets and env files, a clone of `storage/`, and a per-branch `PORT`
- `--init zsh|bash` prints the `wt` function that `cd`s into the worktree

[v0.1.1]: https://github.com/OutlawAndy/git-wt/releases/tag/v0.1.1
[v0.1.0]: https://github.com/OutlawAndy/git-wt/releases/tag/v0.1.0
