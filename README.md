Copyright 2026 Tawni Glover / Aequalia LLC. Licensed under Apache-2.0.

# Iterum

A secure-by-default generation harness for AI coding agents.

Iterum gives agents a sealed authority model: a human-controlled, CODEOWNERS-protected registry of approved GitHub Action SHAs, tool versions, base image digests, and dependency versions. It gates both bootstrap generation and CI runtime on fetching from it before anything else runs.

This is not a rules file. It is not a template. It is a harness: the agent reads the spec, fetches the canon, asks the questions, produces the threat model, then generates the scaffold. Security invariants are sealed before any code is written. The agent implements within them.

**The authority model is the product.** The Angular/Node/Azure scaffold is a reference implementation demonstrating it. If your stack is different, the canon authority pattern, the pre-generation threat model gate, and the 14-stage pipeline requirements apply regardless. The scaffold is replaceable. The authority model is not.

---

## How it works: two repos, two phases

Iterum requires two GitHub repositories and two distinct phases. Do not skip ahead.

```
Phase 1: pipeline repo         Phase 2: project repo
─────────────────────           ────────────────────────
Agent reads pipeline-os.md      Agent reads iterum.md
Generates pipeline repo         Fetches filled canon.json
  └─ canon.json (template)        from pipeline repo
  └─ CODEOWNERS                 Asks PRODUCT.md questions
  └─ FILLING-CANON.md           Generates THREAT-MODEL.md
                                Generates project scaffold
Human fills canon.json          
  (outside any agent session)   
Commits + CODEOWNERS locks it   
```

**Phase 1 comes first. Always.** The pipeline repo must exist and `canon.json` must be filled before any project repo is generated. If you run Iterum without a filled canon, the agent stops at step 1 and will not proceed.

---

## Reference Authority

The public [`tawni-dev/iterum-pipeline`](https://github.com/tawni-dev/iterum-pipeline) repository is the Iterum **Reference Authority**. It is intended for learning, evaluation, demonstrations, and prototypes. Projects may consume it by using its public canon URL, but long-lived and production projects should create and maintain their own pipeline authority and `canon.json`.

The authority is intentionally replaceable: change the configured canon URL to point to the authority the project should trust.

Projects that continue using the Reference Authority are downstream consumers of its integrity. Compromise of the maintainer account or authority repository could affect consumers that have not migrated to their own authority. This is a documented trust-model risk, not a secret or vulnerability in the public contents of `canon.json`.

Publishing the self-referential Iterum demo and its source code does not expose other users' pipelines by itself. The demo may read and display the public canon, but it must contain no credentials, write access, private tokens, or administrative operations against the pipeline repository. It is a reference implementation of the pipeline visibility model, not a control plane for consumer repositories.

---

## The problem this solves

AI coding agents generate pipelines using whatever SHA or version string they find plausible. They hallucinate pinned versions. They use floating tags. They write `actions/checkout@v4` instead of a commit SHA. They install dependencies before any validation runs.

Rules files alone do not address the root issue: the agent has no external authority to check against, so it invents one.

Iterum gives the agent an authority it cannot modify under the documented non-admin trust model: a `canon.json` you maintain in a separate repository protected by CODEOWNERS. The agent fetches it before generating anything. CI fetches it before installing anything. If the fetch fails, everything fails. No fallback. No cached copy.

---

## Phase 1: set up the pipeline repo

### Step 1: generate it

Create a new empty GitHub repository. Name it `pipeline` or any name you choose (you will reference it by URL). Drop `pipeline-os.md` into the workspace and tell your agent:

> Read pipeline-os.md and follow the bootstrap flow.

The agent will ask whether you are using a GitHub organization or personal account, then generate:

- `canon.json`: all values as placeholders
- `CODEOWNERS`: locking the repo to your GitHub username
- `FILLING-CANON.md`: step-by-step instructions for filling every value
- `CONTRIBUTING.md` and `README.md`

### Step 2: fill canon.json yourself

Open `FILLING-CANON.md`. Follow it. Fill every value in `canon.json` by running the verification commands, not by asking an agent to suggest values.

Filling canon.json is the only step that requires manual supply chain verification outside an agent. It is intentional. The canon is the trust anchor. It is yours to maintain.

**Expect this to take 45–60 minutes.** You are resolving twelve action SHAs, six tool versions, two base image digests, and a backend dependency list — each verified individually against the GitHub API or official release pages. The time cost is the point: you are making a human decision about what is approved. An agent that fills canon.json for you has defeated the authority model.

Commit the filled `canon.json` directly to main before enabling branch protection — once protection is on, direct pushes by non-admin actors are blocked.

### Step 3: commit and lock

Replace `<OWNER>` in `.github/CODEOWNERS` with your GitHub username. Commit it to main.

Then enable branch protection:

1. `Settings > Branches > Add branch protection rule`
2. Branch name pattern: `main`
3. Enable: Require a pull request before merging
4. Enable: Require review from Code Owners
5. Save

**Solo maintainer note:** With branch protection enabled, direct pushes by non-admin actors are blocked. As the repository owner, you retain admin access and can push directly to main — GitHub shows a warning but permits it. What protection actually gates is every non-admin write actor: agents, collaborators, bot tokens, and compromised CI identities. That is the threat model. Your own canon updates proceed via direct push or PR; PRs from agents or contributors require your explicit Code Owner approval before merging. Organizations with a second trusted maintainer should also enable "Do not allow bypassing the above settings" to require PRs universally.

Your canon URL is:
```
https://raw.githubusercontent.com/<YOUR_ORG_OR_USER>/<PIPELINE_REPO_NAME>/main/canon.json
```

If the repo is **public** (recommended), no authentication is required to fetch it. `canon.json` contains no secrets, only SHAs and versions.

If the repo is **private**, create a fine-grained PAT scoped to read-only contents on this repo only. Store it as `PIPELINE_READ_TOKEN` in each project repo's secrets.

---

## Phase 2: generate a project repo

Create a new empty GitHub repository for your project. Drop `iterum.md` and `pipeline-os.md` into the workspace and tell your agent:

> Read iterum.md and follow the bootstrap flow.

The agent will:

1. Ask for your canon URL and fetch the filled `canon.json`
2. Confirm it has read the spec
3. Ask which AI coding tool you are using (Cursor / Claude Code / Codex)
4. Ask the six `PRODUCT.md` questions one at a time
5. Run the PII tripwire analysis
6. Generate `THREAT-MODEL.md` from the STRIDE model applied to your use case
7. Generate the merged agent rules file and run the deduplication pass
8. Generate the full repository scaffold using only values from canon.json
9. Open the first GitHub issue: post-scaffold multi-agent threat model

If the agent skips any step, stop it. The steps are ordered for a reason.

---

## What gets generated

**Authority and pipeline controls (stack-agnostic):**
- External human-sealed canon.json authority — SHAs, versions, base image digests, dependency versions — fetched before generation and before CI installs anything
- canon-verify runtime step — fails on unpinned refs, per-action SHA binding, tool version checks
- STRIDE threat model generated before any code is written, gating generation on completion
- PII tripwire — hard stop, not a warning, if user data surfaces in PRODUCT.md answers
- 14-stage GitHub Actions pipeline gated on the fetched canon.json (canon-fetch is Stage 0; stages 1–14 follow)
- SPVS 1.5 self-assessment, OWASP controls mapping, NIST SSDF alignment
- ITERUM-DECISIONS.md — every architectural and security decision with attribution, standard reference, and reasoning

**Reference application (Angular/Node/Azure — adapt to your stack):**
- Angular frontend with enforced facade pattern
- Node.js / TypeScript backend with validation boundary and RFC 9457 error system
- Opaque trace UUID: each error generates a random UUID written to your observability platform only; the public response contains no trace identifier and cannot be correlated to the internal log entry
- PostgreSQL reference data layer
- Cloud infrastructure via Terraform (Azure reference implementation; adapt to your provider)

---

## Pipeline overview

```
canon-fetch → secrets → lint → test → dependency-allowlist → SAST → SCA → container+SBOM → IaC → scorecard → sign → deploy-staging → DAST → manual-gate → deploy-production
```

Each stage runs only after all prior jobs complete. Scorecard findings are non-blocking, and DAST passes with a warning until `DAST_TARGET_URL` is configured. No deployment proceeds unless the remaining required gates pass. `pipeline-os.md` contains the full runtime requirements for each stage; the generated pipeline.yml is built from those requirements.

---

## Updating canon.json

When a tool version changes or a base image digest needs rotation:

1. Open a PR against the pipeline repo, or push directly as the repository owner
2. Fill in the new value with the verification commands from `FILLING-CANON.md`
3. CODEOWNERS requires your explicit approval before it merges
4. All project repos pick up the new value on their next pipeline run

Agents can open PRs against the pipeline repo. Under the documented non-admin trust model, they cannot merge them.

---

## What this is not

- A security certification or compliance guarantee
- A substitute for a threat model review by a qualified engineer
- A complete application: it generates a secure baseline, not a finished product
- A replacement for your own judgment on your specific use case

The spec encodes the architectural decisions made and the reasoning behind them. The STRIDE threat model step and the post-scaffold issue exist precisely because generated output must be reviewed.

---

## Contributing

See `CONTRIBUTING.md`. PRs welcome for:

- Closing documented gaps
- Additional cloud provider reference implementations (AWS, GCP)
- Additional agent tool support

Changes to the architecture or security invariants require opening an issue first with the decision reasoning before the change is accepted.

The external review history — three passes, all findings, all fixes, what the fixes introduced — is in `AUDIT.md`.

---

## Attribution

Apache 2.0. The `NOTICE` file credits the origin and must be carried in any distribution or derivative work.

If you build on Iterum, carry the NOTICE. If you write about it, link here.

---

## References

Standards, frameworks, and tools this project implements or aligns with:

**Security standards**
- [OWASP SPVS 1.5](https://github.com/OWASP/www-project-spvs) — Software Pipeline Verification Standard, community feedback draft (not a finalized standard; v2.0 targets October 2026). Iterum maps to V1–V5 AI pipeline controls.
- [OWASP Top 10 2025](https://owasp.org/Top10/2025/) — A02, A03, A09, A10 addressed in the generated baseline.
- [OWASP API Security Top 10 2023](https://owasp.org/API-Security/editions/2023/en/0x00-header/) — API4, API8 addressed.
- [NIST SP 800-218 SSDF](https://csrc.nist.gov/Projects/ssdf) — Secure Software Development Framework alignment documented in generated ITERUM-DECISIONS.md.
- [NIST SP 1800-44 NCCoE DevSecOps](https://pages.nist.gov/nccoe-devsecops/) — DevSecOps practices reference implementation.
- [RFC 9457 Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457) — Error system standard implemented in generated backend.

**Supply chain and pipeline tools**
- [OpenSSF Scorecard](https://github.com/ossf/scorecard) — Supply chain hygiene audit, included as pipeline stage 9.
- [step-security/harden-runner](https://github.com/step-security/harden-runner) — Egress restriction on all pipeline jobs.
- [Wiz secure rules files](https://github.com/wiz-sec-public/secure-rules-files) — Fetched at bootstrap and merged with Iterum application layer.
- [Sigstore cosign](https://github.com/sigstore/cosign) — Container image signing before production deployment.
- [anchore/grype](https://github.com/anchore/grype) — SCA lockfile scanning.
- [anchore/syft](https://github.com/anchore/syft) — SBOM generation per build.
- [semgrep](https://github.com/semgrep/semgrep) — SAST.
- [gitleaks](https://github.com/gitleaks/gitleaks) — Secrets detection.
- [checkov](https://github.com/bridgecrewio/checkov) — IaC misconfiguration scanning.
- [OWASP ZAP](https://github.com/zaproxy/zaproxy) — DAST after staging deploy.
- [aquasecurity/trivy](https://github.com/aquasecurity/trivy) — Container scanning.

**Research and threat intelligence informing design decisions**
- [StepSecurity: Secure GitHub Actions](https://www.stepsecurity.io/) — Egress restriction and harden-runner patterns.
- [OpenSSF: Preventing supply chain attacks](https://openssf.org/blog/2023/07/19/understanding-the-open-source-supply-chain/) — SHA pinning rationale.
- [GitHub docs: CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) — CODEOWNERS behavior and branch protection requirements verified against official documentation.
- [GitHub docs: Branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — CODEOWNERS requires branch protection to enforce merges; both required.
- [GitHub docs: OIDC in Actions](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect) — OIDC preferred over stored credentials for cloud auth.
- [OWASP CI/CD Top 10](https://owasp.org/www-project-top-10-ci-cd-security-risks/) — `pull_request_target` prohibition and pipeline attack surface.

---

## License

Apache-2.0. See `LICENSE` and `NOTICE`.

*the pipeline cannot catch decisions that were never made.*  
*the architecture is the security.*

— Tawni Glover / Aequalia LLC
