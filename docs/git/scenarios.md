# 🌳 Git — Scenario-Based Interview Questions

---

## 🟢 Branching, Merging & Conflicts

---

**Q1. [L1] You started coding directly on `main` and made three commits. You realize this work belongs on a feature branch and you haven't pushed yet. How do you move those commits to a new branch and clean up `main`?**

> *What the interviewer is testing:* Basic branch mechanics, understanding that branches are just movable pointers to commits.

**Answer:**
A branch in Git is just a pointer to a commit, so "moving" commits is really just moving pointers around.

1. **Create the feature branch from where you are right now** — this captures your three commits on the new branch:
   ```bash
   git branch feature/my-work
   ```
2. **Reset `main` back to where the remote is** — your local `main` still points at the third commit; rewind it:
   ```bash
   git reset --hard origin/main
   ```
3. **Switch to the feature branch and continue:**
   ```bash
   git checkout feature/my-work
   ```

The order matters: create the new branch *before* you reset, otherwise the commits become unreachable from any branch (they'd still be in `reflog` for ~90 days, but don't rely on that). Since you never pushed, no one else is affected.

---

**Q2. [L1] What is the difference between `git clone` and `git fork`, and when would you use each in practice?**

> *What the interviewer is testing:* Understanding of remote workflows and contribution models.

**Answer:**
`git clone` is a Git command — `git fork` is **not** a Git command, it's a GitHub/GitLab feature.

- **Clone** (`git clone <url>`) — downloads a copy of a repo to your laptop. You get the full history and a remote called `origin` pointing back to wherever you cloned from. Use it when you have push access to the repo.
- **Fork** — creates a server-side copy of the repo under your own account on GitHub/GitLab. Then you clone *your fork* locally. Use it when you don't have push access to the original (typical for open-source contributions).

The contribution flow for an open-source project is: fork on GitHub → clone your fork → add the original repo as `upstream` → push branches to your fork → open a PR from your fork to the original. For an internal company repo where everyone has write access, just clone directly and push branches.

---

**Q3. [L2] You ran `git merge` and Git stopped with conflicts in three files. Walk me through exactly how you resolve them, and explain what the conflict markers mean.**

> *What the interviewer is testing:* Practical conflict resolution, understanding of three-way merge.

**Answer:**
Git stopped because it couldn't auto-merge — both branches changed the same lines.

1. **See what's conflicted:**
   ```bash
   git status   # files marked "both modified"
   ```
2. **Open each file.** You'll see markers like:
   ```
   <<<<<<< HEAD
   your version (current branch)
   =======
   their version (incoming branch)
   >>>>>>> feature/x
   ```
   `HEAD` is what's on the branch you're merging *into*. The lines below `=======` are from the branch you're merging *in*. The `|||||||` (if `merge.conflictstyle = diff3` is set) shows the common ancestor — extremely useful for understanding intent.
3. **Edit the file** to the correct final state and remove all `<<<`, `===`, `>>>` markers. Don't just pick a side blindly — re-read both intents.
4. **Mark resolved and finish:**
   ```bash
   git add <file>
   git merge --continue   # or: git commit
   ```
5. If you panic, `git merge --abort` puts you back where you started.

For repeated conflicts on the same hunk in long-lived branches, enable `git rerere` so Git remembers your resolution next time.

---

**Q4. [L2] When should you use `git rebase` instead of `git merge`, and what is the one rule you must never break?**

> *What the interviewer is testing:* Workflow tradeoffs, understanding the danger of rewriting shared history.

**Answer:**
Both integrate changes from one branch into another, but they produce very different histories.

- **Merge** preserves the actual branching history (creates a merge commit). Good for `main`/release branches where the history of *when things converged* matters.
- **Rebase** replays your commits on top of the target branch, producing a linear history with no merge commits. Good for cleaning up your local feature branch *before* opening a PR — it makes review easier and `git log` readable.

**The one rule: never rebase commits that have been pushed and that other people are working on.** Rebase rewrites commit hashes. If a teammate has pulled the old commits and you force-push rewritten ones, their next pull is a mess and they may re-introduce the old commits. Rebase your private branch all you want; never rebase shared branches like `main` or `develop`.

Common workflow: rebase your feature branch onto latest `main` (`git pull --rebase origin main`), squash interactively (`git rebase -i`), force-push to your *own* feature branch, then merge the PR.

---

## 🔵 Undo, Recovery & History Rewriting

---

**Q5. [L1] You have local changes in your working directory that you don't want anymore. How do you discard them, and what's the difference between discarding tracked vs untracked changes?**

> *What the interviewer is testing:* Understanding of the working tree, staging area, and untracked files.

**Answer:**
Different commands for different states — and one of them is destructive, so know which:

- **Modified tracked files (not staged):** `git restore <file>` (or older syntax `git checkout -- <file>`) reverts the file to what's in `HEAD`.
- **Staged changes:** `git restore --staged <file>` unstages, leaving the modification in the working tree. Add another `git restore <file>` to discard it too.
- **Untracked (new) files:** `git restore` won't touch them — Git doesn't know they exist. Use `git clean -fd` to delete them. Run `git clean -nd` first to preview what will be deleted.
- **Nuke everything back to a clean checkout:** `git reset --hard HEAD && git clean -fd`.

These are destructive — there's no undo for working-tree changes that were never committed. When unsure, `git stash` first; you can drop the stash later if you don't need it.

---

**Q6. [L1] You added `config.yaml` to `.gitignore` but Git is still tracking it and showing it as modified. Why, and how do you fix it?**

> *What the interviewer is testing:* Understanding that `.gitignore` only affects untracked files.

**Answer:**
`.gitignore` only prevents *new* files from being tracked. It does **not** untrack files that are already in the repo. Git is still watching `config.yaml` because it was committed before the ignore rule existed.

Fix it by removing the file from the index while keeping it on disk:

```bash
git rm --cached config.yaml
git commit -m "Stop tracking config.yaml"
```

The `--cached` flag is critical — without it, `git rm` deletes the local file too. After the commit, `.gitignore` will keep it out going forward.

For a whole directory: `git rm -r --cached path/to/dir`. To re-check what Git is currently tracking that *should* be ignored: `git ls-files -ci --exclude-standard`.

---

**Q7. [L1] You're in "detached HEAD" state. What does that mean, and how do you get out of it without losing work?**

> *What the interviewer is testing:* Mental model of HEAD, branches, and commits.

**Answer:**
Normally `HEAD` points to a branch (e.g. `main`), and the branch points to a commit. "Detached HEAD" means `HEAD` points directly at a commit with no branch in between — usually because you ran `git checkout <commit-hash>` or `git checkout v1.2.0` (a tag).

It's not broken, just risky: any new commits you make in this state aren't on a branch. If you switch away, those commits become unreachable and will eventually be garbage-collected.

To recover:

- **No new commits made yet?** Just `git switch main` (or whichever branch). No data at risk.
- **Made commits you want to keep?** Create a branch from where you are *before* switching:
  ```bash
  git switch -c rescue-branch
  ```
  Now those commits are anchored to a branch and safe.
- **Already switched away and panicking?** `git reflog` shows everywhere `HEAD` has been; find your commit hash and `git branch rescue-branch <hash>`.

---

**Q8. [L2] You ran `git push --force` to your feature branch but realized you wiped out a teammate's commit they had pushed five minutes earlier. How do you recover their commit, and how do you prevent this next time?**

> *What the interviewer is testing:* Reflog recovery, `--force-with-lease`, awareness of force-push hazards.

**Answer:**
The commit isn't lost on the remote yet — it's still in the remote's reflog and likely in your teammate's local repo. Recovery:

1. **Ask the teammate** — their local branch still has the commit. They can push it back (after pulling your changes and rebasing it on top), or share the hash.
2. **Check the remote's reflog** if you have shell access to the server (rare on managed services). On GitHub, the events API and "Network" graph sometimes still show the dangling commit hash for a while; you can fetch it directly: `git fetch origin <sha>`.
3. **Check your own clone** — if you fetched their commit before force-pushing, `git reflog` on the remote-tracking branch (`git reflog show origin/feature-x`) will list it. Cherry-pick it back: `git cherry-pick <sha>` then push.

**Prevention:**

- Use `git push --force-with-lease` instead of `--force`. It refuses to push if the remote has commits you haven't seen — exactly this scenario.
- Better: `git push --force-if-includes` (Git 2.30+) — also verifies your local ref includes the latest fetch.
- Branch protection rules on `main`/`develop` should block force-push entirely. Force-push should only be allowed (and only via `--force-with-lease`) on personal feature branches.

---

**Q9. [L2] You merged a PR into `main` that turned out to be broken in production. The merge has been there for two hours and 15 commits have landed since. How do you back it out safely?**

> *What the interviewer is testing:* `git revert` on merge commits, understanding that you can't just delete published history.

**Answer:**
You can't `reset` `main` — it's shared and 15 other commits have built on top. The right tool is `git revert`, which creates a *new* commit that undoes the changes.

For a merge commit you need `-m` to tell Git which parent to revert to (the "mainline"):

```bash
git revert -m 1 <merge-commit-sha>
```

`-m 1` means "treat parent 1 (the `main` side) as the mainline, and revert everything that came in from the feature side." `-m 2` would do the opposite.

Push the revert. Now `main` is back to a working state and the 15 unrelated commits are preserved.

**Gotcha:** if later you want to re-merge the fixed version of that feature branch, a plain merge will appear to do nothing because Git thinks those changes are already on `main` (they were, then reverted). You either revert the revert (`git revert <revert-sha>`) before re-merging, or rebase the feature branch onto current `main` so the commits get fresh hashes.

---

**Q10. [L2] You have eight messy "WIP" / "fix typo" / "more fixes" commits on your feature branch. Reviewers want to see one clean commit per logical change. How do you reshape the history before opening the PR?**

> *What the interviewer is testing:* Interactive rebase, squash/fixup workflow.

**Answer:**
Use interactive rebase. Pick a base (usually where your branch diverged from `main`):

```bash
git rebase -i origin/main
```

An editor opens with one line per commit:

```
pick a1b2c3 Add user model
pick d4e5f6 WIP
pick 7g8h9i fix typo
pick j0k1l2 Add user controller
pick m3n4o5 more fixes
...
```

Change `pick` to:

- `squash` (or `s`) — combine into the previous commit and let you edit the message.
- `fixup` (or `f`) — same but discard this commit's message (great for "fix typo" commits).
- `reword` (or `r`) — keep the commit but rewrite its message.
- `drop` (or `d`) — remove the commit entirely.
- Reorder lines to reorder commits.

Save and exit. Git replays the commits according to your plan; resolve any conflicts and `git rebase --continue`.

Force-push your *own* feature branch with `--force-with-lease`. Never do this on `main` or any branch others have based work on.

Pro tip: as you go, commit with `git commit --fixup=<sha>` and finish with `git rebase -i --autosquash` — the editor pre-arranges fixups for you.

---

**Q11. [L3] A junior engineer ran `git reset --hard HEAD~5` on their local branch and lost five commits of in-progress work. Nothing was pushed. Walk me through the recovery.**

> *What the interviewer is testing:* Deep understanding of refs, reflog, and Git's garbage collection model.

**Answer:**
A `--hard` reset moves the branch pointer and discards working-tree changes — but the commits themselves are not deleted. They're orphaned (no branch points to them) and will sit in the object database until garbage collection runs (default: ~30 days for unreachable objects, 90 days if reachable from reflog).

Recovery via the reflog, which is a per-clone log of every move `HEAD` (and each branch ref) has made:

```bash
git reflog                       # shows HEAD movements
# or, more targeted:
git reflog show feature-branch   # shows that branch's history
```

You'll see something like:

```
abc1234 HEAD@{0}: reset: moving to HEAD~5
def5678 HEAD@{1}: commit: Last good work
...
```

`def5678` is the tip of the work that was wiped. Restore it:

```bash
git reset --hard def5678
# or, safer if you're unsure:
git branch rescue def5678
git switch rescue
```

Two important caveats: (1) **the reflog is local** — it doesn't help if the commits never existed on this clone (e.g. teammate's machine). (2) Run recovery *before* `git gc` runs. If you suspect a teammate already ran `git gc --prune=now`, the objects may genuinely be gone. As a habit, alias `reset --hard` to require confirmation, and teach the team `git stash` and `git switch -c backup` before risky operations.

---

## 🟡 Collaboration & Remote Workflows

---

**Q12. [L1] In one paragraph: what does `git pull` actually do, and why do some teams prefer `git pull --rebase`?**

> *What the interviewer is testing:* Understanding of fetch + merge composition and history shape preferences.

**Answer:**
`git pull` is two commands stitched together: `git fetch` (download new commits and tags from the remote into `origin/<branch>`) followed by `git merge origin/<branch>` into your current branch. If your local branch has commits the remote doesn't, that merge produces a merge commit ("Merge branch 'main' of …"), which clutters history with bookkeeping commits that don't represent real work.

`git pull --rebase` replaces the merge step with a rebase: your local commits are temporarily set aside, the new remote commits are applied first, and then your commits are replayed on top. Result: a clean linear history with no noise commits. Many teams set `git config --global pull.rebase true` so this becomes the default. The tradeoff is that rebase rewrites your *local* commit hashes — fine for unpushed work, but you should never rebase commits others have already pulled.

---

**Q13. [L1] Explain the four states a file can be in inside a Git repo: untracked, modified, staged, committed. Why does Git have a separate staging area?**

> *What the interviewer is testing:* Mental model of the index/staging area.

**Answer:**
A file moves through these states:

- **Untracked** — exists in your working directory but Git has never been told about it. Shows up under "Untracked files" in `git status`.
- **Modified (tracked)** — Git knows about the file, and the working-tree version differs from what's in the last commit. Shows under "Changes not staged for commit."
- **Staged** — you ran `git add <file>`, copying its current content into the *index* (staging area). Shows under "Changes to be committed." `git commit` will record exactly this snapshot.
- **Committed** — the staged content has been written into a commit object. Working tree, index, and `HEAD` all agree.

The staging area exists so you can build the *next* commit deliberately instead of having every saved file immediately become part of it. You can edit ten files, but `git add` only the three related ones, then `git commit` a focused logical change. `git add -p` takes this further — letting you stage individual hunks within a file. Without the index, you'd have to commit everything at once or stash/branch around it.

---

**Q14. [L2] A critical bug-fix commit exists on `develop` and you need exactly that one commit on a release branch — without bringing along the other 50 commits on `develop`. How do you do it, and what could go wrong?**

> *What the interviewer is testing:* Cherry-pick mechanics and its hazards.

**Answer:**
This is `git cherry-pick`:

```bash
git switch release/1.4
git cherry-pick <commit-sha>
git push
```

Cherry-pick takes the diff that commit introduced and applies it as a *new* commit on the current branch (new SHA, same content). For multiple commits: `git cherry-pick <sha1> <sha2> <sha3>` or a range `git cherry-pick A..B`.

What can go wrong:

- **Conflicts** — if the surrounding code on `release/1.4` is different from `develop`, the patch may not apply cleanly. Resolve, then `git cherry-pick --continue`.
- **Missing dependencies** — the commit may rely on a refactor or a helper that was introduced in an earlier commit on `develop` but isn't on `release/1.4`. The cherry-pick may apply but the code won't compile or behave correctly. You may need to cherry-pick the prerequisite commit too, or write a backport patch.
- **Duplicate-looking commits** — when `develop` is later merged into `release` (or vice versa), Git usually figures out the commits are equivalent (via patch-id), but in some workflows you can end up with two commits doing the same thing under different SHAs. Use `git cherry-pick -x` to record "(cherry picked from commit …)" in the message — invaluable later for traceability.

For ongoing back-port flows (fix on `main`, port to `release/*`), some teams prefer a dedicated `hotfix/*` branch that's merged into both, which avoids cherry-picks entirely.

---

**Q15. [L2] Production is broken. You know it worked at the v2.3 tag but not at HEAD, with about 200 commits between them. How do you find the exact commit that introduced the bug efficiently?**

> *What the interviewer is testing:* `git bisect` workflow, ability to automate root-cause analysis.

**Answer:**
Use `git bisect` — a binary search over commits.

```bash
git bisect start
git bisect bad HEAD       # tell Git the current commit is broken
git bisect good v2.3      # tell Git this old tag was fine
```

Git checks out a commit roughly halfway between them. You build/test it and report:

```bash
git bisect good   # if the bug isn't there
git bisect bad    # if the bug is there
```

Each step halves the search space. With 200 commits, you'll converge in ~8 steps (`log2(200)`). When done, Git prints "X is the first bad commit." Then:

```bash
git bisect reset   # restore your branch
```

**Automate it** if you have a reproducer script that exits 0 (good) or non-zero (bad):

```bash
git bisect start HEAD v2.3
git bisect run ./scripts/reproduce-bug.sh
```

Walk away — Git does the rest.

Tips: skip commits that don't build with `git bisect skip`; use `--first-parent` if your history has many merge commits and you only care about which *merge* introduced the bug. Bisect works because every commit is an immutable snapshot, so each one is independently testable.

---

**Q16. [L2] A developer accidentally committed an AWS access key and pushed it to the public repo. The team noticed 30 minutes later. What's the correct response, in priority order?**

> *What the interviewer is testing:* Incident response thinking; understanding that Git history rewriting alone is not enough.

**Answer:**
Treat the credential as compromised the moment it touches a public surface. Order matters:

1. **Rotate the credential first, before anything else.** In AWS IAM, deactivate the leaked key and issue a new one. Anything you do to Git history is secondary — the key was public for 30 minutes, scrapers are constant, and assume it was harvested.
2. **Audit usage.** Check CloudTrail for any calls authenticated with that key — region, source IP, services touched. If anything looks suspicious, escalate to security.
3. **Remove the secret from history.** A plain `git revert` is **not enough** — the file is still in old commits in `.git/objects` and visible on GitHub forever. Use `git filter-repo` (the modern replacement for `filter-branch`):
   ```bash
   git filter-repo --path secrets.env --invert-paths
   ```
   or `--replace-text` to redact a string everywhere. Then force-push (this rewrites history; coordinate with the team).
4. **Invalidate forks and caches.** GitHub caches the SHA — open a support ticket asking them to purge the leaked commit, and tell anyone with a fork to re-clone.
5. **Add prevention.** Pre-commit hook with `gitleaks` or `detect-secrets`, plus push protection / secret scanning enabled at the org level so this is blocked next time.

The order is non-negotiable: rotate → audit → scrub → prevent. Reversing 1 and 3 is a common mistake — you can't un-leak a key, but you can stop it from being valid.

---

## 🔴 Advanced

---

**Q17. [L3] Your engineering org has grown to 50 developers across three time zones, all working on a single backend service. You currently use long-lived `develop`/`feature/*` branches with weekly merges to `main`. Releases are painful and conflict-heavy. How would you change the branching strategy, and what tradeoffs are you accepting?**

> *What the interviewer is testing:* Strategic thinking on branching models for scale.

**Answer:**
Long-lived feature branches don't scale — the longer a branch lives, the more it diverges, and merges become expensive. The three viable options:

- **Git Flow** (`main`, `develop`, `release/*`, `hotfix/*`, `feature/*`) — heavy ceremony, originally designed for shrink-wrapped software with explicit version numbers. For a fast-moving SaaS backend, this is overkill and a step in the wrong direction.
- **GitHub Flow** — single `main`, short-lived feature branches, PR review, deploy from `main`. Light, simple, well understood.
- **Trunk-based development** — everyone commits to (or merges tiny PRs into) `main` at least daily. No long-lived branches. Incomplete features hide behind feature flags. Continuous deployment from `main`.

For a 50-engineer SaaS team with conflict pain, I'd push toward **trunk-based**:

- Mandate short-lived branches (≤ 24-48 hours from branch to merge).
- Require PR review + green CI to merge — branch protection enforced.
- Adopt feature flags so half-built features can land safely behind a toggle.
- Continuously deploy `main` to staging; promote to prod on a cadence (or per-merge once confident).

**Tradeoffs you're accepting:**

- Investment in a feature-flag platform (LaunchDarkly, Unleash, or homegrown) and the discipline to clean up stale flags.
- Stronger CI investment — `main` must always be releasable, which means fast and reliable test suites and probably required status checks, code coverage gates, etc.
- Cultural shift — engineers must break work into smaller increments, which some find harder than long branches.

What you gain: dramatically smaller merge conflicts (because everyone is integrating against the same recent `main`), faster lead time, simpler mental model, and the ability to ship hotfixes without coordinating across release branches.

---

**Q18. [L3] Your monorepo has grown to 25 GB and 10 years of history. New hires take 45 minutes to clone, IDE indexing is slow, and most engineers only need ~5% of the tree. What Git-side techniques would you use, and where do they fall short?**

> *What the interviewer is testing:* Understanding of partial clone, sparse-checkout, LFS, and their limits.

**Answer:**
Three orthogonal techniques solve three different problems:

1. **Partial clone** — defers downloading file blobs until they're actually accessed:
   ```bash
   git clone --filter=blob:none <url>
   ```
   Fetches all commits and trees but no file contents. When you `checkout` or `log -p` a path, Git lazily fetches just those blobs. Cuts initial clone from 25 GB to a few hundred MB. Requires a server that supports it (GitHub, GitLab, Azure DevOps, Gitea all do).

2. **Sparse-checkout** — limits which paths exist in your working tree:
   ```bash
   git sparse-checkout init --cone
   git sparse-checkout set services/payments libs/common
   ```
   You still have the full history, but only the directories you listed materialize on disk. IDE indexing now sees 5% of the tree. Combine with partial clone for maximum effect (`git clone --filter=blob:none --sparse <url>`).

3. **Git LFS** for large binaries — design files, ML model artifacts, video. LFS stores a small pointer file in Git and the actual blob on a separate object store, fetched on demand. Adopt this *going forward*; converting historical large blobs requires `git lfs migrate import` plus a force-push, which rewrites history.

**Where they fall short:**

- Partial clone makes operations like `git log -p`, `git blame`, and bisect across many files trigger on-demand fetches that can be slow or fail offline. Engineers on flaky networks suffer.
- Sparse-checkout doesn't help operations that *traverse* the repo — `git grep` over a sparse checkout misses files you didn't materialize, which can be surprising. Cone mode mitigates but doesn't eliminate this.
- LFS adds operational dependency on the LFS server, increases hosting cost, and storage isn't free or fast for huge blobs. It's not a silver bullet — for truly large binary pipelines (e.g. game asset workflows), Perforce or a content-addressed object store is sometimes a better fit than Git.
- None of these address the *root* problem on Git itself: extremely deep histories on a few hot files (think: lockfiles touched by everyone) can still slow `blame` and `log`. At true Google/Microsoft scale you eventually outgrow vanilla Git and look at VFS for Git, Scalar, or Piper-style virtual filesystems.

The pragmatic recipe for most teams: sparse + partial clone via a `git clone` wrapper script for new hires, LFS adopted for any binary > a few MB, and a "no committing build artifacts" lint in CI.

---

**Q19. [L3] Your company is enforcing supply-chain integrity. The CISO wants every commit on `main` to have a verifiable author and to be tamper-evident. How do you implement this, and what attack does it actually prevent?**

> *What the interviewer is testing:* Understanding of commit signing, identity vs authorship, and threat modeling.

**Answer:**
By default, Git's `Author` and `Committer` headers are unauthenticated free-text — anyone can run `git config user.email "ceo@company.com"` and produce commits that look like the CEO. Signing fixes that with cryptographic proof.

**Implementation:**

1. **Signing keys per developer.** Either GPG (traditional) or **SSH signing** (Git ≥ 2.34, much simpler — reuse the SSH key engineers already have):
   ```bash
   git config --global gpg.format ssh
   git config --global user.signingkey ~/.ssh/id_ed25519.pub
   git config --global commit.gpgsign true
   git config --global tag.gpgsign true
   ```
2. **Allowed-signers file** mapping email → public key, distributed via your IDP / SSO so trust is centralized, not per-laptop.
3. **Upload the public key to GitHub/GitLab** under "Signing key." Now the platform shows a "Verified" badge and exposes verification status via the API.
4. **Branch protection on `main`** — require signed commits, require linear history, require status checks. Block merges where any commit is unsigned.
5. **CI verification** — a pre-merge job that runs `git verify-commit` against each commit in the PR, failing if any commit isn't signed by a key in the allowed-signers list. Don't rely solely on the platform UI badge.
6. **CODEOWNERS** to require domain-expert review on sensitive paths (e.g. `/infra/`, `/auth/`, CI workflows themselves).

**What this actually prevents:**

- **Author spoofing** — an attacker who compromises one developer's laptop can no longer forge commits as a different developer (different key).
- **Tampering with merged history** — if someone with repo-admin access tries to silently rewrite a past commit, the signature on the original commit chain breaks and CI rejects it.
- **Compromised CI tokens** pushing arbitrary code as humans — unsigned commits are blocked, so a leaked PAT can't ship merge-able changes without a key.

**What it does *not* prevent:**

- A compromised developer laptop with their key on it — the attacker's commits will sign correctly. Mitigations: hardware-backed keys (YubiKey for GPG, or SSH keys in `ssh-agent` with confirmation), short-lived keys via SSO-issued certificates, and behavioral monitoring.
- Malicious code that's *legitimately* authored and signed (insider threat, social-engineered review). That's what CODEOWNERS, mandatory review, and SCA tooling are for.

Signing answers "who wrote this commit?" with cryptography. It does not answer "is this code safe?" — that's a separate problem.

---

**Q20. [L3] Design a branch-protection and merge-policy setup for a regulated environment (PCI / SOC 2) with 30 services in a monorepo. Walk me through the controls you'd put on `main` and how they interact with developer ergonomics.**

> *What the interviewer is testing:* Holistic policy design balancing compliance, security, and velocity.

**Answer:**
Compliance auditors essentially want to see: every change to `main` is reviewed by someone other than the author, every change is traced to a ticket, every change runs the same controls, and the history is tamper-evident.

**Controls on `main`:**

1. **No direct pushes** — pushes only via PR. `Restrict who can push to matching branches` set to no one (not even admins, with `Do not allow bypassing` enabled).
2. **Linear history required** — no merge commits. Forces squash- or rebase-merge, making each `main` commit a single reviewable, revertable unit.
3. **Required reviews** — minimum 1 for low-risk paths, 2 for sensitive paths via CODEOWNERS. `Dismiss stale reviews` on new commits so a reviewer's approval doesn't carry over after the author force-pushes.
4. **CODEOWNERS** mapping critical paths (`/infra/`, `/auth/`, `/billing/`, `/.github/workflows/`) to specific owners. Workflows-on-workflows is a common attack vector — protect `.github/` aggressively.
5. **Required status checks** — CI must pass: unit tests, integration tests, SAST (e.g. Semgrep), dependency scan (Dependabot/Snyk/Trivy), secret scan (gitleaks), license scan, and the duplicate-Q&A or contract-tests if relevant. Pin the exact check names so they can't be renamed away.
6. **Signed commits required** — see prior answer.
7. **Conversation resolution required** — every PR comment must be resolved before merge.
8. **No bypass for admins** — auditors will ask. The tradeoff: a real emergency (rare) requires temporarily lifting the rule, which is logged in the audit trail and is itself a finding to triage.

**Process controls layered on top:**

- **Ticket linking** — PR title regex enforced to include `JIRA-1234` (or similar). The `commit-msg` hook on the server rejects PRs without it. Auditors get traceability from ticket → PR → commit → deployment.
- **PR templates** — checkboxes for "tests added," "security impact considered," "rollback plan." Required for sensitive areas via CODEOWNERS-driven PR templates.
- **Production deploy gating** — `main` is continuously deployed to staging, but production promotion goes through a separate change-management workflow (ServiceNow, Linear, etc.) with the deploying engineer different from the commit author. Many auditors require this segregation of duties.
- **Audit log retention** — Git host audit logs (GitHub Enterprise, GitLab Premium) retained 1+ year and shipped to your SIEM. The audit log captures every protection-rule change, force-push attempt, and admin override.

**Where this hits ergonomics:**

- Two reviewers required on `/auth/` slows urgent fixes — mitigate with a documented break-glass process and clear on-call ownership.
- Required signed commits adds setup friction for new hires — build it into the laptop bootstrap script and have the onboarding checklist verify a signed test commit before they get repo write access.
- Many required checks → slow merges. Invest in CI parallelism and selective testing (run only the affected service's tests when paths under `services/foo/` change). Without this, developers will route around the system, defeating the policy.
- "No bypass for admins" feels paranoid until you're sitting in a SOC 2 audit; lean into it and design the emergency process upfront so engineers know what to do without disabling controls.

The principle: **make the secure path the easy path.** If the protected workflow is faster than circumventing it (because of good tooling, fast CI, and sane PR sizes), engineers stays on it; the policy then enforces itself culturally rather than only via lockdown.

---

## 🟠 Stashing, Tags & Everyday Workflows

---

**Q21. [L1] You're halfway through coding a feature when an urgent bug report comes in. You need to switch branches immediately, but your changes aren't ready to commit. How do you save your in-progress work?**

> *What the interviewer is testing:* Practical understanding of `git stash` and its usage patterns.

**Answer:**
Use `git stash` to save your uncommitted changes onto a stack without committing them:

```bash
git stash push -m "WIP: user profile feature"
git switch hotfix/critical-bug
# ... fix the bug, commit, push ...
git switch feature/user-profile
git stash pop
```

Key details:
- `git stash` saves both staged and unstaged changes to tracked files. **Untracked files are not stashed by default** — add `-u` (or `--include-untracked`) to include them. Add `-a` to also include ignored files.
- `git stash list` shows all stashes. They're numbered `stash@{0}`, `stash@{1}`, etc.
- `git stash pop` applies the most recent stash and removes it from the stack. `git stash apply` applies it but keeps it on the stack — safer if you want to apply the same stash to multiple branches.
- `git stash drop stash@{2}` removes a specific stash. `git stash clear` removes all.
- You can create a branch directly from a stash: `git stash branch new-branch stash@{0}` — useful if the stash conflicts with current work.

Don't use stash as long-term storage — stashes are easy to forget. If the work will sit for more than a few hours, commit it on a throwaway branch instead.

---

**Q22. [L1] What is the difference between a lightweight tag and an annotated tag? When should you use each?**

> *What the interviewer is testing:* Understanding of Git's tag objects and release workflows.

**Answer:**
Both point to a commit, but they're stored differently:

- **Lightweight tag** — just a pointer (like a branch that doesn't move). Created with `git tag v1.0`. No metadata, no message, no signature.
- **Annotated tag** — a full Git object with its own SHA, author, date, message, and optional GPG/SSH signature. Created with `git tag -a v1.0 -m "Release 1.0"`.

When to use each:
- **Annotated tags** for anything published — releases, versioning, deployment markers. They show up in `git describe`, they carry context about *who* tagged and *why*, and they can be signed for tamper-evidence. Most CI/CD pipelines trigger on annotated tags.
- **Lightweight tags** for personal/temporary bookmarks — "I want to remember this commit." They're fine for local use but shouldn't be pushed as release markers.

`git push` does **not** push tags by default. Use `git push --tags` or `git push origin v1.0`. To delete a remote tag: `git push origin --delete v1.0`.

---

**Q23. [L1] You committed with the wrong message — or forgot to add a file to the last commit. How do you fix it without creating a new commit?**

> *What the interviewer is testing:* `--amend` usage and awareness of its implications.

**Answer:**
Use `git commit --amend`:

- **Fix the message only:**
  ```bash
  git commit --amend -m "Correct commit message"
  ```
- **Add a forgotten file to the last commit:**
  ```bash
  git add forgotten-file.js
  git commit --amend --no-edit   # keeps the original message
  ```

`--amend` replaces the last commit with a new one (new SHA). The old commit becomes orphaned and will be garbage-collected eventually.

**Critical rule:** only amend commits that haven't been pushed. If you've already pushed, amending requires a force-push, which rewrites shared history. For pushed commits, create a follow-up commit instead, or use `--force-with-lease` on your own feature branch if the team agrees.

---

**Q24. [L2] Your team uses Git submodules to include a shared library in three different services. A developer reports that after cloning, the submodule directory is empty. What happened, and how do you manage submodules correctly?**

> *What the interviewer is testing:* Submodule mechanics, common pitfalls, and workflow.

**Answer:**
`git clone` does **not** initialize submodules by default — it clones the parent repo and creates the submodule directory, but leaves it empty. The fix:

```bash
git submodule update --init --recursive
```

Or clone with submodules from the start:

```bash
git clone --recurse-submodules <url>
```

How submodules work:
- The parent repo stores a `.gitmodules` file (URL + path mapping) and a special tree entry recording the *exact* commit SHA the submodule should point to.
- `git submodule update` checks out that pinned commit inside the submodule directory — it puts the submodule in **detached HEAD** state.
- To update the submodule to its latest upstream commit: `cd <submodule-dir> && git pull origin main`, then go back to the parent repo and `git add <submodule-dir> && git commit` to record the new SHA.

Common pitfalls:
- **Forgetting `--recurse-submodules` on clone, pull, and checkout** — configure globally: `git config --global submodule.recurse true`.
- **Committing without updating the submodule pointer** — you update the library but forget to commit the new SHA in the parent repo. Other developers don't get the update.
- **Nested submodules** — always use `--recursive`.

Many teams eventually migrate away from submodules to package managers (npm, pip, Maven) or monorepo approaches because submodules add significant cognitive overhead.

---

**Q25. [L2] You need to work on two branches of the same repo simultaneously — for example, testing a fix on `release/2.0` while actively developing on `feature/new-api`. Switching branches back and forth is painful because of build artifacts and IDE reindexing. What's the solution?**

> *What the interviewer is testing:* Awareness of `git worktree` for parallel development.

**Answer:**
Use `git worktree` to check out multiple branches simultaneously in separate directories, all backed by the same `.git` database:

```bash
# From your main checkout (on feature/new-api):
git worktree add ../release-2.0-worktree release/2.0
```

Now you have two working directories:
- `~/projects/my-repo/` → `feature/new-api`
- `~/projects/release-2.0-worktree/` → `release/2.0`

Each has its own working tree, index, and HEAD, but they share the same object store, refs, and config. No extra disk space for the Git history.

Key rules:
- **A branch can only be checked out in one worktree at a time.** Git enforces this to prevent conflicting index states.
- `git worktree list` shows all worktrees.
- `git worktree remove ../release-2.0-worktree` cleans it up when done.
- Worktrees share refs — a commit made in one worktree is immediately visible in the other (they share `.git`).

This is vastly better than cloning the repo twice (which doubles disk usage and requires separate fetches) and avoids the constant `stash/switch/pop` dance.

---

**Q26. [L2] Your team wants to enforce coding standards and prevent certain mistakes at commit time — for example, blocking commits with `console.log()` or failing if unit tests don't pass. How would you set this up with Git hooks?**

> *What the interviewer is testing:* Understanding of Git hooks, client-side vs server-side, and tooling like Husky.

**Answer:**
Git hooks are scripts that run at specific points in the Git workflow. For pre-commit enforcement:

1. **`pre-commit` hook** — runs before the commit is created. Exit non-zero to block the commit:
   ```bash
   #!/bin/sh
   # .git/hooks/pre-commit
   if git diff --cached --name-only | xargs grep -l 'console.log'; then
     echo "ERROR: Remove console.log() before committing"
     exit 1
   fi
   ```

2. **`commit-msg` hook** — validates the commit message format (e.g., enforce Conventional Commits or JIRA ticket references):
   ```bash
   #!/bin/sh
   if ! grep -qE '^(feat|fix|chore|docs|refactor|test): .+' "$1"; then
     echo "ERROR: Commit message must follow Conventional Commits format"
     exit 1
   fi
   ```

3. **`pre-push` hook** — run tests before allowing a push.

**The distribution problem:** hooks live in `.git/hooks/`, which is not committed to the repo. Solutions:
- **Husky** (Node.js) — stores hooks in `.husky/` in the repo and installs them via `npm prepare`.
- **pre-commit framework** (Python) — `.pre-commit-config.yaml` defines hooks from shared repos.
- **lefthook** (Go) — similar, language-agnostic, fast.

**Client-side hooks can be bypassed** with `git commit --no-verify`. For enforcement you can't skip, use **server-side hooks** (`pre-receive`, `update`) on the Git server, or platform-based checks (GitHub required status checks, GitLab push rules).

---

**Q27. [L2] You need to find out who last changed a specific line in a file, when they changed it, and why. Walk me through how you'd investigate.**

> *What the interviewer is testing:* `git blame`, `git log` for a specific line range, and investigation workflow.

**Answer:**
Start with `git blame`:

```bash
git blame -L 42,42 src/auth/login.js
```

This shows the commit SHA, author, date, and content for line 42. The `-L` flag limits output to specific lines (`-L 40,50` for a range).

To understand *why* the change was made:

```bash
git show <commit-sha>      # full diff and commit message
git log --oneline -1 <sha> # just the message
```

If the blame shows a commit like "Apply formatting" (not the real author), dig deeper:

```bash
git blame -L 42,42 <sha>~1 -- src/auth/login.js
```

This blames the file at the commit *before* the formatting change. Or use `git log -L 42,42:src/auth/login.js` to see the full history of changes to that specific line range — every commit that touched those lines, with diffs.

For files that have been renamed, add `--follow`:

```bash
git log --follow -p -- src/auth/login.js
```

Pro tips:
- `git blame --ignore-rev <sha>` skips a known bulk-formatting commit. Create a `.git-blame-ignore-revs` file listing such commits and configure it globally.
- `git blame -w` ignores whitespace changes.
- `git blame -C` detects code moved or copied from other files.

---

**Q28. [L2] You maintain a shared library used by multiple teams. They want updates without a full monorepo migration. A colleague suggests `git subtree` instead of submodules. What's the difference, and how does subtree work?**

> *What the interviewer is testing:* Subtree vs submodule tradeoffs, practical subtree workflow.

**Answer:**
`git subtree` embeds the contents of another repo directly into a subdirectory of your repo — no special `.gitmodules` file, no detached HEADs, no initialization step.

**Adding a subtree:**

```bash
git subtree add --prefix=libs/shared-lib https://github.com/org/shared-lib.git main --squash
```

This pulls the library's code into `libs/shared-lib/` as a single squashed commit. The files are regular tracked files in your repo.

**Pulling updates:**

```bash
git subtree pull --prefix=libs/shared-lib https://github.com/org/shared-lib.git main --squash
```

**Pushing changes back upstream** (if you modify the library in your repo):

```bash
git subtree push --prefix=libs/shared-lib https://github.com/org/shared-lib.git feature/fix-from-consumer
```

**Subtree vs submodule tradeoffs:**

| Aspect | Submodule | Subtree |
|--------|-----------|---------|
| Storage | Pointer to external commit | Full code embedded |
| Clone behavior | Requires `--recurse-submodules` | Just works |
| Updating | `submodule update` | `subtree pull` |
| History | Separate repo history | Merged into parent history |
| Contributor friction | High (extra commands) | Low (files are just there) |
| Pushing changes back | cd into submodule, push | `subtree push` (can be slow on large repos) |

Use subtree when consumers vastly outnumber contributors to the shared library, or when you want zero friction for developers who don't care about the library's internals.

---

**Q29. [L1] What does `git fetch` do vs `git pull`? When would you use `git fetch` alone?**

> *What the interviewer is testing:* Understanding of the two-step nature of pull and when to inspect before integrating.

**Answer:**
`git fetch` downloads new commits, branches, and tags from the remote into your local remote-tracking branches (e.g., `origin/main`) but **does not touch your working directory or local branches**. Your code stays exactly as it is.

`git pull` = `git fetch` + `git merge` (or `git rebase` if configured).

Use `git fetch` alone when:

1. **You want to see what changed before integrating** — after fetching, run `git log HEAD..origin/main` to see incoming commits, or `git diff HEAD origin/main` to see the actual changes. Then decide whether to merge, rebase, or wait.
2. **You're on a different branch** and just want to update your tracking refs for later.
3. **You want to fetch all branches** without merging any: `git fetch --all`.
4. **In CI/CD scripts** where you need full control over what gets integrated and when.

Think of `fetch` as "check for mail" and `pull` as "check for mail and read it immediately." In collaborative workflows, fetching first avoids surprise merge conflicts mid-coding.

---

**Q30. [L2] Your CI pipeline has a job that needs the last 10 commits for changelog generation but the full repo history (50,000 commits) takes too long to clone. How do you optimize this?**

> *What the interviewer is testing:* Shallow clones, depth limiting, and their tradeoffs.

**Answer:**
Use a **shallow clone** with `--depth`:

```bash
git clone --depth=10 <repo-url>
```

This downloads only the last 10 commits and their associated tree/blob objects. The clone is fast and small.

For CI systems that already have a cached checkout, use shallow fetch:

```bash
git fetch --depth=10 origin main
```

**Variations:**

- `--depth=N` — last N commits on each fetched branch.
- `--shallow-since="2024-01-01"` — everything since a date.
- `--shallow-exclude=<ref>` — everything excluding what's reachable from a ref.

**Tradeoffs:**

- `git log` only shows the shallow history — earlier commits are invisible.
- `git merge-base` may fail if the common ancestor is beyond the shallow boundary, breaking merge/rebase operations.
- `git blame` and `git bisect` stop at the shallow boundary.
- `git push` from a shallow clone can fail if the server can't find common ancestors.

**Deepen later if needed:**

```bash
git fetch --unshallow   # converts to a full clone
git fetch --deepen=50   # adds 50 more commits
```

Most CI platforms (GitHub Actions, GitLab CI) expose a `fetch-depth` option. Set it to the minimum needed for your job. For jobs that only need to build and test the latest commit (no history needed), `--depth=1` is ideal.

---

**Q31. [L2] `git rerere` is enabled in your config. What does it do, and in what workflows does it save the most time?**

> *What the interviewer is testing:* Understanding of rerere (reuse recorded resolution) and its niche but powerful use case.

**Answer:**
`rerere` stands for "**re**use **re**corded **re**solution." When enabled (`git config --global rerere.enabled true`), Git silently records how you resolved each merge conflict. If the *exact same* conflict appears again later, Git applies the same resolution automatically.

**How it works:**

1. You hit a conflict during merge or rebase and manually resolve it.
2. Git records the conflict (before) and resolution (after) in `.git/rr-cache/`.
3. Next time the same conflict appears (same pre-image), Git auto-applies your recorded resolution. You still need to `git add` and commit, but the file is already correctly resolved.

**Where it shines:**

- **Long-lived feature branches** that you repeatedly rebase onto `main`. Each rebase replays all commits, and the same conflicts keep appearing. With rerere, you resolve each conflict once; subsequent rebases are automatic.
- **Topic branch workflows** where you test-merge branches into an integration branch, then discard the merge and re-merge later for the real release. Without rerere, you re-resolve every conflict.
- **Cherry-pick-heavy workflows** (backporting fixes across release branches) where similar conflicts recur.

**Caveats:**

- The cache is local — not shared across clones. Each developer's rerere database is separate.
- If you resolve a conflict incorrectly, rerere will faithfully re-apply the wrong resolution. Use `git rerere forget <file>` to clear a bad recording.
- `git rerere gc` cleans up old recordings (default: unresolved conflicts older than 15 days, resolved ones older than 60 days).

---

**Q32. [L1] What is a `.gitkeep` file, and why do you sometimes see empty files with that name committed to repos?**

> *What the interviewer is testing:* Understanding that Git tracks content, not directories.

**Answer:**
Git does **not** track empty directories. If you create `logs/` with nothing in it and run `git add logs/`, nothing happens — Git has nothing to track because the directory has no files.

`.gitkeep` is a **convention** (not a Git feature) — it's an empty file placed inside an otherwise-empty directory so Git will track the directory. The name `.gitkeep` is arbitrary; you could use `.placeholder` or any filename. `.gitkeep` is just the community convention.

Common use cases:
- Ensuring a `tmp/`, `logs/`, or `uploads/` directory exists when someone clones the repo (the app expects the directory to exist at runtime).
- Skeleton project templates where the directory structure matters.

If the directory should exist but its *contents* should be ignored (e.g., `logs/` should be present but log files shouldn't be committed), combine both:

```
# .gitignore
logs/*
!logs/.gitkeep
```

This ignores everything inside `logs/` except the `.gitkeep` file, preserving the directory.

---

**Q33. [L2] You accidentally committed a 500 MB video file three commits ago. The file was deleted in a later commit, but the repo is still huge. Why, and how do you actually remove it?**

> *What the interviewer is testing:* Understanding of Git's object model and history rewriting for large files.

**Answer:**
Deleting a file in a new commit does not remove it from history. Git stores every version of every file as an immutable blob object. The 500 MB blob is still in the object database, reachable from the old commit. Every clone downloads it.

**Finding the large objects:**

```bash
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sed -n 's/^blob //p' | sort -rnk2 | head -20
```

Or use `git-sizer` for a comprehensive report.

**Removing the file from all history:**

The modern tool is `git filter-repo` (replaces the deprecated `git filter-branch`):

```bash
pip install git-filter-repo
git filter-repo --path big-video.mp4 --invert-paths
```

This rewrites every commit that ever contained `big-video.mp4`, removing the file entirely. All commit SHAs from that point onward change.

**After rewriting:**

1. Force-push all branches: `git push --force --all && git push --force --tags`.
2. **Every teammate must re-clone** (or `git fetch origin && git reset --hard origin/main`). Their old local commits reference the rewritten SHAs and will cause confusion.
3. GitHub/GitLab may still cache the old objects — contact support to trigger garbage collection on the server, or wait for the platform's scheduled GC.

**Prevention:** set up Git LFS for large binaries, and add a pre-receive hook or CI check that rejects files above a size threshold.

---

**Q34. [L3] You need to deliver a Git repo to an air-gapped environment with no network. How do you transfer commits?**

> *What the interviewer is testing:* Knowledge of `git bundle` for offline transfer.

**Answer:**
Use `git bundle` — it packages Git objects into a single file that can be copied via USB, email, or any file-transfer mechanism:

**Creating a bundle (on the source machine):**

```bash
# Bundle everything:
git bundle create repo.bundle --all

# Bundle only new commits since a tag:
git bundle create update.bundle v1.5..main
```

**Using the bundle (on the air-gapped machine):**

```bash
# Clone from a bundle (first time):
git clone repo.bundle my-project
cd my-project
git remote set-url origin <real-url-for-later>

# Pull updates from an incremental bundle:
git bundle verify update.bundle   # check it's valid and has prerequisites
git fetch update.bundle main:refs/remotes/origin/main
git merge origin/main
```

**How it works:**

A bundle is essentially a packfile with a header listing the refs it contains and the prerequisite commits it assumes the receiver already has. `git bundle verify` checks that the receiver has those prerequisites.

**Use cases:**

- Air-gapped/classified environments (government, defense, healthcare).
- Transferring repos between disconnected networks.
- Backups — a bundle is a self-contained snapshot of the entire repo.
- Bootstrapping repos on machines where installing Git hosting is impractical.

The incremental approach (`v1.5..main`) keeps bundles small by only including new commits. For regular transfers, maintain a marker tag at each delivery point.

---

**Q35. [L3] Explain how Git stores data internally. What are blobs, trees, commits, and tags at the object level?**

> *What the interviewer is testing:* Deep understanding of Git's content-addressable object model.

**Answer:**
Git is fundamentally a content-addressable filesystem. Every piece of data is stored as an **object** identified by its SHA-1 (or SHA-256) hash. There are four object types:

1. **Blob** — stores file content (just the raw bytes, no filename or permissions). Two files with identical content share the same blob object, regardless of filename or location. This is how Git deduplicates storage.

2. **Tree** — represents a directory. Contains entries mapping filenames → blob SHAs (for files) or other tree SHAs (for subdirectories), along with file mode (permissions). A tree is a snapshot of a directory at a point in time.

3. **Commit** — points to one tree (the root tree of the project at that moment) and zero or more parent commits. Contains author, committer, timestamp, and message. The first commit has no parent; merge commits have two or more parents. The commit's SHA is a hash of all this — changing anything (message, author, parent, tree) produces a different SHA.

4. **Tag (annotated)** — points to a commit (or any object) and adds tagger identity, date, message, and optional signature.

**How a commit represents a full snapshot:**

```
commit abc123
  └── tree def456 (root directory)
       ├── blob 111aaa  README.md
       ├── tree 222bbb  src/
       │    ├── blob 333ccc  main.py
       │    └── blob 444ddd  utils.py
       └── tree 555eee  tests/
            └── blob 666fff  test_main.py
```

Each commit stores a **complete snapshot**, not a diff. Git computes diffs on the fly by comparing two commits' trees. This is why checkout is fast (just materialize one tree) and why branches are cheap (a branch is a 41-byte file containing a commit SHA).

**Packfiles:** for efficiency, Git periodically packs loose objects into packfiles (`.pack` + `.idx`), using delta compression (storing diffs between similar blobs) to reduce disk usage. `git gc` triggers this. Packing is a storage optimization — the logical model is still immutable, content-addressed objects.

---

## 🟣 Diffing, Logging & Code Archaeology

---

**Q36. [L1] What's the difference between `git diff`, `git diff --staged`, and `git diff HEAD`? When would you use each?**

> *What the interviewer is testing:* Understanding of the three-tree architecture (working tree, index, HEAD).

**Answer:**
Git has three important snapshots: the last commit (`HEAD`), the staging area (index), and the working directory. Each `diff` variant compares two of these:

- **`git diff`** — working directory vs staging area. Shows changes you've made but haven't staged yet.
- **`git diff --staged`** (or `--cached`) — staging area vs `HEAD`. Shows what will go into the next commit if you run `git commit` right now.
- **`git diff HEAD`** — working directory vs `HEAD`. Shows *all* changes since the last commit, whether staged or not.

Other useful variations:

```bash
git diff main..feature     # diff between two branches
git diff HEAD~3..HEAD      # last 3 commits' changes
git diff --stat            # summary (files changed, insertions, deletions)
git diff --name-only       # just filenames
git diff -- path/to/file   # limit to specific file
```

For reviewing what you're about to commit, the workflow is: `git diff` to review unstaged changes → `git add -p` to selectively stage → `git diff --staged` to verify what's staged → `git commit`.

---

**Q37. [L1] How do you view the commit history effectively? What are the most useful `git log` options?**

> *What the interviewer is testing:* Practical familiarity with log filtering and formatting.

**Answer:**
`git log` has many options to make history readable and filterable:

**Formatting:**

```bash
git log --oneline              # compact: SHA + message
git log --graph --oneline      # ASCII branch/merge visualization
git log --pretty=format:"%h %an %ar %s"  # custom format
```

**Filtering:**

```bash
git log -n 10                  # last 10 commits
git log --since="2024-01-01"   # time-based
git log --author="Alice"       # by author
git log --grep="fix login"     # search commit messages
git log -S "functionName"      # pickaxe: commits that added/removed this string
git log -G "regex_pattern"     # regex version of pickaxe
git log -- path/to/file        # commits touching a specific file
git log --merges                # only merge commits
git log --no-merges             # exclude merge commits
git log main..feature           # commits on feature not on main
```

**Most useful combination for daily work:**

```bash
git log --oneline --graph --all --decorate
```

This shows the entire branch topology in a compact view. Many developers alias this:

```bash
git config --global alias.lg "log --oneline --graph --all --decorate"
```

For investigating a bug, `git log -S "broken_function"` (pickaxe search) is invaluable — it finds the commit that introduced or removed a specific string.

---

**Q38. [L2] You want to generate a patch file from your commits, email it to a colleague, and have them apply it to their repo. How does the patch workflow work in Git?**

> *What the interviewer is testing:* Understanding of `git format-patch` and `git am` for email-based or offline collaboration.

**Answer:**
Git has built-in support for email-style patch workflows (this is how the Linux kernel is developed):

**Creating patches:**

```bash
# Patch for the last 3 commits:
git format-patch -3

# Patches for commits on your branch not on main:
git format-patch main..HEAD

# Single patch for all changes (combined):
git format-patch main..HEAD --stdout > all-changes.patch
```

`format-patch` creates one `.patch` file per commit, containing the diff, commit message, author, and date in a format that preserves all metadata.

**Applying patches:**

```bash
# Apply and create commits (preserves author, message, date):
git am *.patch

# If there are conflicts:
git am --3way *.patch   # enables three-way merge for better conflict handling
# resolve conflicts, then:
git am --continue
```

**Simpler alternative for one-off diffs:**

```bash
git diff > changes.patch        # create a raw diff
git apply changes.patch         # apply it (no commit created)
```

`git apply` is simpler but loses commit metadata. Use `format-patch` + `am` when you want full commit fidelity.

**Use cases:**

- Contributing to projects that use mailing list workflows (Linux kernel, Git itself).
- Transferring changes when you can't push/PR (air-gapped, different hosting platforms).
- Code review via email.
- Backing up specific commits as portable files.

---

**Q39. [L2] Two developers merge different image files (PNG) with the same filename. Git reports a binary conflict. How do you resolve it?**

> *What the interviewer is testing:* Handling binary file conflicts, custom merge drivers.

**Answer:**
Git can't merge binary files — there's no line-by-line diff. When both branches modify the same binary file, Git marks it as conflicted and keeps both versions accessible:

```bash
git checkout --ours -- assets/logo.png    # keep your version
git checkout --theirs -- assets/logo.png  # keep their version
```

After choosing, stage and commit:

```bash
git add assets/logo.png
git merge --continue
```

If you need to compare both versions before deciding:

```bash
git show :2:assets/logo.png > logo-ours.png    # stage 2 = ours
git show :3:assets/logo.png > logo-theirs.png  # stage 3 = theirs
```

Open both files, decide which to keep (or combine them in an image editor), replace the conflicted file, and stage it.

**Prevention strategies:**

- **Git LFS** with file locking — `git lfs lock assets/logo.png` prevents concurrent edits to binary files. Others see the file as locked and coordinate.
- **Custom merge drivers** in `.gitattributes` — you can define how specific file types are merged:
  ```
  *.png merge=binary
  *.psd merge=ours
  ```
  `merge=ours` automatically keeps the current branch's version for `.psd` files, avoiding conflicts entirely (with the tradeoff of silently ignoring the other side's changes).

---

**Q40. [L2] Your company has multiple Git remotes — the primary on GitHub, a mirror on GitLab for CI, and a backup on an internal server. How do you manage pushing to all of them?**

> *What the interviewer is testing:* Multi-remote configuration and push strategies.

**Answer:**
Git supports multiple remotes natively. Set them up:

```bash
git remote add github https://github.com/org/repo.git
git remote add gitlab https://gitlab.com/org/repo.git
git remote add backup git@internal-server:org/repo.git
```

**Option 1: Push to each explicitly:**

```bash
git push github main
git push gitlab main
git push backup main
```

**Option 2: Configure a push URL group** — add multiple push URLs to a single remote:

```bash
git remote set-url --add --push origin https://github.com/org/repo.git
git remote set-url --add --push origin https://gitlab.com/org/repo.git
git remote set-url --add --push origin git@internal-server:org/repo.git
```

Now `git push origin main` pushes to all three simultaneously. `git pull` still fetches from the first (fetch) URL.

**Option 3: CI-driven mirroring** — a post-push webhook or CI job on GitHub that mirrors to GitLab and the internal server. This is the most reliable approach for teams because it's centralized and auditable.

**Verify your setup:**

```bash
git remote -v
```

Shows fetch and push URLs for each remote.

**Best practice:** designate one remote as the "source of truth" (usually `origin`). Mirrors should be read-only replicas populated by automation, not by developers pushing manually, to avoid divergence.

---

**Q41. [L2] What is a refspec, and why would you need to understand it?**

> *What the interviewer is testing:* Understanding of the plumbing behind fetch/push and remote branch mapping.

**Answer:**
A refspec defines the mapping between remote refs and local refs. It's the rule Git follows when fetching or pushing to know *which* remote branch maps to *which* local branch.

Format: `+<src>:<dst>`

- The `+` prefix means force-update (like `--force`).
- `<src>` is the source ref pattern.
- `<dst>` is the destination ref pattern.

**Default fetch refspec** (set by `git clone`):

```
[remote "origin"]
    fetch = +refs/heads/*:refs/remotes/origin/*
```

This means: "take all branches on the remote (`refs/heads/*`) and store them locally as `refs/remotes/origin/*`." That's why `origin/main` exists locally after a fetch.

**Practical use cases:**

- **Fetch a specific branch only:**
  ```bash
  git fetch origin +refs/heads/main:refs/remotes/origin/main
  ```

- **Push to a differently-named remote branch:**
  ```bash
  git push origin local-branch:remote-branch
  ```

- **Delete a remote branch** (push "nothing" to it):
  ```bash
  git push origin :old-branch
  # modern syntax: git push origin --delete old-branch
  ```

- **Fetch PR refs from GitHub** (PRs aren't branches by default):
  ```bash
  git fetch origin +refs/pull/*/head:refs/remotes/origin/pr/*
  git checkout origin/pr/42
  ```

Most developers never write refspecs manually, but understanding them helps debug fetch/push issues and configure advanced workflows like PR testing or mirroring specific branches.

---

**Q42. [L1] What does `git log --all --graph --oneline` show, and how do you read the output?**

> *What the interviewer is testing:* Ability to visualize and interpret branch topology.

**Answer:**
This command shows a compact, visual representation of the entire commit history across all branches:

```
* e4f5g6h (HEAD -> feature/auth) Add JWT validation
| * a1b2c3d (origin/main, main) Update README
|/
* 7h8i9j0 Merge pull request #42
|\
| * k1l2m3n Fix login bug
|/
* o4p5q6r Initial commit
```

How to read it:

- **`*`** = a commit
- **`|`** = a branch line continuing vertically
- **`/` and `\`** = branches diverging or converging
- **`|\`** = a merge (two parents coming together)
- **`|/`** = a branch that was merged in
- **Text in `()`** = refs (branches, tags, HEAD) pointing to that commit
- **`HEAD ->`** = which branch you're currently on

This is the single most useful command for understanding "what happened" in a repo — which branches exist, where they diverged, and how they were merged. It replaces the need for a GUI in most cases.

Add `--decorate` (usually default) to see branch/tag labels. Add `--date=short` and `--pretty=format:...` for more detail.

---

**Q43. [L2] What is `.mailmap` and when would you use it?**

> *What the interviewer is testing:* Awareness of author normalization in Git history.

**Answer:**
`.mailmap` is a file that maps different author names and emails to a canonical identity. It fixes inconsistencies in `git log`, `git shortlog`, and `git blame` without rewriting history.

**The problem:** over time, the same person commits with different identities:

```
Alice Smith <alice@company.com>
Alice <alice@personal.com>
A. Smith <asmith@oldcompany.com>
```

**The solution — `.mailmap` in the repo root:**

```
Alice Smith <alice@company.com> <alice@personal.com>
Alice Smith <alice@company.com> A. Smith <asmith@oldcompany.com>
```

Format: `Canonical Name <canonical@email> [Original Name] <original@email>`

After adding this file, `git shortlog -sne` and `git log --format='%aN'` consolidate all entries under the canonical identity. `git blame` also uses it.

**Use cases:**

- Open-source projects where contributors use different emails (work vs personal).
- Post-acquisition consolidation (merging email domains).
- Correcting typos in historical author names.
- `git shortlog -sne` for accurate contribution statistics.

The `.mailmap` file should be committed to the repo. It doesn't change any commit objects — it's a display-time mapping only.

---

## 🔶 Git Configuration & Optimization

---

**Q44. [L1] How do you set up useful Git aliases, and what are some productivity aliases every developer should have?**

> *What the interviewer is testing:* Practical Git workflow optimization.

**Answer:**
Git aliases are shortcuts defined in your Git config:

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
```

Now `git co main` = `git checkout main`, etc.

**Power aliases:**

```bash
# Pretty log with graph
git config --global alias.lg "log --oneline --graph --all --decorate"

# Short status
git config --global alias.ss "status -sb"

# Undo last commit (keep changes staged)
git config --global alias.undo "reset --soft HEAD~1"

# Show what you did today
git config --global alias.today "log --since='midnight' --oneline --author='$(git config user.email)'"

# Interactive rebase shortcut
git config --global alias.ri "rebase -i"

# List branches sorted by last commit date
git config --global alias.recent "branch --sort=-committerdate --format='%(committerdate:relative) %(refname:short)'"

# Amend without editing message
git config --global alias.oops "commit --amend --no-edit"
```

**Shell command aliases** (prefix with `!`):**

```bash
git config --global alias.cleanup '!git branch --merged main | grep -v main | xargs git branch -d'
```

This deletes all local branches already merged into `main`.

Aliases live in `~/.gitconfig` under `[alias]`. They're portable — commit a shared alias config in your team's dotfiles repo for consistency.

---

**Q45. [L2] What does `git gc` do, and when should you run it manually?**

> *What the interviewer is testing:* Understanding of Git's garbage collection and object packing.

**Answer:**
`git gc` (garbage collection) performs housekeeping on the Git object database:

1. **Packs loose objects** into packfiles — individual objects in `.git/objects/` are compressed into efficient `.pack` files with delta compression.
2. **Removes unreachable objects** — commits, blobs, and trees that no ref (branch, tag, reflog) points to. These are typically from amended commits, rebases, or deleted branches.
3. **Packs refs** — consolidates individual ref files in `.git/refs/` into a single `packed-refs` file.
4. **Prunes old reflog entries** — entries older than 90 days (reachable) or 30 days (unreachable) by default.

**When Git runs it automatically:**

Git triggers `gc --auto` after certain operations (e.g., when there are more than 6700 loose objects or more than 50 packfiles). You rarely need to run it manually.

**When to run manually:**

- After large history rewrites (`filter-repo`, mass deletions) to reclaim disk space.
- When the repo feels slow and `.git/` is unusually large.
- Before creating a bundle or archive — ensures maximum compression.

**Aggressive GC:**

```bash
git gc --aggressive --prune=now
```

This spends more CPU time on delta compression for a smaller packfile. Only useful occasionally (e.g., after importing from another VCS). Don't run it routinely — the default compression is good enough and `--aggressive` is slow.

**Caution:** `--prune=now` immediately deletes unreachable objects. If you're recovering lost commits via reflog, run recovery *before* GC.

---

**Q46. [L2] What is `git fsck` and when would you use it?**

> *What the interviewer is testing:* Knowledge of Git's integrity checking and disaster recovery.

**Answer:**
`git fsck` (file system check) verifies the integrity of the Git object database — it walks every object and checks that nothing is corrupted, missing, or orphaned.

```bash
git fsck --full
```

**What it checks:**

- Every object's SHA matches its content (detects corruption).
- Every commit's parent exists.
- Every tree's referenced blobs and sub-trees exist.
- No invalid object types or malformed headers.

**What it reports:**

- **`dangling commit`** — a commit not reachable from any branch or tag. Often from amended commits, rebases, or deleted branches. Harmless; `git gc` cleans them up.
- **`dangling blob`** — file content not referenced by any tree. Usually from staged-but-uncommitted files.
- **`missing object`** — an object referenced but not found. This is actual corruption.
- **`broken link`** — a tree or commit references an object that doesn't exist.

**When to use it:**

- **After a disk failure or unclean shutdown** — verify the repo isn't corrupted.
- **Recovering lost commits** — `git fsck --unreachable` lists orphaned commits you might want to rescue (alternative to `reflog`).
- **Debugging weird Git errors** — "fatal: bad object" errors often point to corruption that `fsck` can diagnose.

**Recovery from corruption:** if `fsck` reports missing objects, try re-fetching from the remote (`git fetch origin`). If the remote has the objects, they'll fill the gaps. For truly lost objects, restore from a backup or re-clone.

---

**Q47. [L3] Your team debates whether to use squash-merge, rebase-merge, or regular merge commits when closing PRs. What are the tradeoffs of each strategy?**

> *What the interviewer is testing:* Understanding merge strategies and their impact on history, bisect, and revert.

**Answer:**
Each strategy produces a different commit history shape with different tradeoffs:

**1. Merge commit (`--no-ff`):**

```
*   Merge PR #42: Add user auth
|\
| * Fix test
| * Add JWT middleware
| * Add login endpoint
|/
* Previous main commit
```

- **Pros:** preserves full branch history; you can see exactly what happened on the feature branch; easy to revert the entire PR with `git revert -m 1 <merge-sha>`.
- **Cons:** history is noisy with "fix typo" and "WIP" commits; `git log` on `main` shows every commit from every branch; `git bisect` may land on broken intermediate commits.

**2. Squash merge:**

```
* Add user auth (#42)
* Previous main commit
```

- **Pros:** one clean commit per PR on `main`; `git log` is easy to read; each commit is a complete, reviewable unit; `git bisect` is highly effective.
- **Cons:** original branch history is lost (individual commits disappear); the squashed commit has a single author even if the PR had multiple contributors; if the PR is large, the single commit is hard to review or partially revert.

**3. Rebase merge (fast-forward):**

```
* Fix test
* Add JWT middleware
* Add login endpoint
* Previous main commit
```

- **Pros:** linear history, no merge commits; each commit is preserved individually; clean `git log`.
- **Cons:** no visual grouping of "this PR's commits"; harder to revert an entire PR (must revert multiple commits); requires the feature branch to be rebased onto latest `main` before merging; commit SHAs change.

**Recommendation by team maturity:**

- **Small team, disciplined commits:** rebase-merge — clean, linear, each commit meaningful.
- **Medium team, mixed commit quality:** squash-merge — hides messy history, one commit = one PR.
- **Large team, compliance needs:** merge commits — full traceability, easy PR-level reverts, audit trail preserved.

Many teams use squash for feature PRs and merge commits for release/hotfix merges.

---

**Q48. [L3] You're setting up a Git server for an organization. What are the differences between the four transfer protocols Git supports (Local, HTTP, SSH, Git), and which would you choose?**

> *What the interviewer is testing:* Understanding of Git transport protocols and security considerations.

**Answer:**
Git supports four protocols for communication between client and server:

**1. Local protocol (`file://` or just a path):**

```bash
git clone /srv/git/repo.git
```

- Uses the filesystem directly. No network, no authentication beyond filesystem permissions.
- Fast, simple, but only works on the same machine or NFS mounts.
- Use case: shared server with SSH access where repos live on a mounted filesystem.

**2. HTTP/HTTPS (Smart HTTP):**

```bash
git clone https://github.com/org/repo.git
```

- Runs over standard HTTP(S) — passes through firewalls, proxies, and load balancers.
- Authentication via username/password, tokens, or SSO.
- Smart HTTP (default since Git 1.6.6) negotiates only needed objects — efficient as SSH.
- **Recommended for most organizations** — works everywhere, supports all auth methods, TLS for encryption.

**3. SSH:**

```bash
git clone git@github.com:org/repo.git
```

- Encrypted, authenticated via SSH keys. No anonymous access possible.
- Well-understood security model; keys can be centrally managed (LDAP, SSO-issued certificates).
- No built-in authorization granularity — you either have shell access or you don't. Tools like Gitolite or platform features (GitHub, GitLab) add per-repo/per-branch authorization on top.
- **Best for: internal teams** where everyone has SSH keys, and you want simplicity without HTTP infrastructure.

**4. Git protocol (`git://`):**

```bash
git clone git://github.com/org/repo.git
```

- Unauthenticated, unencrypted, read-only by convention. Runs on port 9418.
- Fastest protocol (no encryption overhead), but no security.
- **Almost never used today.** Was useful for public read-only mirrors; HTTPS has replaced it.

**For a new organization:** Smart HTTPS with token-based authentication (PATs or OAuth via SSO). It works through corporate proxies, supports granular permissions via the hosting platform, and uses TLS. SSH as a secondary option for developers who prefer it. Never use the raw Git protocol.

---

**Q49. [L3] Your monorepo CI is slow because every PR triggers tests for all 30 services. How would you use Git to determine which services are affected by a PR and only run their tests?**

> *What the interviewer is testing:* Using Git diff for selective CI, path-based triggering.

**Answer:**
Use `git diff` to identify changed paths and map them to affected services:

**Step 1: Get changed files in the PR:**

```bash
# Compare PR branch against the merge target:
CHANGED_FILES=$(git diff --name-only origin/main...HEAD)
```

The three-dot syntax (`...`) finds the merge-base and shows only what the PR changed — not what `main` changed since the branch was created.

**Step 2: Map changed paths to services:**

```bash
# services/payments/*, libs/payment-sdk/* → payments
# services/auth/*, libs/auth-common/* → auth
# libs/common/* → ALL services (shared dependency)
```

This mapping can be a simple bash script, a JSON config file, or defined in CI config:

```yaml
# .github/workflows/ci.yml (GitHub Actions with path filters)
on:
  pull_request:
    paths:
      - 'services/payments/**'
      - 'libs/payment-sdk/**'
jobs:
  test-payments:
    # only runs when the above paths change
```

**Step 3: Handle shared dependencies:**

The tricky part. If `libs/common/` changes, every service that depends on it must be tested. Maintain a dependency graph (or use build tools like Bazel, Nx, or Turborepo that understand it natively).

```bash
# Determine affected services based on changed paths
AFFECTED=$(python scripts/affected-services.py $CHANGED_FILES)
for service in $AFFECTED; do
  run_tests "$service"
done
```

**Advanced approaches:**

- **Bazel/Pants/Buck** — build systems that understand the dependency graph and test only affected targets: `bazel test //... --test_tag_filters=-manual` with remote caching.
- **Nx/Turborepo** (JS/TS monorepos) — `nx affected --target=test` automatically determines what to test based on the Git diff.
- **git diff with `--diff-filter`** — distinguish added, modified, deleted, and renamed files for more precise mapping.

The key insight: `git diff --name-only` gives you the raw data; the intelligence is in the mapping from file paths to services/test suites.

---

**Q50. [L3] A developer reports that `git push` is extremely slow (takes 5+ minutes) even for small commits. The repo itself is only 500 MB. How do you diagnose and fix this?**

> *What the interviewer is testing:* Debugging Git performance, understanding pack negotiation, and server-side issues.

**Answer:**
Slow pushes with small changesets typically aren't about bandwidth — the bottleneck is usually pack negotiation or server-side processing.

**Diagnosis:**

1. **Enable Git tracing:**
   ```bash
   GIT_TRACE=1 GIT_TRANSFER_TRACE=1 GIT_CURL_VERBOSE=1 git push origin main
   ```
   This shows every step: DNS resolution, TLS handshake, ref advertisement, pack negotiation, and upload.

2. **Check where time is spent:**
   - **"Counting objects" / "Compressing objects" takes minutes?** — the local repo has too many loose objects or a bloated object database. Run `git gc` and `git repack -a -d --depth=250 --window=250`.
   - **Negotiation phase is slow?** — Git exchanges "have/want" lists with the server. With many branches and tags, this can be large. Prune stale remote refs: `git remote prune origin`. Delete merged branches.
   - **Server-side hooks are slow?** — pre-receive or update hooks (code scanning, large file checks, CI triggers) run synchronously. Check with the platform admin.
   - **Network latency?** — `curl -o /dev/null -w "time_connect: %{time_connect}\ntime_total: %{time_total}\n" <git-server-url>` to check.

3. **Check for large files in recent commits:**
   ```bash
   git rev-list --objects HEAD~5..HEAD | \
     git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' | \
     sort -rnk2 | head -10
   ```
   A single large blob forces a big packfile upload.

**Fixes:**

- **Run `git gc --aggressive`** if the local object store is fragmented.
- **Enable `push.negotiate` (Git 2.36+):** `git config push.negotiate true` — uses a more efficient negotiation algorithm.
- **Use SSH instead of HTTPS** if TLS overhead is contributing.
- **Push only the branch you need:** `git push origin main` instead of `git push --all`.
- **Consider `--thin` (default)** — verify it's not disabled. Thin packs send deltas against objects the server already has, reducing transfer size.
- **Check for server-side quotas or throttling** — some Git hosting providers rate-limit pushes or run expensive server-side operations.

If the root cause is many stale refs, regularly run `git fetch --prune` and delete merged feature branches both locally and remotely.

---
