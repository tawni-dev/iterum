# Filling canon.json

This file is the supply chain authority for all your Iterum projects. Every value must be verified by you against the GitHub API or official release pages before you commit it. Do not copy values from generated output. Do not ask an agent to fill this file.

## Resolving GitHub Action SHAs

Every SHA in `canon.json` under `actions` is the full commit SHA of a specific tagged release of that action. You must determine whether a tag is annotated or lightweight before resolving it, because the resolution path differs.

**How to tell:** Run `gh api repos/<owner>/<repo>/git/refs/tags/<tag> --jq '.object.type'`
- Returns `"commit"` — lightweight tag. The `.object.sha` is already the commit SHA. Use it directly.
- Returns `"tag"` — annotated tag. The `.object.sha` is the tag object SHA, not the commit SHA. You must dereference it.

**For annotated tags:**

```bash
# Step 1: get the tag ref object SHA
TAG_SHA=$(gh api repos/<owner>/<repo>/git/refs/tags/<tag> --jq '.object.sha')

# Step 2: dereference to the commit SHA
COMMIT_SHA=$(gh api repos/<owner>/<repo>/git/tags/$TAG_SHA --jq '.object.sha')

echo $COMMIT_SHA
```

**Reliable method that handles both types automatically — recommended:**

```bash
git ls-remote https://github.com/<owner>/<repo> "refs/tags/<tag>^{}" | cut -f1
```

The `^{}` suffix peels the tag to the underlying commit regardless of tag type. If no output is returned, the tag is lightweight — use `git ls-remote` without `^{}` instead:

```bash
git ls-remote https://github.com/<owner>/<repo> "refs/tags/<tag>" | cut -f1
```

Run this for each action in canon.json. Verify the result is a 40-character hex string. A truncated, guessed, or tag-object SHA (rather than commit SHA) will cause immediate workflow failure.

## Resolving tool versions

For each tool under `tools`, check the official release page and confirm the version string exactly as it appears in release artifacts:

- Grype: https://github.com/anchore/grype/releases
- Semgrep: https://github.com/semgrep/semgrep/releases
- Gitleaks: https://github.com/gitleaks/gitleaks/releases
- Checkov: https://github.com/bridgecrewio/checkov/releases
- Trivy: https://github.com/aquasecurity/trivy/releases
- ZAP: https://github.com/zaproxy/zaproxy/releases

## Resolving base image digests

```bash
docker pull node:24-alpine
docker inspect node:24-alpine --format='{{index .RepoDigests 0}}'

docker pull nginx:alpine
docker inspect nginx:alpine --format='{{index .RepoDigests 0}}'
```

The digest format is `node:24-alpine@sha256:<64-char-hex>`. Paste the full string including the image name prefix.

## Resolving dependency versions

For each backend dependency, check the current version on npm:

```bash
npm view <package> version
```

To verify a version is audited clean, install it in a scratch directory and run `npm audit`:

```bash
mkdir /tmp/audit-check && cd /tmp/audit-check
npm init -y
npm install --ignore-scripts <package>@<version>
npm audit --audit-level=high
cd - && rm -rf /tmp/audit-check
```

A clean audit at `--audit-level=high` is required before adding the version to canon.json.

## After filling

1. Confirm no value contains `<sha>`, `<version>`, or `<digest>`.
2. Commit `canon.json` directly to `main` — do this before enabling branch protection.
3. Confirm `<OWNER>` in `.github/CODEOWNERS` is replaced with your GitHub username.
4. Enable branch protection on `main` (see Iterum README, Phase 1 Step 3).

Your pipeline repo is now the canon authority. You can run Iterum in any project repo and it will fetch from here.

If you need to update a value later — a new tool version, a rotated base image digest, an action SHA update — you can push directly to main as the repository owner (GitHub shows a warning but permits it), or open a PR. PRs from agents or other contributors require your Code Owner approval before merging. The protection gates non-admin actors, not you.
