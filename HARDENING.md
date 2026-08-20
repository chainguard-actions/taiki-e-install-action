<!-- markdownlint-disable -->

# Hardening Report: taiki-e--install-action/v2.68.32

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **taiki-e--install-action/v2.68.32** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): ${{ matrix.* }} expressions are directly interpolated inside run: shell commands in ci.yml. In the 'test' job, the step 'Generate tool list' runs: `tools/ci/tool-list.sh "${{ matrix.tool }}" "${{ matrix.os }}" "${{ matrix.bash }}" >>"${GITHUB_OUTPUT}"` — three matrix context values are substituted directly into the shell command string before the shell parses it. In the 'test-container' job, the 'Install requirements (centos)' step contains `if [[ "${{ matrix.container }}" == "centos:6" ]]; then` inside a run: block, and the 'Generate tool list' step runs: `tools/ci/tool-list.sh "" "${{ matrix.container }}" >>"${GITHUB_OUTPUT}"`. All of these are script-injection violations because ${{ ... }} is expanded by the Actions runner before the shell sees the string, allowing a workflow-controlled matrix value to inject shell metacharacters.

Locations:

- `.github/workflows/ci.yml:75`
- `.github/workflows/ci.yml:232`
- `.github/workflows/ci.yml:278`

### github-env-injection (severity: high)

Two run: steps in ci.yml write matrix.* values to $GITHUB_OUTPUT without sanitization (no `printf '%s' ... | tr -d '\n\r'` applied before the write). The 'Generate tool list' step in the 'test' job passes ${{ matrix.tool }}, ${{ matrix.os }}, and ${{ matrix.bash }} directly as arguments to tools/ci/tool-list.sh which appends its output to $GITHUB_OUTPUT. Similarly, the 'Generate tool list' step in the 'test-container' job passes ${{ matrix.container }} to the same script which writes to $GITHUB_OUTPUT. A newline injected via a matrix value could define additional GITHUB_OUTPUT key-value pairs.

Locations:

- `.github/workflows/ci.yml:75`
- `.github/workflows/ci.yml:278`

### unpinned-uses (severity: high)

Multiple uses: references are pinned to mutable branch names or version tags instead of immutable 40-character commit SHAs, making the workflow vulnerable to supply-chain attacks if the referenced repository is compromised.

ci.yml:
- `taiki-e/github-actions/.github/workflows/miri.yml@main` (branch ref)
- `taiki-e/github-actions/.github/workflows/msrv.yml@main` (branch ref)
- `taiki-e/github-actions/.github/workflows/test.yml@main` (branch ref)
- `taiki-e/github-actions/.github/workflows/tidy.yml@main` (branch ref)
- `taiki-e/checkout-action@v1` (tag ref, appears multiple times)

manifest.yml:
- `taiki-e/github-actions/.github/workflows/gen.yml@main` (branch ref)

release.yml:
- `taiki-e/checkout-action@v1` (tag ref)
- `taiki-e/create-gh-release-action@v1` (tag ref)

Only `taiki-e/github-actions/.github/workflows/create-release.yml@853cebf868aa2dce1470668df24176803e05adc8` in release.yml is correctly pinned to a SHA.

Locations:

- `.github/workflows/ci.yml:35`
- `.github/workflows/ci.yml:38`
- `.github/workflows/ci.yml:39`
- `.github/workflows/ci.yml:43`
- `.github/workflows/ci.yml:68`
- `.github/workflows/manifest.yml:32`
- `.github/workflows/release.yml:22`
- `.github/workflows/release.yml:23`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection

**Notes:**

Fixed all three findings across ci.yml, manifest.yml, and release.yml:

1. unpinned-uses: Pinned all mutable branch/tag references to immutable commit SHAs. taiki-e/github-actions@main → @b34de8dc7e2e34cec74650ee3219d15e82f9f742, taiki-e/checkout-action@v1 → @7d1e50e93dc4fb3bba58f85018fadf77898aee8b, taiki-e/create-gh-release-action@v1 → @eba8ea96c86cca8a37f1b56e94b4d13301fba651. All original tag names preserved as inline comments.

2. script-injection: Moved all ${{ matrix.* }} expressions from run: shell strings into step env: blocks. Affected steps: 'Generate tool list' in test job (matrix.tool, matrix.os, matrix.bash), 'Install requirements (centos)' in test-container job (matrix.container), and 'Generate tool list' in test-container job (matrix.container).

3. github-env-injection: Added sanitization (printf '%s' "$VAR" | tr -d '\n\r') for all matrix values before they are passed to tool-list.sh which writes to $GITHUB_OUTPUT. Applied in both 'Generate tool list' steps.

