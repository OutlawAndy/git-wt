# git-wt

Create or switch to a git worktree for a branch, set up so a Rails app boots in it.

```sh
git wt feat 1234-login           # new worktree on branch feat/1234-login
git wt --from HEAD feat 1234-v2  # branch off the one you're on
git wt --remove feat 1234-login  # remove that worktree
git wt --remove feat             # remove every feat/ worktree
git wt --remove                  # remove everything under .worktrees/
```

Every worktree lives at `<main checkout>/.worktrees/<type>/<name>`, whichever worktree you run it from. If the branch is already checked out somewhere, you get that worktree instead of a new one.

## Install

```sh
brew install rolemodel/rmr/git-wt
```

Or in a Brewfile. Homebrew won't load a formula from a tap you haven't trusted, and `brew bundle` only trusts what the Brewfile says to:

```ruby
tap "rolemodel/rmr", trusted: true
brew "rolemodel/rmr/git-wt"
```

Then add the `wt` function to `~/.zshrc` (or `~/.bashrc`):

```sh
eval "$(git wt --init zsh)"
```

`wt feat 1234-login` creates the worktree, `cd`s into it, and then runs setup. A script can't change your shell's directory, which is why the function exists. If you interrupt setup, you're still in the worktree.

## What a new worktree gets

- `/.worktrees/` added to `.git/info/exclude`, if it isn't ignored already.
- Copies of the gitignored files a fresh checkout lacks, when they exist: `config/master.key`, `config/credentials/{development,test}.key`, `.env`, `.env.local`, `.env.development.local`, `.env_overrides.rb` and `.vscode/settings.json`.
- A copy-on-write clone of `storage/`. Without it, local Active Storage attachments are missing. On Rails 8 this also gives each worktree its own SQLite databases.
- Your repo's `bin/worktree-setup`, or `bin/setup --skip-server` if there isn't one. It runs with `PORT` set to a fixed port derived from the branch name (3001–3900), so dev servers in different worktrees don't collide.

## Removing worktrees

`--remove` runs `git worktree remove` on each match, so a worktree with uncommitted changes or untracked files is left alone (gitignored files don't count). Each worktree's branch is then deleted with `git branch -d`, which keeps any that aren't merged. `wt --remove` moves you to the main checkout, in case you were standing in a worktree it removed.

## What it doesn't do

Worktrees share whatever Postgres databases `config/database.yml` names. To give each worktree its own, read the database name from an env var in `database.yml` and set it in `bin/worktree-setup`. Make sure the hook is safe to run more than once.

## For agents and scripts

Call `git wt` directly, not the `wt` function. The worktree's absolute path is the only thing written to stdout. With `--no-provision`, the exit status is 3 if the worktree already existed. See `man git-wt`.

## Development

```sh
test/run
```

To release, run the **Release** workflow from the Actions tab and pick a version bump. It bumps `VERSION` in `bin/git-wt`, moves the Unreleased notes in `CHANGELOG.md` under the new version, then tags and publishes a GitHub release. The tap picks up the new release on its own.
