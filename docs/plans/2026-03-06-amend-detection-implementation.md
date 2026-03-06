# Amend Detection Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Modify the GitHub Action to amend previous commits when they only contain the game file with the same commit message, preventing repository bloat.

**Architecture:** Add amend detection logic in the "Commit and push" step of action.yml. Compare last commit message and files changed with incoming commit. If both match, amend with `--force-with-lease`; otherwise create new commit.

**Tech Stack:** GitHub Actions (composite action), Bash scripting

---

## Task 1: Update action.yml inputs

**Files:**
- Modify: `action.yml:9-35`

**Step 1: Add no-amend input**

Add after the `fps` input (line 28):

```yaml
  no-amend:
    description: 'Disable amending previous commits (always create new commits)'
    required: false
    default: 'false'
```

**Step 2: Remove write-dataurl-to input**

Delete lines 29-31:
```yaml
  write-dataurl-to:
    description: 'Write WebP as HTML <img> data URL to text file (mutually exclusive with output-path)'
    required: false
```

**Step 3: Update description**

Change line 2 from:
```yaml
description: 'Transform your GitHub contribution graph into a space shooter game GIF or embeddable HTML'
```
To:
```yaml
description: 'Transform your GitHub contribution graph into a space shooter game GIF'
```

**Step 4: Commit**

```bash
git add action.yml
git commit -m "feat(action): add no-amend input, remove write-dataurl-to"
```

---

## Task 2: Update generate step to remove dataurl support

**Files:**
- Modify: `action.yml:56-74`

**Step 1: Simplify generate step**

Replace the entire "Generate game" step with:

```yaml
    - name: Generate game
      id: generate
      shell: bash
      env:
        GH_TOKEN: ${{ inputs.github-token }}
      run: |
        gh-space-shooter ${{ inputs.username }} \
          --output ${{ inputs.output-path }} \
          --strategy ${{ inputs.strategy }} \
          --fps ${{ inputs.fps }}
        echo "output-file=${{ inputs.output-path }}" >> $GITHUB_OUTPUT
```

**Step 2: Commit**

```bash
git add action.yml
git commit -m "refactor(action): remove dataurl support from generate step"
```

---

## Task 3: Implement amend detection in commit step

**Files:**
- Modify: `action.yml:76-87`

**Step 1: Replace commit and push step**

Replace the entire "Commit and push" step with:

```yaml
    - name: Commit and push
      shell: bash
      run: |
        git config --local user.email "github-actions[bot]@users.noreply.github.com"
        git config --local user.name "github-actions[bot]"

        OUTPUT_PATH="${{ inputs.output-path }}"
        COMMIT_MSG="${{ inputs.commit-message }}"
        NO_AMEND="${{ inputs.no-amend }}"

        git add "$OUTPUT_PATH"

        # Check if we should amend the previous commit
        SHOULD_AMEND="false"
        if [ "$NO_AMEND" != "true" ]; then
          # Get last commit message (first line only)
          LAST_MSG=$(git log -1 --pretty=%s 2>/dev/null || echo "")
          # Get files changed in last commit (relative paths, no empty lines)
          LAST_FILES=$(git diff-tree --no-commit-id --name-only -r HEAD 2>/dev/null | grep -v '^$' || echo "")

          # Check if message matches and only output file was changed
          if [ "$LAST_MSG" = "$COMMIT_MSG" ] && [ "$LAST_FILES" = "$OUTPUT_PATH" ]; then
            SHOULD_AMEND="true"
          fi
        fi

        if [ "$SHOULD_AMEND" = "true" ]; then
          echo "Amending previous commit (same message and file)"
          git commit --amend --no-edit
          git push --force-with-lease
        else
          echo "Creating new commit"
          git diff --staged --quiet || git commit -m "$COMMIT_MSG"
          git push
        fi
```

**Step 2: Commit**

```bash
git add action.yml
git commit -m "feat(action): implement amend detection for space-shooter commits"
```

---

## Task 4: Update README.md action inputs documentation

**Files:**
- Modify: `README.md:47-54`

**Step 1: Update action inputs table**

Replace the action inputs section (lines 47-54) with:

```markdown
**Action Inputs:**
- `github-token` (required): GitHub token for fetching contributions
- `username` (optional): Username to generate game for (defaults to repo owner)
- `output-path` (optional): Where to save the animation, supports `.gif` or `.webp` (default: `gh-space-shooter.gif`)
- `strategy` (optional): Attack pattern - `column`, `row`, or `random` (default: `random`)
- `fps` (optional): Frames per second for the animation (default: `40`)
- `no-amend` (optional): Set to `true` to disable amending previous commits (default: `false`)
- `commit-message` (optional): Commit message for the update
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update action inputs for v2 (add no-amend, remove write-dataurl-to)"
```

---

## Task 5: Update README.md example workflow

**Files:**
- Modify: `README.md:34-39`

**Step 1: Update example to use v2**

Change line 34 from:
```yaml
      - uses: czl9707/gh-space-shooter@v1
```
To:
```yaml
      - uses: czl9707/gh-space-shooter@v2
```

**Step 2: Remove write-dataurl-to comment**

Delete line 38:
```yaml
          # write-dataurl-to: 'README.md'   # for dataurl generation.
```

**Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update example workflow to v2, remove dataurl example"
```

---

## Task 6: Add amend behavior note to README

**Files:**
- Modify: `README.md:43-45`

**Step 1: Add note after example workflow display**

After the markdown display example (around line 45), add:

```markdown
> **Note:** By default, the action amends the previous commit if it only contains the game file with the same commit message. This prevents repository bloat from daily commits. Set `no-amend: true` to disable this behavior.
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add note about amend behavior"
```

---

## Task 7: Remove data URL CLI documentation

**Files:**
- Modify: `README.md:133-176`

**Step 1: Remove "Generate Data URL" section**

Delete lines 133-176 (the entire "Generate Data URL (for embedding in HTML/Markdown)" section).

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: remove data URL CLI documentation"
```

---

## Task 8: Final verification

**Step 1: Review action.yml**

Run: `cat action.yml`
Expected: Complete action.yml with no-amend input, no write-dataurl-to, amended commit logic

**Step 2: Review README.md**

Run: `cat README.md`
Expected: Documentation reflects v2, no dataurl references in action section

**Step 3: Create summary commit if needed**

If any minor fixes were made:
```bash
git add -A
git commit -m "chore: final cleanup for v2 release"
```

---

## Release Checklist (after implementation)

- [ ] Create v2 git tag: `git tag v2`
- [ ] Push tag: `git push origin v2`
- [ ] Update GitHub release notes with breaking changes
