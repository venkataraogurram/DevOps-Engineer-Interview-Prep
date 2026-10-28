# Git cheatsheet

[← Cheatsheet index](README.md)

## Setup

```bash
git config --global user.name "X"; git config --global user.email x@y; git config --global init.defaultBranch main
git config --global pull.rebase true; git config --global rerere.enabled true; git config --global core.autocrlf input
git config --global alias.lg "log --oneline --graph --decorate --all"; git config -l --show-origin
git config --global commit.gpgsign true; git config --global gpg.format ssh; git config --global user.signingkey ~/.ssh/id_ed25519.pub
```

## Daily

```bash
git clone --depth 1 --branch main URL; git clone --filter=blob:none URL     # fast clones
git status -sb; git add -p; git add -A; git restore --staged f; git restore f
git commit -m "feat(api): add retry"; git commit --amend --no-edit (unpushed only); git commit -S
git fetch --all --prune; git pull --rebase; git push -u origin feat/x; git push --force-with-lease (own branch only)
git switch -c feat/x; git switch -; git branch -a; git branch -d x; git push origin --delete x; git branch -m old new
git log --oneline -n 10; git log --since='2 weeks' --author=me; git log -p f; git log -S 'string' ; git log --follow f; git shortlog -sn
git diff; git diff --staged; git diff main..feat/x --stat; git diff HEAD~3 -- path/
git show HEAD; git show abc123:path/f; git blame -L 10,20 f; git blame -w -C f
git stash; git stash push -m 'wip' -- path; git stash list; git stash pop; git stash apply stash@{1}; git stash drop; git stash -u
```

## Undo (pick the right one)

| Want | Command |
|---|---|
| Discard working changes to a file | `git restore f` |
| Unstage | `git restore --staged f` |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit, keep changes unstaged | `git reset --mixed HEAD~1` |
| Throw away last commit and changes (local only) | `git reset --hard HEAD~1` |
| Undo a pushed commit safely | `git revert <sha>` (new commit) |
| Revert a merge commit | `git revert -m 1 <merge-sha>` |
| Recover deleted branch / lost commit | `git reflog` → `git switch -c x <sha>` |
| Recover a deleted file | `git checkout <sha>~1 -- path` |
| Fix wrong commit author | `git commit --amend --author="A <a@b>"` |

## Branching, merging, rebasing

```bash
git merge feat/x; git merge --no-ff feat/x; git merge --squash feat/x; git merge --abort
git rebase main; git rebase -i HEAD~5 (pick/squash/fixup/reword/drop/edit); git rebase --continue|--abort|--skip
git rebase --onto main server feat     # move range
git cherry-pick <sha>; git cherry-pick A^..B; git cherry-pick -x <sha> (records source)
git merge-base main feat; git branch --merged main; git branch --no-merged
git rerere                              # reuse recorded conflict resolutions
# conflict markers: <<<<<<< HEAD (ours) ======= (theirs) >>>>>>> ; git checkout --ours|--theirs f; git mergetool
```

Merge vs rebase: merge keeps history and a merge commit; rebase rewrites your commits on top of target (linear). Never rebase shared branches. Squash-merge PRs for a clean main.

## Tags and releases

```bash
git tag; git tag -a v1.2.3 -m "release 1.2.3"; git tag v1.2.3 <sha>; git push origin v1.2.3; git push --tags
git tag -d v1.2.3; git push origin --delete v1.2.3; git describe --tags --always; git checkout v1.2.3
git log v1.2.2..v1.2.3 --oneline     # changelog
```

Semver: MAJOR.MINOR.PATCH; conventional commits (`feat:`, `fix:`, `chore:`, `BREAKING CHANGE:`) → semantic-release automates versions.

## Investigate

```bash
git bisect start; git bisect bad; git bisect good v1.2; (test) git bisect good|bad; git bisect run ./test.sh; git bisect reset
git log --all --oneline -- path/deleted-file; git grep -n 'TODO' main
git fsck --lost-found; git count-objects -vH; git gc --prune=now
git ls-files; git ls-tree -r HEAD --name-only; git check-ignore -v f
git whatchanged -n 5; git log --diff-filter=D --name-only
```

## Remotes, submodules, worktrees, LFS

```bash
git remote -v; git remote add upstream URL; git remote set-url origin URL; git fetch upstream; git rebase upstream/main
git submodule add URL path; git submodule update --init --recursive; git clone --recurse-submodules
git worktree add ../hotfix hotfix/x; git worktree list; git worktree remove ../hotfix
git lfs install; git lfs track '*.bin'; git lfs ls-files
git archive --format=tar.gz -o src.tgz HEAD
git sparse-checkout init --cone; git sparse-checkout set infra/
```

## Hygiene

```bash
.gitignore: .terraform/ *.tfstate* *.tfplan .env node_modules/ __pycache__/ .DS_Store crash.log
git rm --cached secret.env            # stop tracking, keep file
pre-commit install; .pre-commit-config.yaml (terraform_fmt, terraform_validate, tflint, gitleaks, shellcheck, detect-private-key)
git filter-repo --path secret.env --invert-paths   # purge from history (then rotate the secret, force-push, everyone re-clones)
CODEOWNERS: /infra/  @platform-team
```

## Hooks

`.git/hooks/pre-commit`, `commit-msg`, `pre-push` (local, not versioned → use pre-commit framework or Husky). Server-side: pre-receive/update (branch protection does this on GitHub/GitLab).

## Branching strategies in one line each

- **Trunk-based**: short branches (<2 days) → main; feature flags; CI on every commit; release tags/branches cut from main.
- **GitHub Flow**: branch → PR → review → merge → deploy main.
- **GitFlow**: main + develop + feature/* + release/* + hotfix/*; versioned releases; heavier.
- **Environment branches** (dev/staging/prod): anti-pattern; promote artifacts not branches.
- **Release branches**: `release/1.4` for patching a shipped version; cherry-pick fixes from main.
