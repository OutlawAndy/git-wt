# git-wt

<div>
  <img src="https://github.com/OutlawAndy/git-wt/actions/workflows/ci.yml/badge.svg" alt="CI">
  <img src="https://img.shields.io/github/v/release/OutlawAndy/git-wt?label=homebrew&logo=homebrew" alt="Homebrew">
</div>
<br/>

Worktrees that *work*, for humans and agents alike.

```sh
git wt feat 1234-login
```

That creates branch `feat/1234-login` in its own worktree, copies over the secrets and env files git doesn't track, and runs your app's setup. If the branch is already checked out somewhere, you get that worktree instead.

## Installation

```sh
brew install rolemodel/rmr/git-wt
```

or in a Brewfile

```ruby
tap "rolemodel/rmr", trusted: true
brew "rolemodel/rmr/git-wt"
```

> [!NOTE]
> Homebrew won't load a formula from a tap you haven't trusted, and `brew bundle` only trusts the taps your Brewfile says to. That's what `trusted: true` is for.

## Setup

Add the `wt` function to your _~/.zshrc_ (or _~/.bashrc_).

```sh
eval "$(git wt --init zsh)"
```

`wt feat 1234-login` creates the worktree, `cd`s into it, and then runs setup. A script can't change your shell's directory, which is why the function exists. If you interrupt setup, you're still in the worktree.

The same line sets up tab completion of your existing types and worktree names, for both `wt` and `git wt`.

> [!IMPORTANT]
> In zsh, the `eval` line must come after `compinit`. Otherwise `git wt` still completes, but `wt` won't.

## Basic Usage

```sh
wt feat 1234-login           # new worktree on branch feat/1234-login
wt -f HEAD feat 1234-v2      # branch off the one you're on
wt -l                        # list your worktrees
wt -c feat 1234-login        # finish up in the main checkout instead
wt -r feat 1234-login        # remove that worktree
wt -r feat                   # remove every feat/ worktree
wt -r                        # remove everything under .worktrees/
```

Every worktree lives at _&lt;main checkout&gt;/.worktrees/&lt;type&gt;/&lt;name&gt;_, whichever worktree you run it from, so worktrees are always siblings and never nested.

New branches start from your remote's default branch, or from `--from <ref>`. An existing local or remote branch is checked out as is.

## Options

| Option                   | Description                                                                                         |
| ------------------------ | --------------------------------------------------------------------------------------------------- |
| `-f`, `--from <ref>`     | Start a new branch from `<ref>` instead of the default branch. `HEAD` branches off the one you're on |
| `-l`, `--list`           | List every worktree with its branch, `PORT`, path, uncommitted changes and upstream tracking        |
| `-c`, `--checkout`       | Check the branch out in the main checkout and remove its worktree                                   |
| `-r`, `--remove`         | Remove one worktree, every worktree of a type, or all of them                                       |
| `-v`, `--version`        | Print the version                                                                                   |
| `-h`, `--help`           | Print usage                                                                                         |
| `--init <zsh\|bash>`     | Print the `wt` function and tab completion                                                          |
| `--no-provision`         | Create the worktree but skip setup. Exits 3 if the worktree already existed                         |
| `--provision-only`       | Run setup in an existing worktree                                                                   |

See `man git-wt` for the details.

## What a New Worktree Gets

- `/.worktrees/` added to _.git/info/exclude_, if it isn't ignored already.
- Copies of the gitignored files a fresh checkout lacks, when they exist: _config/master.key_, _config/credentials/{development,test}.key_, _.env_, _.env.local_, _.env.development.local_, _.env_overrides.rb_ and _.vscode/settings.json_.
- A copy-on-write clone of _storage/_, so local Active Storage attachments still show up. On Rails 8 this also gives each worktree its own SQLite databases.
- A `PORT` derived from the branch name (3001–3900), so dev servers in different worktrees don't collide.
- Your app's setup, run in the new worktree. See [Hooks](#hooks).

Your repo's `post-checkout` hook, if it has one, runs before all of this. If it fails, **git-wt** warns and carries on, since the worktree already exists.

## Hooks

**git-wt** doesn't decide what "set up" means for your app. Your repo does, with two optional executables.

| Hook                   | Runs                                              | Fallback                                                                     |
| ---------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------- |
| `bin/worktree-setup`    | in a new worktree, with `PORT` set                | `bin/setup --skip-server`, or `bundle install` for a gem with neither          |
| `bin/worktree-teardown` | in a worktree about to be removed by `-r` or `-c` | nothing                                                                      |

> [!IMPORTANT]
> `bin/worktree-setup` must be safe to run more than once. `PORT` is only set for the hook, so persist it wherever your app reads its environment from.

Teardown never runs for a worktree with uncommitted or untracked files, since that worktree stays put. If teardown fails, the worktree is kept too.

## Removing Worktrees

`--remove` tears down each matching worktree and runs `git worktree remove` on it. A worktree with uncommitted changes or untracked files is left alone (gitignored files don't count). Each removed worktree's branch is then deleted with `git branch -d`, which keeps any that aren't merged.

`--checkout` is for when you'd rather finish the work in the main checkout. It checks the branch out there and removes the worktree. It refuses, changing nothing, if the worktree has uncommitted or untracked files, if teardown fails, or if the checkout would overwrite local changes in the main checkout. The branch is kept.

Either way, `wt` moves you to the main checkout, in case you were standing in a worktree it just removed.

## Per-Worktree Databases

Out of the box, worktrees share whatever Postgres databases _config/database.yml_ names. With the [hooks](#hooks), each worktree can have its own.

Read the database name from an env var, under `development:` only.

```yaml
development:
  <<: *default
  database: <%= ENV.fetch("DEV_DATABASE", "app_development") %>
```

Then, in `bin/worktree-setup`, pick a name, set `DEV_DATABASE` wherever your app loads its environment from, and create the database. In `bin/worktree-teardown`, drop it.

```sh
dropdb --if-exists --force "$DEV_DATABASE"
```

> [!WARNING]
> `DATABASE_URL` is a special env variable that should be avoided in local development. When present, it overrides _config/database.yml_ regardless of `Rails.env`.

> [!TIP]
> Rails doesn't read _.env_ files itself. If your app uses [dotenv], set `DEV_DATABASE` in _.env.development.local_, which dotenv only loads in development. **git-wt** copies that file from the main checkout, so have your hook change the one line rather than overwrite the file.

On Postgres 18, the new database can start as a copy of your main one, almost instantly and with no extra disk on a copy-on-write filesystem like APFS.

```sql
SET file_copy_method = clone;
CREATE DATABASE app_dev_1234_login TEMPLATE app_development STRATEGY FILE_COPY;
```

Older versions accept the same `CREATE DATABASE` without the `SET`, but make a full copy. In every version, it fails while anything is connected to the template, like the main checkout's dev server.

## For Agents and Scripts

Call `git wt` directly, not the `wt` function. The worktree's absolute path is the only thing written to stdout, so this works:

```sh
cd "$(git wt fix 99-typo)" && $EDITOR .
```

With `--no-provision`, the exit status is 3 when there's nothing to set up: the worktree already existed, or `--remove` or `--checkout` succeeded.

## Development

Run the test suite with `test/run`.

To release, run the **Release** workflow from the Actions tab and pick a version bump. It bumps `VERSION` in _bin/git-wt_, moves the Unreleased notes in _CHANGELOG.md_ under the new version, then tags and publishes a GitHub release. The tap picks up the new release on its own.

## Acknowledgments

**git-wt** is [MIT-licensed](LICENSE), open-source software.

[dotenv]: https://github.com/bkeepers/dotenv
