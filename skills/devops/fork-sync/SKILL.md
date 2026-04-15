---
name: fork-sync
description: Sync fork repo with upstream, detect install script changes, and merge without breaking China mainland mirror acceleration patches.
version: 1.0.0
metadata:
  hermes:
    tags: [git, fork, upstream, sync, merge, mirror]
---

# Fork Upstream Sync

Syncs fork's `dev` branch with upstream `main` branch. Detect whether upstream changes affect install script (`scripts/install.sh`). If they do, merge carefully to preserve mirror acceleration patches. If not, fast-forward merge directly.

## Workflow Rules

**Always follow these rules:**

1. **`main` branch stays clean** — it must always match upstream `main` exactly. Never make any changes directly on `main`.
2. **All changes go on `dev`** — all mirror acceleration patches and modifications are committed to the `dev` branch only.
3. **Squash before syncing** — when syncing, squash multiple `dev` commits into one before force-pushing, so `dev` is always exactly one commit ahead of `main`.
4. **One commit ahead** — after each sync, verify `dev` has exactly one more commit than `main`.

This workflow ensures clean history and easy rebase when upstream updates.

## Prerequisites

- The repo must have both `origin` (fork) and `upstream` (source) remotes configured.
- The `dev` branch contains mirror acceleration patches that must be preserved.

### Verify Remote Setup

```bash
git remote -v
```

Expected output should include:
- `origin` → `https://github.com/Drswith/hermes-agent.git` (fork)
- `upstream` → `https://github.com/NousResearch/hermes-agent.git` (source)

If `upstream` is missing:

```bash
git remote add upstream https://ghfast.top/https://github.com/NousResearch/hermes-agent.git
```

## Sync Procedure

Follow these steps in order:

### Step 1: Fetch Upstream

```bash
git fetch upstream
```

### Step 2: Compare Upstream Changes

Check if there are new commits on upstream `main`:

```bash
LOCAL_MERGE_BASE=$(git merge-base main upstream/main)
UPSTREAM_LOG=$(git log $LOCAL_MERGE_BASE..upstream/main --oneline)

if [ -z "$UPSTREAM_LOG" ]; then
    echo "Already up to date. No action needed."
    exit 0
fi

echo "New upstream commits:"
echo "$UPSTREAM_LOG"
```

### Step 3: Check If Install Script Is Affected

```bash
CHANGED_FILES=$(git diff $LOCAL_MERGE_BASE..upstream/main --name-only)

if echo "$CHANGED_FILES" | grep -q "scripts/install.sh"; then
    echo "install script changed — careful merge required"
    NEEDS_CAREFUL_MERGE=true
else
    echo "install script NOT changed — safe to fast-forward"
    NEEDS_CAREFUL_MERGE=false
fi
```

### Step 4A: No Install Script Changes — Simple Sync

When `NEEDS_CAREFUL_MERGE=false`:

```bash
# Sync main with upstream
git checkout main
git merge upstream/main --ff-only
git push origin main

# Squash dev commits to one, then rebase
git checkout dev
git reset --soft main
git commit -m "feat: <squashed commit message>"
git push origin dev --force-with-lease
```

Done. No conflicts expected.

### Step 4B: Install Script Changed — Careful Merge

When `NEEDS_CAREFUL_MERGE=true`, the mirror acceleration patches in `dev` may conflict with upstream changes to `scripts/install.sh`. Handle as follows:

```bash
# Sync main with upstream
git checkout main
git merge upstream/main --ff-only
git push origin main
```

Now merge into `dev`:

```bash
git checkout dev
git rebase main
```

If rebase succeeds without conflicts:

```bash
# Squash to one commit
git reset --soft main
git commit -m "feat: <squashed commit message>"
git push origin dev --force-with-lease
echo "Rebase and squash succeeded. Verify mirror patches are intact:"
grep -n "UV_INDEX_URL\|NPM_CONFIG_REGISTRY\|PLAYWRIGHT_DOWNLOAD_HOST\|ghfast.top" scripts/install.sh
```

If rebase hits a conflict in `scripts/install.sh`:

1. Read the conflicted file:

```bash
cat scripts/install.sh
```

2. Resolve the conflict by keeping BOTH the upstream changes AND the mirror acceleration additions. The mirror patches are:
   - The three `export` lines at the top (`UV_INDEX_URL`, `NPM_CONFIG_REGISTRY`, `PLAYWRIGHT_DOWNLOAD_HOST`)
   - The `REPO_URL_HTTPS` line using `ghfast.top`
   - Any README additions for the China mainland install command

3. After resolving:

```bash
git add scripts/install.sh
git rebase --continue
```

4. After rebase completes, squash to one commit:

```bash
git reset --soft main
git commit -m "feat: <squashed commit message>"
git push origin dev --force-with-lease
```

5. Verify patches survived:

```bash
grep -n "UV_INDEX_URL\|NPM_CONFIG_REGISTRY\|PLAYWRIGHT_DOWNLOAD_HOST\|ghfast.top" scripts/install.sh
```

Expected output should include:
```
17:export UV_INDEX_URL="https://pypi.tuna.tsinghua.edu.cn/simple"
18:export NPM_CONFIG_REGISTRY="https://registry.npmmirror.com"
19:export PLAYWRIGHT_DOWNLOAD_HOST="https://npmmirror.com/mirrors/playwright"
20:export PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1
34:REPO_URL_HTTPS="https://ghfast.top/https://github.com/NousResearch/hermes-agent.git"
```

### Step 5: Final Verification

After any sync, always verify:

```bash
echo "=== Mirror patches in install.sh ==="
grep -n "UV_INDEX_URL\|NPM_CONFIG_REGISTRY\|PLAYWRIGHT_DOWNLOAD_HOST\|ghfast.top" scripts/install.sh

echo ""
echo "=== README accelerated install command ==="
grep -n "ghfast.top" README.md

echo ""
echo "=== Branch status ==="
git log --oneline -5 main
echo "---"
git log --oneline -5 dev
```

## Mirror Acceleration Patch Checklist

These are the lines that must be preserved after any merge:

| File | Line(s) | Content |
|------|---------|---------|
| `scripts/install.sh` | ~17-19 | `export UV_INDEX_URL=...`, `export NPM_CONFIG_REGISTRY=...`, `export PLAYWRIGHT_DOWNLOAD_HOST=...` |
| `scripts/install.sh` | ~34 | `REPO_URL_HTTPS="https://ghfast.top/https://github.com/..."` |
| `README.md` | Quick Install section | China mainland accelerated install command block |

If any of these are missing after merge, re-apply them before pushing.
