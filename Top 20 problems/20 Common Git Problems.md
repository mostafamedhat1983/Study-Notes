---
tags:
  - Git
  - Troubleshooting
---

Knowing these problems helps you recover safely and understand Git failures.

---

## Main idea

Most Git problems involve working-tree state, history, remotes, authentication, or ignored files.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Not a Git repository

- **Why it happens:** The current directory is outside a repository.
- **Quick sign:** Git cannot find .git.
- **Solution:** Navigate to the intended clone; initialize only when creating a new repository.
- **Useful commands:** `pwd`, `git rev-parse --show-toplevel`

---

## 2. Wrong commit identity

- **Why it happens:** user.name or user.email is missing or incorrect.
- **Quick sign:** Git asks who you are or commits show the wrong author.
- **Solution:** Set repository-local identity where appropriate; changing config does not rewrite existing commits.
- **Useful commands:** `git config --show-origin --get user.email`, `git config user.name 'Your Name'`

---

## 3. HTTPS authentication fails

- **Why it happens:** Credentials are expired, incorrectly cached, or insufficiently scoped.
- **Quick sign:** Authentication fails during fetch or push.
- **Solution:** Verify the remote and refresh credentials using the host's supported authentication method; never embed tokens in URLs.
- **Useful commands:** `git remote -v`, `git config --get credential.helper`

---

## 4. SSH permission denied

- **Why it happens:** The wrong key is offered or the public key lacks access.
- **Quick sign:** Permission denied (publickey).
- **Solution:** Inspect the agent and SSH configuration; verify the account and repository permissions.
- **Useful commands:** `ssh-add -l`, `ssh -T git@github.com`, `ssh -vT git@github.com`

---

## 5. Repository not found

- **Why it happens:** The remote URL is wrong or the account lacks access.
- **Quick sign:** Clone or fetch reports repository not found.
- **Solution:** Verify owner, repository name, and access before changing the URL.
- **Useful commands:** `git remote -v`, `git ls-remote origin`

---

## 6. Push rejected as non-fast-forward

- **Why it happens:** The remote contains commits absent from the local branch.
- **Quick sign:** Push is rejected with fetch first.
- **Solution:** Fetch, inspect divergence, then merge or rebase according to team policy.
- **Useful commands:** `git fetch origin`, `git log --oneline --graph --all -15`

---

## 7. Merge conflicts

- **Why it happens:** Git cannot automatically reconcile overlapping changes.
- **Quick sign:** Unmerged paths and conflict markers appear.
- **Solution:** Resolve each file, stage it, test the result, then finish the merge; abort if necessary.
- **Useful commands:** `git status`, `git diff --name-only --diff-filter=U`, `git merge --abort`

---

## 8. Rebase stops with conflicts

- **Why it happens:** A replayed commit conflicts with the new base.
- **Quick sign:** Git reports a rebase in progress.
- **Solution:** Resolve and stage changes, then continue; abort to return to the pre-rebase state.
- **Useful commands:** `git rebase --continue`, `git rebase --abort`, `git status`

---

## 9. Local changes block checkout or merge

- **Why it happens:** The operation would overwrite uncommitted changes.
- **Quick sign:** Git asks you to commit or stash changes.
- **Solution:** Inspect the changes, then commit or stash deliberately; include untracked files if needed.
- **Useful commands:** `git diff`, `git stash push -u -m 'before branch switch'`

---

## 10. Detached HEAD

- **Why it happens:** A commit or tag was checked out instead of a branch.
- **Quick sign:** HEAD is not attached to a branch.
- **Solution:** Create a branch before leaving if the detached commits should be retained.
- **Useful commands:** `git status`, `git switch -c rescue-work`

---

## 11. Commit made on the wrong branch

- **Why it happens:** Work began without checking the active branch.
- **Quick sign:** A commit appears in the wrong branch history.
- **Solution:** Preserve the commit on a branch, cherry-pick where needed, and use revert for shared-history corrections.
- **Useful commands:** `git branch --show-current`, `git branch rescue-work`, `git cherry-pick <commit>`

---

## 12. Accidentally lost a local commit

- **Why it happens:** Reset, rebase, or branch deletion removed the visible reference.
- **Quick sign:** Expected work disappears from git log.
- **Solution:** Inspect the local reflog and create a recovery branch promptly; reflogs are not permanent backups.
- **Useful commands:** `git reflog`, `git branch recovered-work <commit>`

---

## 13. Need to undo a published commit

- **Why it happens:** A bad change was already shared.
- **Quick sign:** Teammates or deployment use the faulty commit.
- **Solution:** Prefer a new revert commit over rewriting shared history; merge reverts require mainline consideration.
- **Useful commands:** `git show <commit>`, `git revert <commit>`

---

## 14. .gitignore does not ignore a file

- **Why it happens:** The file is already tracked or the pattern does not match.
- **Quick sign:** Ignored-looking files still appear as tracked changes.
- **Solution:** Check the rule; if appropriate, remove only the index entry and commit that change.
- **Useful commands:** `git ls-files path/to/file`, `git check-ignore -v --no-index path/to/file`, `git rm --cached -- path/to/file`

---

## 15. Secret committed to Git

- **Why it happens:** Credentials were staged or stored in tracked configuration.
- **Quick sign:** A token or password appears in history.
- **Solution:** Revoke or rotate first; coordinate history cleanup and prevent recurrence. Deleting the latest file does not erase history.
- **Useful commands:** `git status`, `git log --all -- path/to/file`

---

## 16. Large file rejected

- **Why it happens:** A file exceeds the hosting service's size policy.
- **Quick sign:** A server hook rejects the push.
- **Solution:** Use approved artifact storage or Git LFS; previously committed large blobs may require coordinated history cleanup.
- **Useful commands:** `git lfs version`, `git lfs track '*.bin'`, `git status`

---

## 17. Line endings change entire files

- **Why it happens:** Editors or Git settings convert LF and CRLF inconsistently.
- **Quick sign:** Nearly every line appears modified.
- **Solution:** Agree on .gitattributes rules and review any renormalization separately.
- **Useful commands:** `git ls-files --eol`, `git diff --ignore-space-at-eol`, `git config --get core.autocrlf`

---

## 18. No upstream branch configured

- **Why it happens:** The local branch has no tracking branch.
- **Quick sign:** Plain pull or push asks for a remote branch.
- **Solution:** Verify the intended remote and set tracking explicitly.
- **Useful commands:** `git branch -vv`, `git push -u origin HEAD`

---

## 19. Stale index lock

- **Why it happens:** Another Git operation is running or an interrupted one left a lock.
- **Quick sign:** Git reports index.lock already exists.
- **Solution:** Check running operations first; remove only a confirmed stale lock.
- **Useful commands:** `ps -ef`, `git rev-parse --git-path index.lock`

---

## 20. Submodule directory is empty or wrong

- **Why it happens:** Submodules were not initialized or point to a different recorded commit.
- **Quick sign:** Required nested code is missing.
- **Solution:** Inspect local submodule changes before initializing or updating to the parent's recorded revision.
- **Useful commands:** `git submodule status`, `git submodule update --init --recursive`

---

## Common mistakes

- using reset --hard without preserving work
- force-pushing shared branches without coordination
- treating .gitignore as a secret-removal tool
- resolving conflicts without testing

---

## Good practices

- inspect status and diff before committing
- fetch before diagnosing divergence
- preserve recovery branches before history changes
- use revert for shared-history corrections
- rotate exposed credentials immediately

---

## Must memorize

```bash
git status
git diff
git diff --staged
git branch -vv
git remote -v
git fetch origin
git log --oneline --graph --all -20
git reflog
git stash list
```

---

## Key ideas

- Working tree, index, and commits are different layers.
- Fetch updates remote-tracking information without merging your work.
- Preserve work before resetting or rewriting history.
- Reflog recovery is local and time-limited.
