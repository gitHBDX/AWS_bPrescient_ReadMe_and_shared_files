-----------------------------------------------------------------
# ALL USERS (Peter & Ishan)

### 0) Freeze: nobody pushes during cleanup

### 1) Backup
```bash
git fetch --all --prune
git branch backup/pre-cleanup-$(date +%Y%m%d-%H%M%S)
git bundle create ../pre-cleanup.bundle --all
```
-----------------------------------------------------------------
# ONLY ONE of you

### 2) Ensure local branches exist (from your screenshot)
```bash
git checkout -B main origin/main
git checkout -B add-eda origin/add-eda
git checkout -B eda origin/eda
git checkout main
```

### 3) Install tool (if needed)
```bash
python -m pip install --user git-filter-repo
```

### 4) Remove large notebooks from branch history
```bash
~/.local/bin/git-filter-repo --force \
  --path init_eda.ipynb \
  --path smallRNA_pipeline.ipynb \
  --invert-paths \
  --refs refs/heads/main refs/heads/add-eda refs/heads/eda
```

### 5) Verify (should show nothing)
```bash
git log --all -- init_eda.ipynb smallRNA_pipeline.ipynb
git rev-list --objects --all | rg 'init_eda.ipynb|smallRNA_pipeline.ipynb'
```

### 6) Push rewritten branches to CodeCommit
```bash
git push origin main --force-with-lease
git push origin add-eda --force-with-lease
git push origin eda --force-with-lease
```

-----------------------------------------------------------------
# ALL USERS (Peter & Ishan)

### Then all other users must resync (no normal pull)
```bash
git fetch --all --prune
git checkout main && git reset --hard origin/main
```
git checkout add-eda && git reset --hard origin/add-eda
git checkout eda && git reset --hard origin/eda
