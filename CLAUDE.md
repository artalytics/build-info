# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

The `.build-info` repository is a dedicated metadata and version tracking repository for the Artalytics Platform R packages. It serves as a centralized location for:

- Package version information and DESCRIPTION files
- Build status badges (R-CMD-check, test coverage, lint status)
- Cross-package dependency tracking
- GitHub Actions workflow status monitoring
- Version coordination across the multi-package ecosystem

## Repository Structure

```
.build-info/
├── package/                    # Package metadata directory
│   ├── artcore/               # Core platform tools
│   ├── artutils/              # Shared utilities toolkit
│   ├── artpipelines/          # Data processing pipelines
│   ├── artbenchmark/          # Performance benchmarking
│   ├── artopensea/            # OpenSea API integration
│   ├── artopenai/             # OpenAI API integration
│   ├── pixelsense/            # Image analysis tools
│   ├── pixelsense2/           # Enhanced image analysis
│   ├── modGallery/            # Gallery module
│   ├── modBrowse/             # Browse module
│   ├── modFrames/             # Frames module
│   ├── modUpload/             # Upload module
│   └── {r-package}/           # Package template
├── README.md                   # Package status dashboard
├── build-info.Rproj          # RStudio project file
├── *-badge.json              # Status badge configurations
└── .gitignore                # Git ignore rules
```

## Key Files

### Package Metadata
- `package/*/DESCRIPTION`: R package metadata files containing version numbers, dependencies, and package information
- Each package directory contains only the DESCRIPTION file used for version tracking

### Status Badges
- `lint-badge.json`: Lint status badge configuration
- `R-CMD-check-badge.json`: R CMD check status badge configuration
- `test-coverage-badge.json`: Test coverage badge configuration

### Documentation
- `README.md`: Comprehensive dashboard showing all package versions, coverage percentages, and CI/CD status badges

## Development Workflow

### Version Management
```bash
# Navigate to build-info repository
cd /path/to/.build-info

# Check current status
git status

# Update package version in DESCRIPTION file
# Edit package/[package-name]/DESCRIPTION

# Commit version changes
git add package/[package-name]/DESCRIPTION
git commit -m "Update [package-name] to version X.Y.Z"

# Push changes
git push origin main
```

### Badge Updates
The badge JSON files are used by GitHub Actions and external services to display current build status. These are typically updated automatically by CI/CD pipelines.

### Cross-Package Dependencies
The repository tracks dependencies between Artalytics packages:
- `artcore`: Foundation package for platform tools
- `artutils`: Depends on `artcore`, provides shared utilities
- Module packages (`mod*`): Depend on `artutils` and `artcore`
- Integration packages (`art*`): Specialized functionality with varying dependencies

## Commands

### Git Operations
```bash
# Clone the repository
git clone git@github.com:artalytics/build-info.git

# Check repository status
git status

# View recent commits
git log --oneline -10

# Check remote configuration
git remote -v
```

### R Development
```bash
# Open in RStudio (if available)
open build-info.Rproj

# Validate DESCRIPTION files
Rscript -e "tools::checkRdaFiles('package/[package-name]/DESCRIPTION')"
```

### Package Version Queries
```bash
# Check specific package version
grep "^Version:" package/artcore/DESCRIPTION

# List all package versions
find package -name "DESCRIPTION" -exec grep -H "^Version:" {} \;
```

## Architecture Notes

This repository follows a hub-and-spoke model where:
- Individual package repositories contain the actual source code
- This central repository maintains version coordination and build status
- GitHub shields.io badges pull version information from DESCRIPTION files here
- CI/CD pipelines across all repositories reference this for version consistency

The separation allows for:
- Centralized version tracking without code duplication
- Independent development of individual packages
- Coordinated releases across the platform
- Single source of truth for package metadata

## Integration Points

- **GitHub Actions**: CI/CD workflows reference badge configurations
- **README Badges**: Version shields pull from `package/*/DESCRIPTION` files
- **Package Releases**: Version updates trigger cross-repository synchronization
- **Dependency Management**: DESCRIPTION files maintain inter-package dependencies