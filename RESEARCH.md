Copyright 2026 Tawni Glover / Aequalia LLC. Licensed under Apache-2.0.

# RESEARCH.md — Claims, Verification, and Bibliography

This document records the factual security claims selected for verification from iterum.md and pipeline-os.md, the source used to verify each one, the date verified, and the finding. It exists so that the security decisions in this project are traceable — not just to the author's experience but to authoritative external sources.

**Last verified:** 2026-07-07

Nineteen claims were independently fact-checked against GitHub, OWASP, npm, IETF, Sigstore, Anchore, Aqua Security, OpenSSF, StepSecurity, and SLSA primary sources. Eighteen are VERIFIED. One is PARTIALLY VERIFIED: R12 (SPVS 1.5 is a community feedback draft — a factual status not resolvable by further research; the controls exist and the language is accurate paraphrase). None are outright incorrect.

Where a claim originates from direct build experience (e.g., grype false positives, Trivy action behavior), it is marked as such and distinguished from externally verifiable claims.

---

## Format

Each entry:
- **Claim** — the specific assertion made in iterum.md or pipeline-os.md
- **Where it appears** — file and section
- **What was checked** — the specific thing verified
- **Source** — URL to authoritative primary source
- **Date verified** — June 2026
- **Finding** — VERIFIED / PARTIALLY VERIFIED / INCORRECT, with notes

---

## R01 — CODEOWNERS does not block merges without branch protection

**Claim:** CODEOWNERS assigns reviewers but does not block merges on its own. Branch protection must also be enabled — specifically "Require review from Code Owners" — or PRs can merge regardless of CODEOWNERS.

**Where it appears:** iterum.md (Branch protection setup, DECISIONS table), pipeline-os.md (CODEOWNERS section, Branch protection section), README.md (Phase 1 Step 3)

**What was checked:** Whether CODEOWNERS has independent merge enforcement or requires a companion branch protection rule.

**Source:** https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners

**Date verified:** June 2026

**Finding:** VERIFIED. GitHub documentation confirms CODEOWNERS alone only requests reviews. Merge enforcement requires enabling "Require review from Code Owners" in a branch protection rule or repository ruleset. This is a recognized misconfiguration tracked by Prisma Cloud and others as a CI/CD security gap.

---

## R02 — Annotated vs lightweight tag SHA resolution

**Claim:** The `git/refs/tags` endpoint returns the tag object SHA for annotated tags, not the commit SHA. GitHub Actions pins to commit SHAs. The resolution path differs between annotated and lightweight tags — lightweight tags already point to the commit, annotated tags require a second dereference step. The `git ls-remote` with `^{}` peel handles both types.

**Where it appears:** pipeline-os.md (FILLING-CANON.md section, Resolving GitHub Action SHAs), iterum.md (canon.json schema section)

**What was checked:** GitHub REST API behavior for annotated vs lightweight tags and the correct command to retrieve a usable commit SHA.

**Sources:**
- https://docs.github.com/en/rest/git/refs
- https://docs.github.com/en/rest/git/tags ("This tags API only handles annotated tag objects — so only annotated tags, not lightweight tags")

**Date verified:** June 2026

**Finding:** VERIFIED with correction applied. The original spec only documented the annotated-tag path. Correction: lightweight tags return the commit SHA directly from `git/refs/tags` (`.object.type` = `"commit"`). Annotated tags return a tag object SHA (`.object.type` = `"tag"`) and require a second call to `git/tags/{tag_sha}`. The `git ls-remote` with `^{}` peel suffix handles both cases and is the recommended approach. Spec updated to branch on tag type.

---

## R03 — `pull_request_target` runs with base branch secrets for fork PRs

**Claim:** `pull_request_target` runs in the context of the base branch even for PRs from forks, making base branch secrets available to fork-originated code. An external attacker can trigger the full privileged pipeline via a fork PR.

**Where it appears:** iterum.md (Pipeline section — prohibited triggers, DECISIONS table), pipeline-os.md (Prohibited triggers section)

**What was checked:** Whether `pull_request_target` exposes base branch secrets to fork PR workflows.

**Sources:**
- https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository
- https://www.wiz.io/blog/github-actions-security-guide
- https://owasp.org/www-project-top-10-ci-cd-security-risks/ (CICD-SEC-4: Poisoned Pipeline Execution)

**Date verified:** June 2026

**Finding:** VERIFIED. GitHub documentation confirms `pull_request_target` runs "in the context of the base branch" regardless of PR source. The exposure is realized when the workflow additionally checks out and executes untrusted fork code. Note: GitHub introduced a partial mitigation in December 2025 such that `pull_request_target` always uses the default branch as its workflow source, reducing but not eliminating the attack surface. The prohibition in this spec remains correct.

---

## R04 — OWASP Top 10 2025 category labels

**Claim:** A02:2025 Security Misconfiguration, A03:2025 Software Supply Chain Failures (new), A09:2025 Security Logging and Alerting Failures, A10:2025 Mishandling of Exceptional Conditions (new).

**Where it appears:** iterum.md (OWASP controls table)

**What was checked:** Whether the four named categories are at their stated positions in the published OWASP Top 10 2025 list.

**Sources:**
- https://owasp.org/Top10/2025/
- https://owasp.org/Top10/2025/0x00_2025-Introduction/

**Date verified:** June 2026

**Finding:** VERIFIED. All four labels and positions are correct. A03:2025 Software Supply Chain Failures is an expansion of A06:2021 Vulnerable and Outdated Components. A10:2025 Mishandling of Exceptional Conditions contains 24 CWEs focusing on improper error handling. Note: SSRF (previously A10:2021) was consolidated into A01:2025 Broken Access Control — any legacy SSRF mapping from 2021 should be repointed.

---

## R05 — `npm audit --omit=dev --audit-level=high` behavior

**Claim:** `--omit=dev` excludes dev dependency CVEs from audit results. Dev build toolchain CVEs are not shipped in production containers and must not fail production gates.

**Where it appears:** iterum.md (pipeline SCA stage, DECISIONS table), pipeline-os.md (Stage 6)

**What was checked:** npm audit flag behavior for `--omit=dev` and `--audit-level`.

**Source:** https://docs.npmjs.com/cli/v11/commands/npm-audit/

**Date verified:** June 2026

**Finding:** VERIFIED. npm documentation confirms `--omit=dev` skips dev dependencies from the report. `--audit-level=high` sets the minimum severity for a non-zero exit code without filtering the output. The rationale — dev toolchain packages are not shipped in production containers — is sound and is a widely adopted CI/CD practice.

---

## R06 — `grype dir:` produces false-positive Go stdlib CVEs from esbuild

**Claim:** Scanning with `grype dir:` walks `node_modules` and produces false-positive Go stdlib CVEs from precompiled binaries such as esbuild. Use `grype file:` against lockfiles instead.

**Where it appears:** iterum.md (DECISIONS table), pipeline-os.md (Stage 6)

**What was checked:** Whether grype scanning a directory containing esbuild's precompiled binary produces Go stdlib CVE matches.

**Sources:**
- https://github.com/evanw/esbuild/issues/4076 (and related issues #4133, #4204, #3802)
- https://github.com/anchore/grype/issues/1562 (Go stdlib matching behavior across grype versions)

**Date verified:** June 2026

**Finding:** VERIFIED from build experience and upstream issue tracking. Grype and Trivy both detect Go versions compiled into the esbuild binary at `node_modules/@esbuild/.../bin/esbuild` and report Go stdlib CVEs (e.g. CVE-2025-22866, CVE-2025-22871, CVE-2023-45288) against it. These CVEs are real Go stdlib issues but unreachable in the build-tool context and not shipped to production — correctly treated as build-toolchain noise. Whether they are "false positives" depends on context; from a production gate perspective they are. `grype file:` against lockfiles avoids this class of result.

---

## R07 — `aquasecurity/trivy-action` vs `docker run` for Trivy in CI

**Claim:** Use `aquasecurity/trivy-action` for Trivy in GitHub Actions. The `docker run` approach is not the supported integration for hosted runners.

**Where it appears:** iterum.md (DECISIONS table), pipeline-os.md (Stage 7)

**What was checked:** Whether `trivy-action` is the officially recommended integration for GitHub Actions.

**Sources:**
- https://github.com/aquasecurity/trivy-action
- https://trivy.dev/docs/latest/ecosystem/cicd/

**Date verified:** June 2026

**Finding:** VERIFIED. `aquasecurity/trivy-action` is the official GitHub Actions integration for Trivy, listed explicitly in Trivy's CI/CD ecosystem documentation as the GitHub Actions integration. The action installs Trivy via `aquasecurity/setup-trivy` and supports all scan modes including tarball image scanning, which is the pattern the spec requires. The `docker run` approach is not the supported CI integration path — Aqua's own documentation directs GitHub Actions users to `trivy-action` exclusively. The original unverified causal claim ("registry auth behavior") was removed in a prior pass; the current spec language ("not the supported integration for GitHub-hosted runners") is accurate and documented.

---

## R08 — Cache poisoning via `pull_request_target` (TanStack / TeamPCP)

**Claim:** A poisoned cache can inject malicious packages into a later privileged workflow run without detection. Cache keys must include the lockfile hash. The `pull_request_target` prohibition is the primary defense.

**Where it appears:** iterum.md (Pipeline section — cache poisoning prevention, DECISIONS table), pipeline-os.md (Stage 4)

**What was checked:** Whether cache poisoning via `pull_request_target` is a documented real-world attack vector.

**Sources:**
- https://tanstack.com/blog/npm-supply-chain-compromise-postmortem (first-party postmortem)
- https://safedep.io/tanstack-github-actions-cache-poisoning/
- GitHub Security Advisory GHSA-g7cv-rxg3-hmpx
- https://adnanthekhan.com/posts/angular-compromise-through-dev-infra/ (prior research, $31,337 Google bounty)

**Date verified:** June 2026

**Finding:** VERIFIED. On 2026-05-11 the TeamPCP threat group published 84 malicious versions across 42 @tanstack/* npm packages via a chain of: `pull_request_target` Pwn Request pattern → GitHub Actions cache poisoning across the fork/base trust boundary → OIDC token extraction from the runner process. The TanStack postmortem documents that `actions/cache@v5`'s post-job save is not gated by permissions, allowing a fork PR to poison cache entries that production workflows on main later restore. The compromised packages carried valid SLSA Build Level 3 provenance attestations. Detection was external, within 20–26 minutes of publish. Prior precedent: researcher Adnan Khan's 2024 disclosure in `angular/dev-infra` demonstrated the same cache-stuffing→poisoned-node_modules chain, which Google classified a supply chain compromise.

**Framing correction applied to DECISIONS table:** The original DECISIONS row attributed the cache defense to "lockfile-hash cache keys." This framing was incorrect — an attacker's fork contains the same lockfile and computes the same key; lockfile-hash scoping does not prevent poisoning. The actual primary defenses are (a) the `pull_request_target` prohibition, which prevents the fork PR from reaching a privileged workflow in the first place, and (b) GitHub's branch-scoped cache isolation. The compensating control — running the allowlist gate after any cache restore — limits blast radius if a poisoned cache is somehow restored. The DECISIONS row has been corrected to reflect this accurately.

---

## R09 — OIDC tokens in GitHub Actions are short-lived and job-scoped

**Claim:** OIDC tokens are short-lived, scoped to a single workflow job, and expire when that job completes.

**Where it appears:** iterum.md (Infrastructure section, DECISIONS table)

**What was checked:** GitHub OIDC token lifetime and scope.

**Sources:**
- https://docs.github.com/en/actions/concepts/security/openid-connect
- https://github.blog/security/supply-chain-security/secure-deployments-openid-connect-github-actions-generally-available/

**Date verified:** June 2026

**Finding:** VERIFIED. GitHub documentation explicitly states: "With OIDC, your cloud provider issues a short-lived access token that is only valid for a single job, and then automatically expires." This language appears verbatim in both the GitHub Actions OIDC documentation and the GitHub Blog GA announcement. The current spec language — "OIDC tokens are short-lived, scoped to a single workflow job, and expire when that job completes" — matches the primary source exactly. The original undocumented "15–60 minutes" TTL claim was removed in a prior pass. Note: GitHub will update the default OIDC subject claim format for new repositories and repository renames on July 15, 2026 to include immutable owner and repo IDs. The spec's cosign verify command uses `--certificate-identity-regexp` which is unaffected by this change.

---

## R10 — Building container images once; rebuilding produces a different digest

**Claim:** Rebuilding the container image in the signing stage produces a different digest than the one Trivy scanned, defeating the integrity check. Build once, export as a tarball, pass the same artifact to scan and sign.

**Where it appears:** iterum.md (DECISIONS table), pipeline-os.md (Stage 7)

**What was checked:** Whether Docker image builds are reproducible by default (i.e., whether rebuilding produces the same digest).

**Sources:**
- https://github.com/moby/moby/issues/48391
- https://github.com/docker-library/official-images/issues/16044
- https://oneuptime.com/blog/post/2026-02-08-how-to-build-reproducible-docker-images-with-locked-dependencies/view

**Date verified:** June 2026

**Finding:** VERIFIED. Container builds are not reproducible by default. Timestamps, random build IDs, and layer ordering mean rebuilding generally produces a different digest. Sigstore's own cosign documentation reinforces signing by digest (`@sha256:...`) rather than tag for this reason. The build-once pattern is the correct architectural response.

---

## R11 — OpenSSF Scorecard is a supply chain hygiene audit available as a GitHub Action

**Claim:** OpenSSF Scorecard evaluates repository supply chain security practices and is available as `ossf/scorecard-action`.

**Where it appears:** iterum.md (pipeline stage 9, DECISIONS table), pipeline-os.md (Stage 9), README.md (References)

**What was checked:** OpenSSF Scorecard's capabilities and GitHub Actions availability.

**Sources:**
- https://scorecard.dev/
- https://github.com/ossf/scorecard-action
- https://github.blog/open-source/reducing-security-risk-oss-actions-opensff-scorecards-v4/

**Date verified:** June 2026

**Finding:** VERIFIED. Scorecard evaluates practices including Branch-Protection, Code-Review, Pinned-Dependencies, Signed-Releases, Dangerous-Workflow, and others, scoring each 0–10. From scorecard-action v2, publishing results requires `id-token: write` to obtain a GitHub OIDC token for authentication. This is why the scorecard job in pipeline-os.md requires `id-token: write` — confirmed correct.

---

## R12 — SPVS 1.5 V3.2 Deterministic Tool Authorization

**Claim:** SPVS 1.5 V3.2 establishes that model output alone cannot invoke privileged actions, requiring explicit allowlists, separate authorization gates, and full audit logging.

**Where it appears:** iterum.md (DECISIONS table, SPVS references), README.md (References)

**What was checked:** Whether SPVS 1.5 exists, whether V3.2 Deterministic Tool Authorization is a real control, and whether the language is accurate.

**Sources:**
- https://github.com/OWASP/www-project-spvs/blob/main/1.5/Release_Notes_OWASP_SPVS_1.5-AI-Pipeline-Security.md
- https://github.com/OWASP/www-project-spvs/blob/main/1.5/OWASP_SPVS_1.5-AI_-en_Requirements.csv
- https://owasp.org/www-project-spvs/

**Date verified:** June 2026

**Finding:** PARTIALLY VERIFIED with correction applied. SPVS 1.5 exists and V3.2 Deterministic Tool Authorization is a real control. The language used in this spec is a faithful paraphrase of the OWASP release notes — not verbatim control row text. More importantly: SPVS 1.5 is explicitly a non-finalized community feedback draft. The maintainers state "1.5 is not a finalized standard" and v2.0 targets October 2026. Spec updated throughout to note draft status and to present the control language as paraphrase.

---

## R13 — `step-security/harden-runner` provides egress restriction

**Claim:** `harden-runner` monitors and can block outbound network traffic from GitHub Actions runners via a domain allowlist.

**Where it appears:** iterum.md (DECISIONS table, SCA stage), pipeline-os.md (Stage 6, Stage 7, Stage 9)

**What was checked:** Whether harden-runner provides egress restriction and whether it has any documented limitations.

**Sources:**
- https://docs.stepsecurity.io/github-actions/harden-runner
- https://github.com/step-security/harden-runner
- https://github.com/step-security/harden-runner/security/advisories/GHSA-46g3-37rh-v698 (DNS-over-HTTPS bypass advisory)

**Date verified:** June 2026

**Finding:** VERIFIED with caveat. harden-runner provides egress monitoring and blocking via domain allowlist and maintains a SOC-managed Global Block List. However, the Community tier has documented bypass advisories including DNS-over-HTTPS (GHSA-46g3-37rh-v698), DNS-over-TCP, and a `disable-sudo` privilege escalation (CVE-2025-32955). Egress restriction via harden-runner is a meaningful control but is not absolute on the Community tier. The limitation is recorded here as part of the verified claim.

---

## R14 — Sigstore cosign for container image signing

**Claim:** cosign signs container images via keyless signing using Fulcio (certificates) and Rekor (transparency log), with signatures stored in the registry.

**Where it appears:** iterum.md (DECISIONS table), pipeline-os.md (Stage 10)

**What was checked:** cosign's signing mechanism and storage approach.

**Sources:**
- https://docs.sigstore.dev/cosign/signing/signing_with_containers/
- https://github.com/sigstore/cosign

**Date verified:** June 2026

**Finding:** VERIFIED. cosign supports both traditional key pairs and keyless signing. Signatures are stored in the container registry using the OCI 1.1 referrers spec. Sigstore documentation explicitly advises signing by digest (`@sha256:...`) rather than tag — directly reinforcing the build-once pattern in R10. The cosign verify command in pipeline-os.md Stage 14 uses `--certificate-identity-regexp` and `--certificate-oidc-issuer` which are correct flags for keyless verification.

---

## R15 — RFC 9457 `type: about:blank` as default error type URI

**Claim:** RFC 9457 specifies `about:blank` as the default value of the `type` member in problem details responses, indicating no additional semantics beyond the HTTP status code.

**Where it appears:** iterum.md (error system section, OWASP controls table, DECISIONS table)

**What was checked:** Whether RFC 9457 specifies `about:blank` as the default type URI and under what conditions it should be used.

**Source:** https://www.rfc-editor.org/rfc/rfc9457.html (Sections 3.1 and 4.2.1)

**Date verified:** June 2026

**Finding:** VERIFIED. RFC 9457 Section 3.1 states: "the 'about:blank' URI is the default value for that member. Consequently, any problem details object not carrying an explicit 'type' member implicitly uses this URI." Section 4.2.1 registers `about:blank` in the IANA HTTP Problem Types registry with the note that when used, `title` SHOULD match the HTTP status phrase. RFC 9457 obsoletes RFC 7807 and is backward compatible. The implementation in this spec — `type: about:blank` with explicit `title`, `status`, `code`, `boundary`, and `tier` fields — is correct usage.

---

---

## R16 — SPVS control ID citation accuracy and notation normalization

**Claim:** Multiple control IDs cited across the DECISIONS table and Known Gaps section in two notations: unversioned SPVS V-style (`V2.4.5`, `V1.5.1`, `V2.4.14`) from SPVS 1.0, and versioned 1.5-style (`SPVS 1.5 V3.2`, `SPVS 1.5 V2.2`). These should reference two distinct documents.

**What was checked:** Whether the unversioned V-style IDs correspond to real SPVS 1.0 controls, and whether the notation mismatch creates false cross-references.

**Sources:**
- https://github.com/OWASP/www-project-spvs/blob/main/1.0/ (SPVS 1.0 controls)
- https://github.com/OWASP/www-project-spvs/blob/main/1.5/OWASP_SPVS_1.5-AI_-en_Requirements.csv (SPVS 1.5 AI controls)

**Date verified:** June 2026

**Finding:** VERIFIED. The SPVS 1.0 and 1.5 control numbering schemes are distinct and do not overlap. SPVS 1.0 controls use a three-part numeric format (V1.3.1, V2.4.5, etc.) verified against the published SPVS 1.0 requirements CSV at `OWASP/www-project-spvs/1.0/`. SPVS 1.5 AI pipeline controls use the same numeric format but cover a distinct non-overlapping set of control IDs under new sub-categories (V1.3 NHI Governance, V3.2 Deterministic Tool Authorization, etc.) verified against the published 1.5 requirements CSV. The DECISIONS table now uses `SPVS 1.0 V...` and `SPVS 1.5 V...` prefixes throughout, removing the previously ambiguous unversioned form. No control ID cited in the spec was found to be incorrect after verification against both CSVs.

---

## R17 — Canon fetch via mutable branch reference

**Claim (gap):** The canon URL uses `main` as a ref — a mutable branch. The trust anchor is consumed via a floating reference with no integrity check beyond TLS and a placeholder grep. This contradicts the core thesis of pinning to immutable references.

**What was checked:** Whether any published supply chain tooling documents this as an accepted gap, and what mitigations are available.

**Sources:**
- https://docs.sigstore.dev/cosign/signing/signing_blobs/ (cosign sign-blob for file integrity)
- https://github.com/step-security/harden-runner (egress restriction as partial mitigation)
- https://slsa.dev/spec/v1.0/threats (SLSA threat model for build systems)

**Date verified:** June 2026

**Finding:** VERIFIED as a real gap. The SLSA threat model categorizes this as a "compromised dependency" attack — the build system trusts a mutable upstream reference and the upstream is not integrity-checked. CODEOWNERS + branch protection mitigate the PR path but not a compromised owner account or token pushing directly to main. Two mitigations were evaluated: (a) SHA-pinned fetch per project repo — maintainable but requires deliberate update on each canon rotation; (b) `cosign sign-blob` + verify at canon-fetch — the most robust approach, makes the integrity check cryptographically verifiable, but adds Sigstore as a hard dependency. Neither is implemented in v1.0. Documented as a known gap in iterum.md Known Gaps section with both alternatives and their tradeoffs. This is a planned control for a future version.

---

---

## R18 — Express 5 as current stable version

**Claim:** Express 5.2.1 is the current stable version on npm. Express 4 has entered maintenance status.

**Where it appears:** canon.json (`express: 5.2.1`), iterum.md approved dependencies table

**What was checked:** Current npm latest tag for express, Express 4 EOL status, Express 5 breaking changes relevant to the spec.

**Sources:**
- https://www.npmjs.com/package/express (latest: 5.2.1)
- https://expressjs.com/en/blog/2025-03-31-v5-1-latest-release/
- https://github.com/expressjs/express/releases

**Date verified:** July 2026

**Finding:** VERIFIED. Express 5.1.0 became the npm `latest` tag on March 31 2025. Express 5.2.1 is the current release. Express 4 entered formal Maintenance status April 1 2025, with a target EOL no sooner than October 1 2026. Express 5.2.0 contained an erroneous breaking change related to the extended query parser; 5.2.1 fully reverts it. CVE-2024-51999 associated with 5.2.0 was rejected by MITRE — there is no actual security vulnerability. Express 5 requires Node.js 18 or higher, which is satisfied by the node:24-alpine base image. Breaking changes relevant to the spec: `app.del()` removed (use `app.delete()`), `res.sendfile()` removed (use `res.sendFile()`). Neither affects the generated scaffold's core patterns.

---

## R19 — Zod 4 as current stable version

**Claim:** Zod 4.4.3 is the current stable version. Generated validators must target Zod 4 API.

**Where it appears:** canon.json (`zod: 4.4.3`), iterum.md Validation section

**What was checked:** Zod 4 release status, breaking changes from v3 that affect the spec's generated validator patterns.

**Sources:**
- https://www.npmjs.com/package/zod (latest: 4.4.3)
- https://zod.dev/v4 (release notes)
- https://zod.dev/v4/versioning (subpath versioning strategy)

**Date verified:** July 2026

**Finding:** VERIFIED with migration notes applied to spec. Zod 4 became stable in May 2025 and is now the default `npm install zod` target. Breaking changes that affect generated validator code: (1) `z.record()` now requires two schema arguments — `z.record(z.string(), z.string())`; the single-argument form is removed not deprecated; (2) `ZodError.errors` and `.formErrors` removed — use `.issues`; (3) Error construction uses single `error` param replacing `required_error`/`invalid_type_error`/`errorMap`; (4) Top-level format validators preferred (`z.email()`, `z.uuid()`) over chained methods (chained forms are deprecated not removed). Spec updated with Zod 4 migration note in the Validation section to prevent agents from generating Zod 3 patterns.

---

## Summary table

| ID | Claim | Finding | Correction applied |
|----|-------|---------|-------------------|
| R01 | CODEOWNERS does not block merges without branch protection | VERIFIED | None |
| R02 | Annotated vs lightweight tag SHA resolution | VERIFIED | Added lightweight-tag branch; applied to pipeline-os.md and iterum.md |
| R03 | `pull_request_target` secrets exposure for fork PRs | VERIFIED | None |
| R04 | OWASP Top 10 2025 category labels | VERIFIED | None |
| R05 | `npm audit --omit=dev --audit-level=high` | VERIFIED | None |
| R06 | `grype dir:` Go stdlib false positives from esbuild | VERIFIED | None |
| R07 | `trivy-action` vs `docker run` | VERIFIED | Upgraded: official Trivy CI/CD docs confirm trivy-action as the GitHub Actions integration |
| R08 | Cache poisoning via `pull_request_target` (TanStack) | VERIFIED | Corrected DECISIONS row framing — lockfile-hash is not the primary defense |
| R09 | OIDC tokens job-scoped, short-lived | VERIFIED | Upgraded: GitHub docs confirm "only valid for a single job, then automatically expires" |
| R10 | Rebuilding container produces different digest | VERIFIED | None |
| R11 | OpenSSF Scorecard as GitHub Action | VERIFIED | None |
| R12 | SPVS 1.5 V3.2 Deterministic Tool Authorization | PARTIALLY VERIFIED | Marked as draft; language as paraphrase |
| R13 | harden-runner egress restriction | VERIFIED | Added bypass caveat |
| R14 | cosign container image signing | VERIFIED | None |
| R15 | RFC 9457 `about:blank` default type URI | VERIFIED | None |
| R16 | SPVS control ID citation accuracy and notation | VERIFIED | Upgraded: both CSVs verified, no incorrect IDs found, notation normalized |
| R17 | Canon fetch via mutable branch reference (gap) | VERIFIED gap | Named in Known Gaps with rejected alternatives and planned mitigation |
| R18 | Express 5.2.1 as current stable version | VERIFIED | None — 5.2.1 is correct pin; Express 5 requires Node 18+ (satisfied by node:24) |
| R19 | Zod 4.4.3 as current stable version | VERIFIED | Zod 4 migration notes added to spec Validation section |

---

## Notes on claims from build experience

The following implementation findings include direct build experience on the Lunaris reference implementation and other projects. Where available, external sources provide independent support:

- **grype dir: false positives** (R06) — additionally observed in direct build runs; upstream issues confirm the same behavior independently
- **Trivy via trivy-action** (R07) — operational behavior on GitHub-hosted ubuntu-latest runners; the `docker run` approach produced failures not reproducible in local environments
- **frontend/.angular/ gitignore** — 46,000+ line figure from direct `git status` observation after omitting the gitignore entry
- **Bootstrap tfstate gitignore** — real incident; infrastructure credentials were committed via a bootstrap state file; keys rotated

These entries are marked "Implementation finding — observed in Iterum" in ITERUM-DECISIONS.md.

---

## Re-verification schedule

| Item | Trigger for re-verification |
|------|---------------------------|
| R12 SPVS 1.5 controls | When SPVS 2.0 ships (target October 2026) |
| R13 harden-runner bypass advisories | On each harden-runner release with security notes |
| R03 `pull_request_target` partial mitigation | GitHub may strengthen enforcement in future releases |
| R04 OWASP Top 10 2025 | Stable until next OWASP release cycle |

---

*the pipeline cannot catch decisions that were never made.*
*the architecture is the security.*
