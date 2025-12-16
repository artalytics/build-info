# Badge Automation Setup

## Overview

The `build-info` repository uses a **registry-driven centralized approach** for:
1. Syncing package DESCRIPTION files from all R package repos
2. Updating PR count badges
3. Auto-generating the README dashboard

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    artalytics/.github                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  .github/workflows/sync-build-info.yaml                  │    │
│  │  (Reusable workflow - called by all R package repos)     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Called by each R package on push to main
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  R Package Repositories                          │
│  (artcore, artutils, modGallery, etc.)                          │
│                                                                  │
│  .github/workflows/build-info.yaml (thin 12-line caller)        │
│  └── uses: artalytics/.github/.github/workflows/                │
│            sync-build-info.yaml@main                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Syncs DESCRIPTION files
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    artalytics/build-info                         │
│                                                                  │
│  registry.yaml          ◄── Single source of truth              │
│  ├── Package definitions                                        │
│  ├── Category structure                                         │
│  └── Repo mappings                                              │
│                                                                  │
│  package/               ◄── DESCRIPTION files (auto-synced)     │
│  ├── artcore/DESCRIPTION                                        │
│  ├── artutils/DESCRIPTION                                       │
│  └── ...                                                        │
│                                                                  │
│  badges/                ◄── PR count badge JSON files           │
│  ├── pr-draft-{pkg}.json                                        │
│  └── pr-ready-{pkg}.json                                        │
│                                                                  │
│  README.md              ◄── Auto-generated from registry        │
│                                                                  │
│  .github/workflows/                                             │
│  ├── update-pr-badges.yaml  (reads from registry.yaml)          │
│  └── generate-readme.yaml   (builds README from registry)       │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. Registry (`registry.yaml`)

The registry is the **single source of truth** for all packages:

```yaml
categories:
  core:
    title: Core Infrastructure
    description: Essential platform services
    order: 1
  # ... more categories

packages:
  - name: artcore
    repo: artalytics/artcore
    category: core
    description: Core platform infrastructure
    codecov_token: ABC123  # Optional
```

**Benefits:**
- Add a new package by editing one file
- Handles package name ≠ repo name (e.g., `artpixeltrace` repo formerly `pixelsense`)
- Categories and ordering defined in one place
- Workflows read from registry instead of hardcoding

### 2. Reusable Workflow (`artalytics/.github`)

The sync workflow is centralized in the org-level `.github` repo:

**Location:** `artalytics/.github/.github/workflows/sync-build-info.yaml`

**Features:**
- Extracts package name from DESCRIPTION (not repo name)
- Validates package isn't a template placeholder
- Cleans up old directories on package rename
- Called by thin 12-line workflows in each R package

### 3. Caller Workflow (each R package)

Each R package has a minimal workflow:

```yaml
name: "Sync: Build Info"

on:
  push:
    branches: [main]
    paths: [DESCRIPTION]
  workflow_dispatch:

jobs:
  sync:
    uses: artalytics/.github/.github/workflows/sync-build-info.yaml@main
    secrets:
      GH_PAT: ${{ secrets.GH_PAT }}
```

### 4. PR Badge Updates

**Workflow:** `.github/workflows/update-pr-badges.yaml`

- Runs every 15 minutes
- Reads package list from `registry.yaml`
- Queries GitHub API for open PRs
- Updates `badges/pr-draft-{pkg}.json` and `badges/pr-ready-{pkg}.json`

### 5. README Generation

**Workflow:** `.github/workflows/generate-readme.yaml`

- Triggers on registry or DESCRIPTION changes
- Reads `registry.yaml` for structure
- Reads `package/*/DESCRIPTION` for versions
- Generates complete README with badges

## Adding New Packages

### Step 1: Add to Registry

Edit `registry.yaml`:

```yaml
packages:
  # ... existing packages

  - name: newpackage
    repo: artalytics/newpackage
    category: tool  # or core, api, app, misc
    description: What this package does
    # Optional:
    subcategory: modules  # Only for 'app' category
    codecov_token: TOKEN123
```

### Step 2: Add Workflow to Package Repo

Create `.github/workflows/build-info.yaml` in the new package repo:

```yaml
name: "Sync: Build Info"

on:
  push:
    branches: [main]
    paths: [DESCRIPTION]
  workflow_dispatch:

jobs:
  sync:
    uses: artalytics/.github/.github/workflows/sync-build-info.yaml@main
    secrets:
      GH_PAT: ${{ secrets.GH_PAT }}
```

### Step 3: Push to Main

When the package pushes to main:
1. Workflow syncs DESCRIPTION to `build-info/package/{name}/`
2. Badge workflow creates PR count badges on next run
3. README workflow regenerates with new package

## Handling Package Renames

If a package is renamed (e.g., `pixelsense` → `artpixeltrace`):

1. Update the package's DESCRIPTION with new name
2. Update `registry.yaml` with new name/repo
3. Push to main - the sync workflow will:
   - Create new directory `package/artpixeltrace/`
   - Remove old directory `package/pixelsense/`
4. Badge/README workflows will use new name

You can optionally add an alias for documentation:
```yaml
- name: artpixeltrace
  repo: artalytics/artpixeltrace
  aliases: [pixelsense]  # Historical reference
```

## Directory Structure

```
build-info/
├── README.md              # Auto-generated dashboard
├── BADGE_AUTOMATION.md    # This file
├── registry.yaml          # Package registry (edit this!)
│
├── badges/                # Badge JSON files
│   ├── pr-draft-artcore.json
│   ├── pr-ready-artcore.json
│   ├── lint-badge.json
│   ├── R-CMD-check-badge.json
│   └── test-coverage-badge.json
│
├── package/               # DESCRIPTION files (auto-synced)
│   ├── artcore/DESCRIPTION
│   ├── artutils/DESCRIPTION
│   └── ...
│
└── .github/workflows/
    ├── update-pr-badges.yaml
    └── generate-readme.yaml
```

## Badge URL Format

Badges are served via shields.io endpoint:

```
https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/artalytics/build-info/main/badges/pr-draft-{package}.json
```

## Troubleshooting

### Package not appearing in README
1. Check package exists in `registry.yaml`
2. Verify DESCRIPTION synced to `package/{name}/`
3. Manually trigger README workflow

### PR counts not updating
1. Check `update-pr-badges.yaml` workflow runs
2. Verify package is in `registry.yaml`
3. Check GitHub API access for private repos

### Template placeholder errors
The sync workflow fails if DESCRIPTION contains `{r-package}` tokens.
This is intentional - update the package DESCRIPTION before pushing to main.

### Badges showing 404
1. Verify badge file exists in `badges/` directory
2. Check the URL uses `badges/` prefix (not root)
3. Wait for workflow to create initial badge files

## Monitoring

- **PR Badge Workflow:** [Actions → Update PR Count Badges](https://github.com/artalytics/build-info/actions/workflows/update-pr-badges.yaml)
- **README Workflow:** [Actions → Generate README](https://github.com/artalytics/build-info/actions/workflows/generate-readme.yaml)

## Manual Triggers

```bash
# Update PR badges now
gh workflow run update-pr-badges.yaml --repo artalytics/build-info

# Regenerate README now
gh workflow run generate-readme.yaml --repo artalytics/build-info
```
