# Badge Automation Setup

## Overview

The `.build-info` repository uses a **centralized pull-based approach** for updating PR count badges. A single GitHub Actions workflow runs in this repository and queries all package repositories via the GitHub API.

## Architecture

```
┌─────────────────────────────────────────┐
│      .build-info Repository             │
│                                         │
│  ┌───────────────────────────────┐     │
│  │  GitHub Actions Workflow       │     │
│  │  (runs every 15 minutes)       │     │
│  └───────────────────────────────┘     │
│              │                          │
│              ├─ Query PR counts via API │
│              ├─ Update JSON files       │
│              └─ Commit changes          │
│                                         │
│  pr-draft-*.json  pr-ready-*.json      │
│  (34 badge endpoint files)             │
└─────────────────────────────────────────┘
         │
         │ GitHub API queries
         ↓
┌─────────────────────────────────────────┐
│   All 17 Package Repositories           │
│   (artcore, artutils, modGallery, etc)  │
│                                         │
│   No workflows needed!                  │
└─────────────────────────────────────────┘
```

## Benefits of Centralized Approach

### 1. Zero Configuration Per Repository
- **No workflows added** to individual package repositories
- **No secrets needed** in each repo
- **No maintenance burden** across 17+ repositories

### 2. Centralized Control
- Single workflow file to maintain: `.github/workflows/update-pr-badges.yaml`
- Easy to update query logic or add new packages
- Consistent behavior across all packages

### 3. Efficient API Usage
- Queries all repos in a single workflow run
- Uses GitHub's built-in `GITHUB_TOKEN` (automatically provided)
- Stays well within API rate limits (5,000 requests/hour for authenticated requests)

### 4. Real-time Updates
- Runs every 15 minutes (configurable)
- Can be manually triggered via workflow dispatch
- Automatically commits changes only when counts change

### 5. Scalable
- Easy to add new packages (just add to array in workflow)
- Can extend to track other metrics (issues, discussions, etc.)
- No per-repo overhead as you scale

## How It Works

### Step 1: Scheduled Trigger
The workflow runs on a cron schedule:
```yaml
schedule:
  - cron: '*/15 * * * *'  # Every 15 minutes
```

You can adjust frequency:
- `*/5 * * * *` - Every 5 minutes (very responsive)
- `*/30 * * * *` - Every 30 minutes (moderate)
- `0 * * * *` - Every hour (conservative)

### Step 2: Query GitHub API
For each package, the workflow uses `gh` CLI to query PRs:
```bash
gh pr list \
  --repo "artalytics/$package" \
  --state open \
  --json isDraft \
  --limit 100
```

### Step 3: Count and Categorize
Using `jq`, the workflow separates PRs into:
- **Draft PRs** (`isDraft == true`)
- **Ready PRs** (`isDraft == false`)

### Step 4: Update JSON Files
Creates/updates badge endpoint files:
```json
{
  "schemaVersion": 1,
  "label": "draft",
  "message": "3",
  "color": "yellow"
}
```

### Step 5: Commit Changes
Only commits if counts have changed, with `[skip ci]` to prevent recursive workflows.

## Setup Instructions

### 1. Create Workflow Directory
```bash
cd .build-info
mkdir -p .github/workflows
```

### 2. Add Workflow File
The workflow file is already created at:
`.github/workflows/update-pr-badges.yaml`

### 3. Push to GitHub
```bash
git add .github/workflows/update-pr-badges.yaml
git commit -m "feat: add centralized PR badge automation"
git push
```

### 4. Verify Workflow
1. Go to GitHub: `https://github.com/artalytics/build-info/actions`
2. You should see "Update PR Count Badges" workflow
3. Click "Run workflow" to trigger manually
4. Verify JSON files are updated with actual counts

### 5. Configure Schedule (Optional)
Edit `.github/workflows/update-pr-badges.yaml` to adjust frequency:
- Line 5: Change cron schedule
- Consider API rate limits vs. freshness needs

## Adding New Packages

When adding a new package to the platform:

1. **Add to workflow array:**
   ```bash
   # Edit .github/workflows/update-pr-badges.yaml
   packages=(
     "existing-package"
     "new-package-name"  # Add here
   )
   ```

2. **Create initial JSON files:**
   ```bash
   # Optional: create empty files to avoid 404s
   echo '{"schemaVersion":1,"label":"draft","message":"0","color":"yellow"}' > pr-draft-newpackage.json
   echo '{"schemaVersion":1,"label":"ready","message":"0","color":"green"}' > pr-ready-newpackage.json
   ```

3. **Update README.md:**
   Add package row to appropriate architectural layer table.

4. **Commit and push:**
   The workflow will automatically populate counts on next run.

## Monitoring and Troubleshooting

### Check Workflow Runs
Visit: `https://github.com/artalytics/build-info/actions/workflows/update-pr-badges.yaml`

### Common Issues

**Issue: Counts not updating**
- Check if workflow is enabled in repository settings
- Verify `GITHUB_TOKEN` has correct permissions
- Check workflow logs for API errors

**Issue: 404 errors for some repos**
- Verify repository name spelling in workflow
- Ensure `GITHUB_TOKEN` has access to private repos
- Check if repo has been renamed or archived

**Issue: API rate limit exceeded**
- Reduce cron frequency (e.g., every hour instead of every 15 min)
- The standard `GITHUB_TOKEN` provides 5,000 requests/hour

**Issue: Badges showing "invalid"**
- Check JSON file syntax (must be valid JSON)
- Verify shields.io endpoint URL format
- Ensure files are in repository root, not subdirectory

### Manual Updates

You can manually trigger the workflow:
```bash
# Using gh CLI
gh workflow run update-pr-badges.yaml --repo artalytics/build-info

# Or via GitHub UI
# Navigate to Actions → Update PR Count Badges → Run workflow
```

## Alternative Approaches Considered

### ❌ Per-Repository Workflows
**Why not:** Would require adding/maintaining workflows in 17+ repos, increasing complexity and maintenance burden.

### ❌ External Service/Cron Job
**Why not:** Requires infrastructure outside GitHub, secrets management, and additional cost/maintenance.

### ❌ Real-time Webhooks
**Why not:** Overly complex for this use case; 15-minute updates are sufficiently fresh.

### ✅ Centralized Pull-based (Current)
**Why yes:** Single point of control, no per-repo overhead, efficient, scalable, uses native GitHub features.

## Performance Characteristics

- **Workflow runtime:** ~30-60 seconds for all 17 packages
- **API calls:** 17 per run (one per package)
- **Git operations:** Only when counts change (minimal)
- **Badge refresh:** Shields.io caches for ~5 minutes
- **End-to-end latency:** 15-20 minutes worst case (cron interval + shields cache)

## Security Considerations

- Uses GitHub's built-in `GITHUB_TOKEN` (no custom secrets needed)
- Token has `contents: write` permission (required for commits)
- Only queries public API endpoints (PR lists)
- Commits signed as `github-actions[bot]`
- No sensitive data in JSON files (just counts)

## Future Enhancements

Potential additions to consider:

1. **Issue counts** - Though standard shields.io badges work fine
2. **Last commit date** - Track repo activity
3. **Test coverage trends** - Historical coverage tracking
4. **Stale PR alerts** - Flag PRs open > 30 days
5. **Dependency status** - Track outdated dependencies
6. **Badge status dashboard** - Single page showing all metrics

All of these can be added to the existing centralized workflow without touching individual repos.
