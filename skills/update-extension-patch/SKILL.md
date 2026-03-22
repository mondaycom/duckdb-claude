---
name: update-extension-patch
description: Update an extension patch for a PR-related bug instead of editing patched output directly
disable-model-invocation: true
---

# Update Extension Patch for PR-Related Bug

Only follow this flow when the bug is in an extension and is related to the current PR. Update the patch file rather than modifying already-patched generated output.

## Step 1: Identify the extension

Find the relevant extension config file in `.github/config/extensions/` and note:
- The extension repository URL
- The pinned commit hash (tag)
- Whether `APPLY_PATCHES` is already set

## Step 2: Clone the extension repository

```bash
git clone <extension-repo> /tmp/<extension-name>
```

## Step 3: Enable patching if needed

If the extension config does not already have `APPLY_PATCHES`, add it to the config file at `.github/config/extensions/<extension-name>`.

## Step 4: Check out the pinned commit

In the cloned extension repository:
```bash
cd /tmp/<extension-name>
git checkout <pinned-commit-hash>
```

Use the pinned commit hash found in Step 1.

## Step 5: Apply all existing patches

Apply every existing patch from `.github/patches/extensions/` for this extension, in order:
```bash
git apply /path/to/duckdb/.github/patches/extensions/<extension-name>/<patch-file>.patch
```

Make sure all existing patches apply cleanly before proceeding.

## Step 6: Make the required code changes

Edit the code in the cloned extension to fix the bug.

## Step 7: Generate a new patch

```bash
git diff > /path/to/duckdb/.github/patches/extensions/<extension-name>/fix.patch
```

## Step 8: Verify

Review the generated patch to confirm it contains only the intended changes. Replace the existing patch file if updating an existing fix, or add the new patch file if it's a new fix.

## Important

- Always apply all existing patches before making changes, so the new patch layers correctly.
- The patch must be generated against the pinned commit with all prior patches applied.
- Do not edit patched output files directly — always update the patch source.
