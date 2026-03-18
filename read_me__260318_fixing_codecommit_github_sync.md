# Description:

The core problem is a repo-specific mirror failure: commits are reaching AWS CodeCommit, but forwarding to GitHub breaks for one affected branch/repo (likely due to problematic large notebook commits).
Because that branch keeps failing in the mirror Lambda flow, synchronization for that repo becomes unreliable/stalled, while other repos can still sync normally.

# CodeCommit -> GitHub Sync Recovery Runbook

This runbook is for cases where:
- commits arrive in AWS CodeCommit,
- but are not forwarded to GitHub for one repo/branch,
- and collaborators (for example Peter and Ishan) have diverged local versions.

## Goal

Recover mirroring for a single affected repo (example: `bPrescient_HBDx_1862`) without losing anyone's work.

## 0) Freeze Changes

Both users stop pushing to the affected repo until reconciliation is complete.

## 1) Backup Both Local States (Peter + Ishan)

Run in each user's SageMaker terminal inside their clone.

```bash
git status
git branch --show-current
git fetch --all --prune

# Create a backup branch with timestamp
ts=$(date +%Y%m%d-%H%M%S)
git checkout -b backup/${USER}-${ts}

# Optional but strongly recommended: full backup bundle
git bundle create ../${USER}-${ts}.bundle --all
```

If there are uncommitted changes:

```bash
git add -A
git commit -m "WIP backup before sync recovery"
```

## 2) Pick One Integrator SageMaker Space

Choose exactly one SageMaker space (Peter or Ishan) as "source of truth" for reconciliation.

## 3) Bring Both Histories to Integrator

### Option A (easiest): push backup branch to CodeCommit

On Peter:

```bash
git push origin backup/peter-YYYYMMDD-HHMMSS
```

On Ishan:

```bash
git push origin backup/ishan-YYYYMMDD-HHMMSS
```

If push permissions are limited, bundles can still be used inside SageMaker, but branch push is usually simpler/faster.

## 4) Create Clean Recovery Branch (Integrator Space)

On integrator SageMaker space:

```bash
git fetch --all --prune
git checkout -b recovery/bPrescient_HBDx_1862 origin/main
```

If mirroring broke on `add-eda`, also inspect that branch:

```bash
git log --oneline --decorate origin/add-eda -n 30
```

## 5) Merge/Cherry-pick Work From Both Users

On integrator SageMaker space, merge backup branches one by one and resolve conflicts deliberately.

```bash
git merge --no-ff origin/backup/peter-YYYYMMDD-HHMMSS
git merge --no-ff origin/backup/ishan-YYYYMMDD-HHMMSS
```

If merge is too noisy, cherry-pick specific commits instead:

```bash
git cherry-pick <commit_sha_1> <commit_sha_2>
```

## 6) Fix Large Notebook Risk (Important)

Large `.ipynb` files or huge outputs can break mirror push behavior.

### Check big objects/files

```bash
find . -name "*.ipynb" -size +20M -print
git count-objects -vH
```

### Strip notebook outputs before final push

```bash
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace init_eda.ipynb smallRNA_pipeline.ipynb
git add init_eda.ipynb smallRNA_pipeline.ipynb
git commit -m "Strip notebook outputs to reduce repository object size"
```

If notebooks must stay large, use Git LFS for notebook artifacts/binaries.

## 7) Push Recovery Branch and Validate Mirror

```bash
git push origin recovery/bPrescient_HBDx_1862
```

Then verify:
- CodeCommit received the branch/commit.
- GitHub mirror receives the same commit within a few minutes.

## 8) Update Target Branch Safely

After verification, fast-forward merge or controlled force update.

Preferred:

```bash
git checkout main
git merge --ff-only recovery/bPrescient_HBDx_1862
git push origin main
```

Only if history rewrite is required:

```bash
git push origin main --force-with-lease
```

## 9) Resync Peter and Ishan SageMaker Clones

On both user machines:

```bash
git fetch --all --prune
git checkout main
git reset --hard origin/main
```

If they work on `add-eda`:

```bash
git checkout add-eda
git reset --hard origin/add-eda
```

## 10) Post-Recovery Guardrails

- Avoid committing notebook outputs unless needed.
- Add notebook cleanup to pre-commit hooks or CI.
- Use feature branches + pull requests before touching mirror-critical branches.
- If one branch repeatedly fails, test with a tiny commit first to confirm mirror health.

---

## SageMaker Notes

It is fully doable in SageMaker if you keep these guardrails:
- Use `tmux` so terminal disconnects do not interrupt work.
- Keep one integrator space to avoid concurrent branch rewrites.
- Always create backup branches first.
- Use `--force-with-lease` instead of `--force`.
