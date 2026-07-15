Copyright 2026 Tawni Glover / Aequalia LLC. Licensed under Apache-2.0.

# pipeline-os.md — Pipeline Repo Generation Spec

You are an infrastructure and application security agent. Read this entire document before generating anything. Every decision here is final.

This document has one job: generate the pipeline repository. The pipeline repository is not an application. It is the supply chain authority for all projects that use Iterum. It is generated once, reviewed by the human maintainer, and then locked. After generation is complete and the human has filled and committed `canon.json`, this repo is sealed. Agents do not modify it again without explicit owner approval through CODEOWNERS.

---

## What you are building

A GitHub repository that serves as the supply chain authority for Iterum projects.

This spec generates an authority owned by the developer. The public `tawni-dev/iterum-pipeline` repository is the Iterum **Reference Authority** for learning, evaluation, demonstrations, and prototypes; long-lived and production projects should generate and maintain their own authority. Projects select the authority they trust through the configured canon URL.

It contains two things:

1. **`canon.json`** — a template with placeholder values the human fills manually after generation. This file is the source of truth for all approved GitHub Action SHAs, tool versions, base image digests, and dependency versions used in every project repo that runs Iterum.

2. **Supporting files** — CODEOWNERS, README, and contributing guidance that lock the repo and explain the filling process.

The agent generates the structure and the canon.json template. The human fills canon.json. No agent fills canon.json. Under the documented non-admin trust model, agents may propose later changes through PRs but cannot merge them.

---

## Bootstrap flow

1. Acknowledge you have read `pipeline-os.md`.
2. Ask whether the developer is using a GitHub organization or a personal account. The pipeline repo will be at `<ORG_OR_USER>/pipeline` or any name the developer chooses — confirm the target repo name before generating.
3. Generate the full repository scaffold.
4. After generation is complete, instruct the developer to fill canon.json manually using the instructions in `FILLING-CANON.md` before using any Iterum project.

Do not fill any value in canon.json. Do not suggest values. Do not look up SHAs. The human maintainer does this outside of any agent session.

---

## Repository layout

```
/
├── canon.json              # template — human fills manually, never agent-generated values
├── CODEOWNERS
├── FILLING-CANON.md        # step-by-step instructions for filling canon.json
├── CONTRIBUTING.md
└── README.md
```

---

## Generate these files

### `canon.json`

Generate with all values as placeholders. No exceptions. No looked-up values. No suggested versions.

```json
{
  "actions": {
    "actions/checkout": "<sha>",
    "actions/upload-artifact": "<sha>",
    "actions/download-artifact": "<sha>",
    "step-security/harden-runner": "<sha>",
    "docker/login-action": "<sha>",
    "docker/build-push-action": "<sha>",
    "aquasecurity/trivy-action": "<sha>",
    "anchore/sbom-action": "<sha>",
    "sigstore/cosign-installer": "<sha>",
    "github/codeql-action": "<sha>",
    "ossf/scorecard-action": "<sha>",
    "gitleaks/gitleaks-action": "<sha>"
  },
  "tools": {
    "grype": "<version>",
    "semgrep": "<version>",
    "gitleaks": "<version>",
    "checkov": "<version>",
    "trivy": "<version>",
    "zap": "<version>"
  },
  "base-images": {
    "node-alpine": "node:24-alpine@sha256:<digest>",
    "nginx-alpine": "nginx:alpine@sha256:<digest>"
  },
  "dependencies": {
    "backend": {
      "express": "<version>",
      "express-rate-limit": "<version>",
      "cors": "<version>",
      "helmet": "<version>",
      "zod": "<version>",
      "pg": "<version>"
    },
    "frontend": {
      "note": "All @angular/* packages version-aligned. Exact versions verified in frontend/package.json."
    }
  }
}
```

### `CODEOWNERS`

Place in `.github/CODEOWNERS`:

```
* @<OWNER>
/.github/CODEOWNERS @<OWNER>
```

Replace `<OWNER>` with the developer's GitHub username. Instruct the developer to replace this before committing.

The second line explicitly self-protects the CODEOWNERS file. Without it, the file itself can be modified without triggering the ownership rule on some GitHub plan configurations.

**CODEOWNERS alone does not block merges.** After pushing this file, the developer must also enable branch protection. Include this as a required post-generation step in the "After generation" section output.

### Branch protection — required after generation

CODEOWNERS assigns reviewers but does not enforce approval without a matching branch protection rule. After committing the generated files and filling canon.json, the developer must:

1. Go to the pipeline repo on GitHub
2. `Settings > Branches > Add branch protection rule`
3. Branch name pattern: `main`
4. Enable: **Require a pull request before merging**
5. Enable: **Require review from Code Owners**
6. Save

**Solo maintainer note:** GitHub does not allow a PR author to approve their own PR. For a solo personal repo, omitting "Do not allow bypassing the above settings" is the pragmatic default: the repository owner retains admin bypass, while CODEOWNERS + the PR requirement gates non-admin actors. Organizations with a second trusted maintainer should enable that setting to require PRs universally.

**Critical:** fill and commit canon.json before enabling branch protection. Once protection is on, direct pushes by non-admin actors are blocked; the repository owner retains admin bypass unless "Do not allow bypassing the above settings" is enabled.

### `FILLING-CANON.md`

Generate with the following content. The standalone `FILLING-CANON.md` in the Iterum repo is the source of truth — the substantive content must match it. If either diverges, the standalone file wins and the generated template must be updated to match.

---

**Filling canon.json**

This file is the supply chain authority for all your Iterum projects. Every value must be verified by you against the GitHub API or official release pages before you commit it. Do not copy values from generated output. Do not ask an agent to fill this file.

**Resolving GitHub Action SHAs**

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

**Resolving tool versions**

For each tool under `tools`, check the official release page and confirm the version string exactly as it appears in release artifacts:

- Grype: https://github.com/anchore/grype/releases
- Semgrep: https://github.com/semgrep/semgrep/releases
- Gitleaks: https://github.com/gitleaks/gitleaks/releases
- Checkov: https://github.com/bridgecrewio/checkov/releases
- Trivy: https://github.com/aquasecurity/trivy/releases
- ZAP: https://github.com/zaproxy/zaproxy/releases

**Resolving base image digests**

```bash
docker pull node:24-alpine
docker inspect node:24-alpine --format='{{index .RepoDigests 0}}'

docker pull nginx:alpine
docker inspect nginx:alpine --format='{{index .RepoDigests 0}}'
```

The digest format is `node:24-alpine@sha256:<64-char-hex>`. Paste the full string including the image name prefix.

**Resolving dependency versions**

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

**After filling**

1. Confirm no value contains `<sha>`, `<version>`, or `<digest>`.
2. Commit `canon.json` directly to `main` — do this before enabling branch protection.
3. Confirm `<OWNER>` in `.github/CODEOWNERS` is replaced with your GitHub username.
4. Enable branch protection on `main` (see Iterum README, Phase 1 Step 3).

Your pipeline repo is now the canon authority. You can run Iterum in any project repo and it will fetch from here.

If you need to update a value later — a new tool version, a rotated base image digest, an action SHA update — you can push directly to main as the repository owner (GitHub shows a warning but permits it), or open a PR. PRs from agents or other contributors require your Code Owner approval before merging. The protection gates non-admin actors, not you.

---

### `CONTRIBUTING.md`

```markdown
# Contributing

This repository is the supply chain authority for Iterum projects.

## Rules

- No PR may modify `canon.json` without explicit owner approval via CODEOWNERS.
- Under the documented non-admin trust model, agents may not merge changes to this repository.
- All SHA, version, and digest values must be verified by the human maintainer against the GitHub API or official release pages before a PR is approved.
- Do not open PRs to update canon.json from automated tooling without human verification of each changed value.

## Updating canon.json

Open a PR with the proposed changes. In the PR description, include:
- Which values changed and why
- The command used to verify each new SHA or version
- A confirmation that `npm audit` passes for any changed dependency versions

The maintainer reviews and approves. No merge without approval.
```

### `README.md`

```markdown
# pipeline

Supply chain authority for Iterum projects.

## What this is

This repository contains `canon.json` — the approved registry of GitHub Action SHAs, tool versions, base image digests, and dependency versions used by all projects generated with Iterum.

Every Iterum project fetches this file at bootstrap and at CI runtime before any other step. If the fetch fails, everything fails. No project repo contains a copy of canon.json. Under the documented non-admin trust model, agents may propose changes through PRs but cannot merge them.

## Setup

See `FILLING-CANON.md` for instructions on filling canon.json before use.

## Updating

Open a PR. CODEOWNERS requires explicit owner approval. No automated merge.
```

---

## Pipeline runtime requirements reference

Do not generate `.github/workflows/pipeline.yml` in the pipeline repo.

This repository is the authority repo only. At generation time its `canon.json` is still placeholders by design, so it cannot produce a valid pinned workflow yet. The 14-stage workflow defined in this document belongs to the **project repo** generated later by `iterum.md`, after the developer has filled and committed `canon.json`. (Stage numbering convention: canon-fetch is Stage 0; fourteen gated stages follow it, numbered 1–14.)

This document defines the canonical stage order and per-stage requirements. When `iterum.md` generates the project repo's `.github/workflows/pipeline.yml`, every requirement below must be reflected in the generated output.

```
canon-fetch → secrets → lint → test → dependency-allowlist → SAST → SCA → container+SBOM → IaC → scorecard → sign → deploy-staging → DAST → manual-gate → deploy-production
```

When generating the project repo workflow, each job must:
- Declare `needs:` referencing only its immediate predecessor(s)
- Declare `permissions:` explicitly — never inherited
- Declare `timeout-minutes:` explicitly — default `30` for most jobs, `60` for container build and DAST. Unbounded jobs violate SPVS 1.5 V3.1.1.
- Use only SHAs from canon.json for all `uses:` references
- Use only versions from canon.json `tools` for any pinned binary or scanner version

The canon-fetch job runs first with no `needs:`. All other jobs declare `needs: canon-fetch` at minimum, plus their stage predecessor. After generation, verify no stage was omitted and no floating tag appears in any `uses:` line.

---

### Stage 0 — canon-fetch

Runs first, no `needs:`. Fetches canon.json from the pipeline repo, validates no placeholders remain, exports canon values as job outputs, runs the canon-verify step, and captures spec provenance. All subsequent jobs depend on this job completing successfully.

If the pipeline repo is public, no authentication is required. If private, add `-H "Authorization: token ${{ secrets.PIPELINE_READ_TOKEN }}"` to the curl command.

```yaml
- name: Fetch canon.json
  run: |
    curl -fsSL "<CANON_URL>" -o .pipeline-canon.json
    if grep -q '<sha>\|<version>\|<digest>' .pipeline-canon.json; then
      echo "canon.json contains unfilled placeholders — aborting"
      exit 1
    fi

- name: Verify workflow SHAs and tool versions against canon
  run: |
    FAIL=0

    # SHA verification — every uses: line must be pinned to a 40-char hex SHA
    # approved for that specific action in canon.json
    while IFS= read -r line; do
      ACTION=$(echo "$line" | grep -oP '(?<=uses: )[^@\s]+' || true)
      SHA=$(echo "$line" | grep -oP '@[a-f0-9]{40}' | tr -d '@' || true)

      if [ -z "$ACTION" ]; then continue; fi

      if [ -z "$SHA" ]; then
        echo "CANON VIOLATION: unpinned or non-SHA ref — $line"
        FAIL=1
        continue
      fi

      APPROVED=$(jq -r --arg a "$ACTION" '.actions[$a] // empty' .pipeline-canon.json)
      if [ -z "$APPROVED" ]; then
        echo "CANON VIOLATION: action not in canon.json — $ACTION"
        FAIL=1
      elif [ "$APPROVED" != "$SHA" ]; then
        echo "CANON VIOLATION: $ACTION SHA $SHA does not match canon approved value $APPROVED"
        FAIL=1
      fi
    done < <(grep -rE 'uses:' .github/workflows/)

    # Tool version verification — every canon tool must have an explicit matching version pin
    for tool in grype semgrep gitleaks checkov trivy zap; do
      CANON_VER=$(jq -r --arg t "$tool" '.tools[$t] // empty' .pipeline-canon.json)
      if [ -z "$CANON_VER" ]; then
        echo "CANON VIOLATION: canon.tools.$tool is missing"
        FAIL=1
        continue
      fi
      FOUND=$(grep -rE "${tool^^}_VERSION|version:.*$tool" .github/workflows/ \
        | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1 || true)
      if [ -z "$FOUND" ]; then
        echo "CANON VIOLATION: $tool has no explicit version pin in workflows"
        FAIL=1
      elif [ "$FOUND" != "$CANON_VER" ]; then
        echo "CANON VIOLATION: $tool version $FOUND does not match canon approved $CANON_VER"
        FAIL=1
      fi
    done

    exit $FAIL

- name: Capture spec provenance
  run: |
    SPEC_SHA=$(git rev-parse HEAD:iterum.md 2>/dev/null || echo "not-tracked")
    echo "iterum.md commit: $SPEC_SHA" >> $GITHUB_STEP_SUMMARY
```

Note: the canon-verify step runs in-band inside the workflow it audits. An actor who can rewrite the workflow can delete this step. The project repo branch protection rule — requiring canon-fetch as a passing status check before any merge — is the structural backstop. Both controls are required.

**This script is the single source of truth for canon-verify.** iterum.md references this section rather than duplicating it. Generate this step verbatim into the project repo's canon-fetch job.

---

### Stage 1 — secrets (Gitleaks)

Two failure modes. Both mitigations required. They address distinct breaks and are not interchangeable.

**Failure mode A — shallow checkout:** Default checkout depth hides older commits. Gitleaks must scan full git history, not only HEAD.

```yaml
- uses: actions/checkout@<sha from canon.json>
  with:
    fetch-depth: 0
```

**Failure mode B — single commit / no parent:** On a repository with only one commit, ranges like `HEAD^..HEAD` produce a fatal ambiguous revision error. Use `--log-opts=--all` so the scanner does not rely on parent-dependent ranges.

Requires explicit `GITHUB_TOKEN` in `env` for `pull_request` events.

---

### Stage 2 — lint

Runs after secrets, before test.

- Frontend: `ng lint` plus Prettier `format:check`
- Backend: `npm run build` / `tsc` (type check, no emit)

---

### Stage 3 — test

Runs after lint, before dependency-allowlist.

**Backend unit tests (Vitest) — required coverage:**
- Every Zod validator: valid inputs, invalid inputs, boundary cases
- `buildErrorPayload`: RFC 9457 shape, `type: about:blank`, `status`, `code`, `boundary`, `tier` present; `searchTerm` absent from public payload
- `buildLogEntry`: UUID `searchTerm` generated, stable shape
- `SAFE_FALLBACK_PAYLOAD`: correct shape, no internal information

**Backend integration tests (Supertest) — abuse cases required:**
- Oversized body
- Malformed JSON
- Injection-style strings in all input fields
- Missing required fields
- Rate-limit enforcement — assert 429 response at configurable limit
- Assert responses never contain stack traces
- Assert responses never contain `searchTerm`
- All external APIs mocked via `vi.mock` — no real HTTP to third parties in CI

**Error system local verification — generate `scripts/test-error-logging.sh`:**
1. Sends a malformed request to the local backend
2. Prints the full public response
3. Prints the `searchTerm` from the internal log if accessible locally
4. Reminds the developer to search their observability platform for the matching `searchTerm`

---

### Stage 4 — dependency-allowlist

Runs after test, before SAST. CI must fail if any dependency outside the canon.json approved list is introduced. This is an actual CI gate — not prose policy.

Allowlist source: `dependencies` key in the fetched canon.json. Any package not present at the exact pinned version fails the gate immediately.

**Cache usage in this and all stages:** If `actions/cache` is used, the cache key must include the lockfile hash:

```yaml
key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

The `pull_request_target` prohibition is the primary defense against cache poisoning. Branch-scoped cache isolation is the platform control. Running the allowlist gate after any cache restore is the compensating control — it catches what slips through if a cache is restored.

---

### Stage 5 — SAST (Semgrep)

Injection flaws, insecure patterns, hardcoded secrets in logic. Version from canon.json `tools.semgrep`.

---

### Stage 6 — SCA

```bash
npm audit --omit=dev --audit-level=high   # frontend workspace
npm audit --omit=dev --audit-level=high   # backend workspace
```

`--omit=dev` is non-negotiable. Dev build toolchain CVEs are not shipped in production containers and must not fail production gates. (R05 — RESEARCH.md)

Grype scans lockfiles, not directories:

```bash
grype file:./frontend/package-lock.json
grype file:./backend/package-lock.json
```

Do not use `grype dir:` — walks `node_modules` and produces false positive Go stdlib CVEs from precompiled binaries such as esbuild. (R06 — RESEARCH.md)

Dependency age/cooldown gate: flag newly published package versions within the configurable cooldown window (default 72 hours).

Egress restriction via `step-security/harden-runner` on this job. SHA from canon.json. (R13 — RESEARCH.md)

---

### Stage 7 — container+SBOM (Trivy, Syft)

Use `aquasecurity/trivy-action`. Do not use `docker run` with the scanner image — not the supported integration for GitHub-hosted runners. (R07 — RESEARCH.md)

Build images once. Export as tarballs. Upload as `container-images` artifact (dotless name — artifact names containing dots cause upload failures with some GitHub Actions runner versions; discovered in build). The signing stage downloads these tarballs — it does not rebuild. Rebuilding produces a different digest than the one Trivy scanned. (R10 — RESEARCH.md)

Artifact names must not contain dots. On `anchore/sbom-action` use `upload-artifact: false` and upload manually via `actions/upload-artifact` with a dotless name.

Scanner image pinned by digest from canon.json. Container base images pinned by digest from canon.json.

Egress restriction via `step-security/harden-runner` on this job.

---

### Stage 8 — IaC (Checkov)

Fail on HIGH severity misconfigurations. Suppressions inside resource blocks only, no space between `#` and `checkov`:

```hcl
resource "resource_type" "name" {
  #checkov:skip=CKV_PROVIDER_XXX:Reason — e.g. non-prod uses smaller SKU by design
}
```

---

### Stage 9 — scorecard (OpenSSF)

Uses `ossf/scorecard-action` pinned to SHA from canon.json. Requires `id-token: write` to publish results. (R11 — RESEARCH.md)

```yaml
permissions:
  security-events: write
  id-token: write
  contents: read
```

Scorecard egress endpoints must be allowlisted in the `harden-runner` egress policy for this job. Findings surface in the GitHub Security tab. Treat as a continuous hygiene signal, not a hard gate. The generated scorecard job must set `continue-on-error: true`.

---

### Stage 10 — sign (cosign)

Push images to `ghcr.io` before signing. Log in with `GITHUB_TOKEN`. Job `permissions` must include `packages: write`. Sign the registry reference (`ghcr.io/owner/image@sha256:...`), not a local tag. (R14 — RESEARCH.md)

Downloads tarballs from stage 7 artifact. Does not rebuild images.

---

### Stage 11 — deploy-staging

Use OIDC for cloud authentication. OIDC tokens are scoped to a single workflow job and expire when that job completes. (R09 — RESEARCH.md)

```yaml
permissions:
  id-token: write
  contents: read
environment: staging
```

Configure the cloud provider trust relationship to accept the `repo:<org>/<repo>:environment:staging` OIDC claim.

---

### Stage 12 — DAST (OWASP ZAP baseline)

Runs after staging deploy. Gates on `DAST_TARGET_URL` secret. If secret is not set, log a notice and exit 0. Set the secret to the staging URL after first successful deploy. Never use `localhost` — CI runs in the cloud.

---

### Stage 13 — manual-gate

No auto-merge. Human approval required via GitHub environment protection.

---

### Stage 14 — deploy-production

`cosign verify` on promoted images before infrastructure apply. Gated on manual-gate passing. Use OIDC with `environment: production` claim.

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/<YOUR_ORG>/<YOUR_REPO>/.github/workflows/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/<YOUR_ORG>/<YOUR_IMAGE>@sha256:<digest>
```

---

### Prohibited triggers

`pull_request_target` is prohibited in all generated workflows. It runs with the base branch's secrets even for PRs from forks — an attacker can trigger the full privileged pipeline from an external PR. Use `pull_request` (unprivileged) for all code review triggers. (R03 — RESEARCH.md)

```yaml
# DO NOT GENERATE
on:
  pull_request_target:
```

---

### Tag protection

Release tags must be protected. Configure tag protection rules in repository settings:

- Protected pattern: `v*`
- Only pipeline service accounts and designated maintainers may create protected tags

This prevents an attacker with limited repo access from creating or moving a fake release tag.

---

## After generation — instruct the developer

When all authority-repo files are generated, output the following:

---

**Pipeline repo generated. Before you use Iterum, complete these steps in order:**

1. Replace `<OWNER>` in `.github/CODEOWNERS` with your GitHub username. Commit directly to main.
2. Fill every value in `canon.json` using `FILLING-CANON.md`. Commit directly to main.
3. Enable branch protection on main:
   - `Settings > Branches > Add branch protection rule`
   - Branch name pattern: `main`
   - Enable: Require a pull request before merging
   - Enable: Require review from Code Owners
   - Save

**Why this order matters:** Fill and commit canon.json before enabling branch protection. Once branch protection is active, direct pushes by non-admin actors are blocked — but as the repository owner (admin), you retain the ability to push directly to main; GitHub will display a warning but permit it. What protection actually gates is every non-admin write actor: agents, collaborators, bot tokens, and compromised CI identities. That is precisely the threat model. Future canon updates you make yourself proceed via direct push or via PR; PRs opened by agents or contributors require your explicit Code Owner approval before merging.

If you are working in a GitHub organization with a second trusted maintainer, also enable "Do not allow bypassing the above settings" to remove the admin bypass and require PRs universally.

**Your canon URL will be:**
```
https://raw.githubusercontent.com/<YOUR_ORG_OR_USER>/<PIPELINE_REPO_NAME>/main/canon.json
```

If this repo is public, Iterum projects fetch from it with no authentication required.
If this repo is private, create a fine-grained PAT scoped to read-only contents on this repo and store it as `PIPELINE_READ_TOKEN` in each project repo's secrets.

Once canon.json is filled and committed, you are ready to run Iterum in a project repo.

## Known gaps

**A valid SHA is not an approved SHA.** CODEOWNERS prevents unauthorized merges but does not prevent an owner from committing a wrong value. Every value in canon.json is only as trustworthy as the verification process used to produce it. The commands in `FILLING-CANON.md` are the verification process — skipping them defeats the authority model.

**Floating tags are not caught if the SHA resolves.** A SHA that was valid at verification time may point to different code if the upstream tag is moved. Periodic re-verification of canon.json entries against the GitHub API is the maintainer's responsibility. Set a calendar reminder.

**Canon fetch uses a mutable branch reference.** The canon URL uses `main` — a mutable ref. The trust anchor is consumed via a floating branch with no integrity check beyond TLS and a placeholder grep. CODEOWNERS and branch protection mitigate the PR path, but a compromised owner account or token can push directly to main and every downstream project consumes the change on the next run. Two alternatives were considered and rejected for v1.0: (a) pinning the fetch to a specific commit SHA per project — adds maintenance overhead and does not self-update on legitimate rotation; (b) `cosign sign-blob` + verify at canon-fetch — the most robust approach but adds Sigstore as a hard dependency and requires key or identity setup outside current scope. Adopters in high-assurance environments should implement cosign sign-blob verification. This is a planned control for a future version.

---

*the pipeline cannot catch decisions that were never made.*
*the architecture is the security.*
