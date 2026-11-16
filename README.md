# git-wt

Create or switch to a git worktree for a branch, set up so a Rails app boots in it.

```sh
git wt feat 1234-login           # new worktree on branch feat/1234-login
git wt --from HEAD feat 1234-v2  # branch off the one you're on
git wt --remove feat 1234-login  # remove that worktree
git wt --remove feat             # remove every feat/ worktree
git wt --remove                  # remove everything under .worktrees/
git wt -c feat 1234-login        # check it out in the main checkout instead
git wt --list                    # every worktree, its PORT and uncommitted changes
```

Options have one-letter forms: `-f` is `--from`, `-r` is `--remove`, `-l` is `--list`, `-c` is `--checkout`, `-v` is `--version` and `-h` is `--help`.

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

That also sets up tab completion of existing types and worktree names for both `wt` and `git wt`. In zsh, put it after `compinit`.

`wt feat 1234-login` creates the worktree, `cd`s into it, and then runs setup. A script can't change your shell's directory, which is why the function exists. If you interrupt setup, you're still in the worktree.

## What a new worktree gets

- `/.worktrees/` added to `.git/info/exclude`, if it isn't ignored already.
- Copies of the gitignored files a fresh checkout lacks, when they exist: `config/master.key`, `config/credentials/{development,test}.key`, `.env`, `.env.local`, `.env.development.local`, `.env_overrides.rb` and `.vscode/settings.json`.
- A copy-on-write clone of `storage/`. Without it, local Active Storage attachments are missing. On Rails 8 this also gives each worktree its own SQLite databases.
- Your repo's `bin/worktree-setup`, or `bin/setup --skip-server` if there isn't one, or `bundle install` for a gem with neither. It runs with `PORT` set to a fixed port derived from the branch name (3001–3900), so dev servers in different worktrees don't collide.

Your repo's `post-checkout` hook, if it has one, runs before all of this. If it fails, `git wt` warns and carries on, since the worktree already exists.

## Removing worktrees

Before removing a worktree, `git wt` runs your repo's `bin/worktree-teardown` in it, if there is one, so the hook can undo what setup made, like dropping that worktree's database. `--remove` then runs `git worktree remove` on each match. A worktree with uncommitted changes or untracked files (gitignored files don't count), or whose teardown fails, is left alone, and its teardown never runs. Each worktree's branch is then deleted with `git branch -d`, which keeps any that aren't merged. `wt --remove` moves you to the main checkout, in case you were standing in a worktree it removed.

`--checkout` (`-c`) moves a worktree's branch into the main checkout and removes the worktree, for when you'd rather finish the work there. It runs the teardown hook too. It refuses, changing nothing, if the worktree has uncommitted or untracked files, if teardown fails, or if the main checkout has local changes the checkout would overwrite. The branch is kept.

## What it doesn't do

Worktrees share whatever Postgres databases `config/database.yml` names. To give each worktree its own, read the name from an env var under `development:` in `database.yml`, for example `database: <%= ENV.fetch("DEV_DATABASE", "app_development") %>`. Then:

- In `bin/worktree-setup`, set the var in `.env.development.local` and create the database. Make sure the hook is safe to run more than once.
- In `bin/worktree-teardown`, drop the database: `dropdb --if-exists --force "$DEV_DATABASE"`.

Avoid `DATABASE_URL` for this: Rails applies it to the test environment too. On Postgres 18, `CREATE DATABASE … TEMPLATE app_development STRATEGY FILE_COPY` after `SET file_copy_method = clone` copies the main database almost instantly on a copy-on-write filesystem like APFS. It fails while anything is connected to the template, such as the main checkout's dev server.

## For agents and scripts

Call `git wt` directly, not the `wt` function. The worktree's absolute path is the only thing written to stdout. With `--no-provision`, the exit status is 3 if the worktree already existed. See `man git-wt`.

## Development

```sh
test/run
```

To release, run the **Release** workflow from the Actions tab and pick a version bump. It bumps `VERSION` in `bin/git-wt`, moves the Unreleased notes in `CHANGELOG.md` under the new version, then tags and publishes a GitHub release. The tap picks up the new release on its own.
