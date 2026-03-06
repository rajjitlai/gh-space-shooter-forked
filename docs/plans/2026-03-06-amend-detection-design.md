# Design: Amend Detection for Space Shooter Commits

**Date:** 2026-03-06
**Version Target:** v2 (major version bump)

## Problem

The GitHub Action commits a new GIF/WebP file daily, causing repository bloat over time. Each binary file is ~100KB-1MB, and after a year this adds significant size to the repository history.

## Solution

Amend the previous commit instead of creating a new one when:
1. The last commit message matches the incoming commit message
2. The last commit only modified the configured output file(s)

This keeps one commit per game update rather than accumulating daily commits.

## Design Details

### 1. Detection Logic

```
IF no-amend == false AND
   last_commit.message == incoming_commit.message AND
   last_commit.files == [output-path]
THEN
   amend the commit
ELSE
   create new commit
```

### 2. Action Input Changes

**Added:**
```yaml
no-amend:
  description: 'Disable amending previous commits (always create new commits)'
  required: false
  default: 'false'
```

**Removed:**
```yaml
write-dataurl-to:
  description: 'Write WebP as HTML <img> data URL to text file'
  # Removed - unreliable due to GitHub's ~1MB limit, use web app instead
```

### 3. Git Operations Flow

```
1. Generate the GIF/WebP file

2. Check if no-amend is false (default):
   ├── git log -1 --pretty=%s --name-only
   ├── Compare message with incoming commit-message
   └── Check if files changed == [output-path]

3. If amend conditions met:
   ├── git add <output-path>
   ├── git commit --amend --no-edit
   └── git push --force-with-lease

4. If amend conditions NOT met:
   ├── git add <output-path>
   ├── git commit -m "<commit-message>"
   └── git push
```

Use `--force-with-lease` instead of `--force` for safety — fails if remote was updated by someone else.

### 4. Error Handling

**Branch protection:**
- If `git push --force-with-lease` fails due to branch protection rules
- Fail with clear message suggesting:
  - Add `no-amend: true` to workflow, OR
  - Remove force push restriction from branch protection, OR
  - Use a separate branch for game output

**First run:**
- No special handling needed — detection naturally fails (last commit won't match)
- Creates new commit as expected

### 5. Versioning

- Bump from v1 to v2 (major version)
- Keep v1 tag for backward compatibility
- Users opt-in to v2 by updating `@v1` → `@v2` in their workflow

### 6. Documentation Updates

**action.yml:**
- Add `no-amend` input
- Remove `write-dataurl-to` input

**README.md:**
- Update action inputs table
- Update example workflow to use `@v2`
- Add note about amend behavior:
  > By default, the action amends the previous commit if it only contains the game file with the same commit message. This prevents repository bloat from daily commits. Set `no-amend: true` to disable this behavior.

## Files to Modify

1. `action.yml` — Add input, remove input, update implementation
2. `README.md` — Update documentation
3. Git tag `v2` on release
