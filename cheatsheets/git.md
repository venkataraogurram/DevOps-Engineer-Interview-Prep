# Git cheatsheet (annotated)

[← Cheatsheet index](README.md)

Format: command → what it does and when you reach for it.

## Setup

| Command | What it does / when to use |
|---|---|
| `git config --global user.name "X"` / `user.email x@y` | Identity on commits. Use your work email on work repos. |
| `git config --global init.defaultBranch main` | New repos start on `main`. |
| `git config --global pull.rebase true` | `git pull` rebases instead of creating merge commits. Keeps feature branches linear. |
| `git config --global rerere.enabled true` | Remembers how you resolved a conflict and reapplies it next time (long rebases). |
| `git config --global core.autocrlf input` | Normalise line endings to LF on commit. Prevents CRLF breaking shell scripts. |
| `git config --global alias.lg "log --oneline --graph --decorate --all"` | A readable history view as `git lg`. |
| `git config -l --show-origin` | Every setting and which file it comes from. Debug "why is git doing that". |
| `git config --global commit.gpgsign true` + `gpg.format ssh` + `user.signingkey ~/.ssh/id_ed25519.pub` | Sign commits with your SSH key. Banks often require verified commits. |

## Daily work

| Command | What it does / when to use |
|---|---|
| `git clone --depth 1 --branch main URL` | Shallow clone of one branch. Fast CI checkouts when history isn't needed. |
| `git clone --filter=blob:none URL` | Partial clone: full history, file contents fetched on demand. Big repos. |
| `git status -sb` | Short status with branch and ahead/behind. |
| `git add -p` | Stage hunks interactively. Lets you commit only the related changes. |
| `git add -A` | Stage everything including deletions. |
| `git restore --staged f` | Unstage without losing changes. |
| `git restore f` | Discard working-tree changes to a file (irreversible). |
| `git commit -m "feat(api): add retry"` | Commit with a conventional-commit message (`feat`, `fix`, `chore`, `docs`; `!` or `BREAKING CHANGE:` for majors). Drives automated changelogs. |
| `git commit --amend --no-edit` | Fold staged changes into the last commit. Only if that commit isn't pushed. |
| `git commit -S` | Signed commit. |
| `git fetch --all --prune` | Download all remote refs and drop deleted remote branches locally. Do this before deciding anything is "local-only". |
| `git pull --rebase` | Update your branch by replaying your commits on top of the remote. |
| `git push -u origin feat/x` | Push and set upstream so later `git push` just works. |
| `git push --force-with-lease` | Force push that refuses if someone else pushed in between. Use for your own rebased branch, never for shared branches. |
| `git switch -c feat/x` / `git switch -` | Create and switch / go back to the previous branch. |
| `git branch -a` / `git branch -d x` / `git push origin --delete x` | List all / delete local (safe, merged only; `-D` forces) / delete remote. |
| `git branch -m old new` | Rename a branch. |
| `git log --oneline -n 10` | Compact recent history. |
| `git log --since='2 weeks' --author=me` | Filter by time and author. |
| `git log -p f` / `git log --follow f` | Changes to one file with diffs / across renames. |
| `git log -S 'string'` | Commits that added or removed a string ("when did this config value appear"). |
| `git shortlog -sn` | Commit counts per author. |
| `git diff` / `git diff --staged` | Unstaged / staged changes. |
| `git diff main..feat/x --stat` | Files changed between branches. |
| `git diff HEAD~3 -- path/` | Last three commits for a path. |
| `git show HEAD` / `git show abc123:path/f` | A commit's diff / a file's content at a commit (without checking out). |
| `git blame -L 10,20 f` | Who last changed lines 10–20. `-w` ignores whitespace, `-C` follows copied code. |
| `git stash` / `git stash push -m 'wip' -- path` | Shelve changes (all / specific paths) to switch branches cleanly. |
| `git stash list` / `pop` / `apply stash@{1}` / `drop` | Manage stashes; `pop` applies and removes. `-u` includes untracked files. |

## Undo: pick the right one

| You want to… | Command | Notes |
|---|---|---|
| Discard working changes to a file | `git restore f` | Irreversible |
| Unstage a file | `git restore --staged f` | Keeps changes |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` | Re-commit with a better message/split |
| Undo last commit, keep changes unstaged | `git reset --mixed HEAD~1` | Default `reset` |
| Throw away last commit and its changes | `git reset --hard HEAD~1` | Local, unpushed only; data loss |
| Undo a pushed commit safely | `git revert <sha>` | Creates a new inverse commit; the only correct way on shared branches |
| Revert a merge commit | `git revert -m 1 <merge-sha>` | `-m 1` keeps the mainline parent |
| Recover a deleted branch or "lost" commit | `git reflog` → `git switch -c x <sha>` | Reflog keeps ~90 days of where HEAD was |
| Recover a deleted file | `git checkout <sha>~1 -- path` | From the commit before the deletion |
| Fix the author of the last commit | `git commit --amend --author="A <a@b>"` | Unpushed only |

## Branching, merging, rebasing

| Command | What it does / when to use |
|---|---|
| `git merge feat/x` | Merge into the current branch (fast-forward if possible). |
| `git merge --no-ff feat/x` | Always create a merge commit, preserving the branch shape (GitFlow style). |
| `git merge --squash feat/x` | Combine the branch into one staged change, then commit once. Clean history for PRs. |
| `git merge --abort` | Back out of a conflicted merge. |
| `git rebase main` | Replay your branch's commits on top of `main`. Linear history; rewrites SHAs, so only on unpushed or personal branches. |
| `git rebase -i HEAD~5` | Interactive: `pick`, `squash`, `fixup`, `reword`, `drop`, `edit` the last 5 commits. Clean up before opening a PR. |
| `git rebase --continue` / `--abort` / `--skip` | After resolving a conflict / give up / drop the conflicting commit. |
| `git rebase --onto main server feat` | Move the commits between `server` and `feat` onto `main`. Fixes "branched from the wrong base". |
| `git cherry-pick <sha>` / `A^..B` | Copy one commit / a range onto the current branch. Hotfix to a release branch. `-x` records the source SHA in the message. |
| `git merge-base main feat` | Common ancestor. Used by `cr`/PR tools to compute the diff. |
| `git branch --merged main` / `--no-merged` | Branches safe to delete / still unmerged. |
| Conflict markers | `<<<<<<< HEAD` (ours) `=======` (theirs) `>>>>>>>`. `git checkout --ours f` / `--theirs f` to take a side wholesale; `git mergetool` for a GUI. |

Merge vs rebase in one line: merge keeps history and adds a merge commit; rebase linearises and rewrites. Never rebase a branch others have pulled. Squash-merge PRs for a tidy `main`.

## Tags and releases

| Command | What it does / when to use |
|---|---|
| `git tag` | List tags. |
| `git tag -a v1.2.3 -m "release 1.2.3"` | Annotated tag (has author/date/message; what releases should use). |
| `git tag v1.2.3 <sha>` | Tag an older commit. |
| `git push origin v1.2.3` / `git push --tags` | Tags aren't pushed by default. |
| `git tag -d v1.2.3` / `git push origin --delete v1.2.3` | Delete locally / remotely (avoid for published releases). |
| `git describe --tags --always` | Nearest tag plus commit count and SHA, e.g. `v1.2.3-4-gabc1234`. Good build version string. |
| `git log v1.2.2..v1.2.3 --oneline` | Changelog between two releases. |

Semver: MAJOR (breaking) . MINOR (features) . PATCH (fixes). Conventional commits + `semantic-release` automate the bump and changelog.

## Investigating

| Command | What it does / when to use |
|---|---|
| `git bisect start; git bisect bad; git bisect good v1.2` then test and `git bisect good\|bad` | Binary search for the commit that introduced a bug. `git bisect run ./test.sh` automates it; `git bisect reset` to finish. |
| `git log --all --oneline -- path/deleted-file` | Find when a file was deleted (any branch). |
| `git grep -n 'TODO' main` | Search file contents at a ref without checking it out. |
| `git log --diff-filter=D --name-only` | Commits that deleted files. |
| `git fsck --lost-found` | Find dangling commits/blobs after a bad reset. |
| `git count-objects -vH` / `git gc --prune=now` | Repo size / compact. |
| `git ls-files` / `git ls-tree -r HEAD --name-only` | Tracked files / tree at a ref. |
| `git check-ignore -v f` | Which `.gitignore` rule is ignoring a file. |

## Remotes, submodules, worktrees, LFS

| Command | What it does / when to use |
|---|---|
| `git remote -v` / `git remote add upstream URL` / `git remote set-url origin URL` | List / add a second remote (fork workflow) / change a URL (HTTPS→SSH). |
| `git fetch upstream && git rebase upstream/main` | Sync a fork with its upstream. |
| `git submodule add URL path` / `git submodule update --init --recursive` / `git clone --recurse-submodules` | Embed another repo at a pinned commit / initialise after clone / clone with them. Terraform module vendoring sometimes uses this; prefer registry/tags. |
| `git worktree add ../hotfix hotfix/x` / `git worktree list` / `git worktree remove ../hotfix` | Check out a second branch in another directory without stashing. Work on a hotfix while a build runs on your feature branch. |
| `git lfs install` / `git lfs track '*.bin'` / `git lfs ls-files` | Large File Storage for binaries so the repo stays small. |
| `git archive --format=tar.gz -o src.tgz HEAD` | Export a tree without `.git`. Build artifacts from a tag. |
| `git sparse-checkout init --cone && git sparse-checkout set infra/` | Check out only a subdirectory of a monorepo. |

## Hygiene and safety

| Item | What it does / when to use |
|---|---|
| `.gitignore`: `.terraform/ *.tfstate* *.tfplan .env node_modules/ __pycache__/ .DS_Store crash.log` | Keep state, plans, secrets and junk out of the repo. State files contain secrets. |
| `git rm --cached secret.env` | Stop tracking a file but keep it on disk; then add to `.gitignore`. |
| `pre-commit install` with `.pre-commit-config.yaml` | Run `terraform fmt`, `terraform validate`, `tflint`, `gitleaks`, `shellcheck`, `detect-private-key` before every commit. Cheapest place to catch problems. |
| `git filter-repo --path secret.env --invert-paths` | Purge a file from all history after a leak. Then rotate the secret (history on forks/clones still has it), force-push, everyone re-clones. |
| `CODEOWNERS`: `/infra/  @platform-team` | Required reviewers per path. Platform owns `infra/`. |
| Branch protection | Require PR reviews, passing status checks, signed commits; block force-push and deletion on `main`. |

Hooks: `.git/hooks/pre-commit`, `commit-msg`, `pre-push` are local and unversioned, so use the pre-commit framework or Husky to share them. Server-side enforcement lives in branch protection rules.

## Branching strategies, one line each

- **Trunk-based**: short branches (<2 days) merged to `main`; feature flags hide unfinished work; CI on every commit; release tags or branches cut from `main`. Enables continuous delivery.
- **GitHub Flow**: branch → PR → review → merge → deploy from `main`. Simple, works for most services.
- **GitFlow**: `main` + `develop` + `feature/*` + `release/*` + `hotfix/*`. Fits scheduled versioned releases; heavy merge overhead.
- **Release branches**: `release/1.4` for patching a shipped version; cherry-pick fixes from `main`.
- **Environment branches** (dev/staging/prod branches): anti-pattern; promote tested artifacts with environment config, not merges.
