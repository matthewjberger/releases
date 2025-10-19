---
marp: true
---

<!--
theme: gaia
class:
 - lead
 - invert
headingDivider: 2
paginate: true
-->

<!--
_class:
 - lead
 - invert
-->

# Release Strategy

Unified versioning and release management

<!--
Talking about our new release strategy. Moving from release-please to a more explicit approach with unified versioning. Let's start with how things work now.
-->

## Current: Main Branch

- Developer merges PR to main with conventional commits
- release-please analyzes commits since last release
- Determines version bump: `feat:` → minor, `fix:` → patch, `BREAKING CHANGE:` → major
- Opens/updates release PR (updates CHANGELOG.md and package.json)
- Merge creates prerelease tagged `v1.25.0-beta`
- Triggers `release/X.Y.x` branch creation
- Empty commit with `Release-As: X.Y.0` removes `-beta` suffix

<!--
Devs use conventional commits - feat, fix, chore, etc. release-please scans commits since last release and figures out the bump. Opens a PR updating changelog and package.json. Merge that PR and it creates a prerelease with -beta, like v1.25.0-beta. Triggers a workflow to create the release branch. Empty commit with Release-As strips the -beta and makes it stable.
-->

## Current: Hotfix Workflow

- Push to `release/**` branch
- release-please auto-bumps patch
- Creates release PR
- Merge creates stable release

<!--
For hotfixes, push to release branches and release-please auto-bumps patch. Automation's been helpful but has limitations we'll get into.
-->

## Current Challenges

Independent versioning per workspace

Package.json: 1.25.0-beta

Controls apps: 0.1.0

Firmware: 0.1.0

Explorer: 1.0.0

No single version number represents the suite of software

<!--
Main issue is compatibility tracking. Someone asks "what firmware works with explorer 1.0.0?" - can't give a straight answer because each workspace versions independently. Need a single version representing the whole system state.
-->

## Benefits Worth Preserving

Automated release process

Conventional commit based

Changelog generation included

Separate workflows for main and release branches

<!--
Current setup has things worth keeping. Automated changelogs and version updates save time. Conventional commits keep things consistent. Main and release branch workflows make sense. Want to preserve these while fixing the versioning issue.
-->

## New Strategy: Unified Versioning

All version strings synced:
- package.json at repo root
- controls workspace Cargo.toml
- firmware workspace Cargo.toml
- explorer workspace Cargo.toml
- cabinet-controller Cargo.toml

<!--
Simple: one version for everything. All 5 locations stay synced. Say "version 1.26.0" and you know exactly what you're getting across all components.
-->

## Release Process

When we want to make a release:

- Bump all version strings uniformly, update changelog, commit on main
- Create release branch - tags commit on main (e.g., `v1.26.0`) and creates branch `release/1.26.x`
- Switch to release branch and publish - creates GitHub release from the tag

<!--
Three steps. Bump all versions on main and commit. Create release branch - tags that commit with v1.26.0 and makes the branch from it. Switch to release branch and publish, which creates the GitHub release using that tag.
-->

## Hotfix Process

To apply hotfixes to release branches:

- Cherry-pick the SHA of the commit and bump the patch version
- Publish creates new tag on release branch (e.g., `v1.26.1`) and GitHub release

<!--
Two steps for hotfixes. Cherry-pick the commit and bump patch. Publish creates a new tag on the release branch plus the GitHub release.
-->

## What We Get

- Uniform semantic versioning
- Tags on main branch commits
- Release branches made from those tagged commits
- Ability to add hotfixes to release branches and publish new releases including them

<!--
End result: unified versioning everywhere, tags on main marking release points, release branches from those tags, clean hotfix workflow.
-->

## The makeline-release Tool

Custom tool called makeline-release handles the release workflow

Just commands wrap each step

Replaces release-please

<!--
Built makeline-release to handle version bumps, tagging, and branch creation. Wrapped in Just commands for convenience. Replaces release-please.
-->

## Commands: Version Bumps

```bash
# Update version in all 5 places
# And update CHANGELOG.md at repo root
just bump-minor-version          # 1.25.0 → 1.26.0
just bump-major-version          # 1.25.0 → 2.0.0
```

CHANGELOG.md automatically updated during bump step

Run on main only - release branches only get patch bumps via hotfix

<!--
One command updates all 5 version files and regenerates the changelog. Everything stays synced. Minor and major bumps only happen on main. Release branches only get patch bumps through the hotfix workflow.
-->

## Commands: Create Release Branch

```bash
# Tag current commit and create release branch
just create-release-branch       # Tag: v1.26.0, Branch: release/1.26.x

# Or with suffix (for variants/pre-releases)
just create-release-branch beta  # Tag: v1.26.0-beta, Branch: release/1.26.x-beta
```

<!--
Tags current commit on main, creates release branch from that tag, and switches you to that branch. Tag on main marks the release branch creation point. Suffix goes in both tag and branch names. Can create multiple release branches from same commit with different suffixes - each gets its own tag.
-->

## How Tagging Works

On main at commit `abc123` with version `1.26.0`:

- Creates tag `v1.26.0` pointing to `abc123` on main
- Creates branch `release/1.26.x` from `abc123`
- Tag on main and first commit on release branch are the same

Tag marks the point in main's history where the release was cut

<!--
When you create a release branch, you're on main at some commit with the version already bumped to 1.26.0. The command creates a tag pointing to that commit on main, then creates the release branch starting from that same commit. So the tag on main and the first commit of the release branch are identical. The tag stays on main marking where in main's history this release came from. Later when hotfixes get applied to the release branch, they diverge, but the tag on main still points to the original release point.
-->

## Commands: Publish Release

```bash
# You're already on the release branch after create-release-branch
# Initial release uses existing tag from main
just publish-release             # GitHub release v1.26.0
```

Development continues on main

<!--
After creating release branch you're already on it, so just run publish. For initial release, uses the tag created on main - no new tag. For hotfixes, creates a new tag on the release branch. Tags on main mark where release branches were created. Tags on release branches mark hotfix releases. Main keeps moving forward independently.
-->

## Commands: Hotfixes

```bash
# On release branch: find commits to backport
git log main --oneline

# Apply hotfix - cherry-picks and bumps patch version
just hotfix <sha>

# Publish release with hotfixes - creates new tag on release branch
just publish-release             # Tag: v1.26.1, GitHub release created
```

<!--
From release branch, find the commit SHA on main to backport. Run just hotfix - cherry-picks and bumps patch automatically. Then publish creates new tag on release branch and GitHub release.
-->

## Commands: Dry Run Mode

You can append -dry to any command:

```bash
just bump-minor-version-dry
just bump-major-version-dry
just create-release-branch-dry
just publish-release-dry
```

<!--
All commands support dry run. See what'll happen before committing to changes.
-->

## Release Validation

- Explorer displays unified version
- Validation widget verifies deployed binaries
- `just generate-hashes` creates executables.json

```json
{
  "version": "1.25.0",
  "executables": {
    "api": "0b4e6bf10c7a63ac...",
    "lifecycler_server": "b52551483b0dd6e1...",
    ...
  }
}
```

<!--
Explorer displays the unified version it was built with and the current system version after connecting to a device. The validation widget verifies release integrity using executables.json - a manifest created by running just generate-hashes that maps executable names to their SHA256 hashes. Explorer uses this to verify release artifacts match what was actually built.
-->

## CI Integration

Publishing with `release/**` branch name still triggers existing workflows:

- Firmware artifact build and upload
- Greengrass component compilation
- Greengrass component deployment
- Artifact uploading to GitHub release

<!--
Publishing from a release branch triggers existing CI - firmware artifacts, Greengrass component builds and deployments, artifact uploads. Nothing changes on the CI side.
-->

## Benefits

- Clear compatibility - single version tells you what works together
- Automated changelogs and version updates
- Development decoupled from releases (no release PRs)
- Explicit control over when to release
- Support for release variants (beta, hardware-specific, etc.)
- Release validation via hash verification
- Tags on main mark branch points, releases published from release branches

<!--
Key benefits: compatibility clarity through unified versioning, automated changelogs and version updates, no release PRs at all, explicit release timing, support for variants with suffixes, hash-based validation, tags on main mark where release branches start and all actual releases get published from release branches. Automation where useful, control where needed.
-->

## Questions?

<!--
That's the new release strategy. Questions?
-->
