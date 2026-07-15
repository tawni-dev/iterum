Copyright 2026 Tawni Glover / Aequalia LLC. Licensed under Apache-2.0.

# AUDIT.md — External Review History

**Last reviewed:** 2026-07-14
**Status:** Final pre-release review complete. Ready for public portfolio release with documented limitations.

This document records the external review passes conducted on Iterum before its first tagged release, the findings from each pass, what was fixed, and what the fix introduced. The process is part of the portfolio signal: audit, remediate, re-audit, catch what the remediation introduced.

All passes were conducted by an external AI reviewer (designated Fable) reading the full document set cold and producing structured findings. The reviewer had no prior context between passes except what was visible in the files. All factual claims in RESEARCH.md were verified by the author against primary sources — the AI reviewer identified claims to verify, the human author ran the verification.

---

## Pass 1 — Pre-publication structural audit

**Files reviewed:** iterum.md, pipeline-os.md, README.md, NOTICE, RESEARCH.md (initial versions)

**Critical findings:**

1. **pipeline-os.md was missing all stage requirement sections.** RESEARCH.md cited "Stage 4," "Stage 6," "Stage 7," "Stage 9," "Stage 10," "Stage 14," and "Prohibited triggers section" — none of which existed in pipeline-os.md. The document contained only the canonical-order string. Every citation was dangling. **Fixed:** Full stage requirement sections (Stage 0–14, Prohibited triggers, Tag protection) written into pipeline-os.md. RESEARCH.md citations now resolve.

2. **RESEARCH.md count was wrong.** Header stated "Eleven are VERIFIED. Four are PARTIALLY VERIFIED" but the summary table showed twelve VERIFIED and three PARTIALLY VERIFIED. **Fixed:** Count corrected.

3. **Canon fetch uses a mutable branch reference.** The trust anchor was consumed via `main` with no integrity check beyond TLS and a placeholder grep. CODEOWNERS and branch protection mitigate the PR path but not a compromised owner account. **Fixed:** Named as a known gap with two rejected alternatives (SHA-pinned fetch, cosign sign-blob) and accurate tradeoff documentation.

4. **Line 305 enforcement claim unimplemented.** "The pipeline fails if any workflow SHA or tool version does not match the fetched canon.json" was an assertion with no enforcing mechanism. **Fixed:** Replaced with an actual canon-verify script covering SHA verification, per-action binding, and tool version checks.

**Additional findings addressed:**

- Attribution inflation: 39 DECISIONS rows labeled "Aequalia original" including SHA pinning, OIDC, Scorecard, fetch-depth — all standard practice. **Fixed:** Replaced with neutral attribution tiers: "Iterum-specific design," "Standard practice — adopted," and "Implementation finding — observed in Iterum."
- R08 DECISIONS row attributed the primary cache-poisoning defense to lockfile-hash keys. **Fixed:** Correctly attributed to `pull_request_target` prohibition and branch-scoped isolation; allowlist-after-restore documented as compensating control.
- "Never on disk" absolute in DECISIONS contradicted the CI step writing `.pipeline-canon.json` to the runner workdir. **Fixed:** Absolute removed; scoped language retained.
- iterum.md stated "GitHub Actions releases use annotated tags" categorically. **Fixed:** Corrected to "may use annotated or lightweight tags" with resolution path for both.
- Project-repo branch protection not documented. **Fixed:** Added to CONTRIBUTING.md generation spec mirroring pipeline-repo instructions.

---

## Pass 2 — Post-remediation verification

**Files reviewed:** All files after Pass 1 fixes applied.

**Critical findings:**

1. **Stage sections still dangling in a new form.** The pipeline-os.md restructure (Codex merge) collapsed stage sections again. pipeline-os.md said "the requirements defined in the stage sections above" with no stage sections present. RESEARCH.md citations still pointed at non-existent locations. **Fixed:** Stage sections restored in full under "Pipeline runtime requirements reference" heading. Citations verified.

2. **R16 correction asserted but not applied.** RESEARCH.md R16 stated "Correction applied: the DECISIONS table now labels SPVS 1.0 citations as SPVS 1.0 V..." but the table still used unversioned form throughout. **Fixed:** Applied `SPVS 1.0 V...` prefix to all 1.0 citations in DECISIONS table. R16 finding now matches the spec.

3. **Branch protection deadlock for solo maintainer.** Three locations instructed enabling "Require a pull request before merging" + "Require review from Code Owners" + "Do not allow bypassing the above settings" — then told the solo maintainer to update canon.json via PR. GitHub does not allow a PR author to approve their own PR. The setup sequence broke itself. **Fixed:** "Do not allow bypassing" removed as a hard requirement. Accurate framing: owner retains admin direct-push with a warning; protection gates non-admin actors. Solo maintainer tradeoff documented explicitly.

4. **canon-verify script had three coverage holes:** floating tags passed silently (non-SHA refs skipped, not failed); tool versions unchecked; SHA membership checked globally not per-action. **Fixed:** Script hardened — fails on any non-SHA `uses:` line, per-action binding via `jq`, tool version strings checked against `canon.tools`.

**Additional findings addressed:**

- FILLING-CANON.md had two copies (standalone file and embedded template in pipeline-os.md) that had already diverged. Standalone dropped tool release-page URLs and 40-char hex verification step. **Fixed:** Standalone rewritten to match embedded template. Single-source-of-truth note added to pipeline-os.md.
- Four attribution rows still overclaimed: RFC 9457 (implementing an RFC is standard), Zero-Secret Baseline (no-.env pattern is standard), Facade as security boundary (facade is standard Angular), Per-job timeout (baseline hygiene). **Fixed:** All four split or relabeled. Iterum-specific elements (four-artifact secret rule, RFC 9457 extension fields plus opaque searchTerm, facade as security enforcement mechanism) are recorded without claims of demonstrable originality.
- Express 5 Node 18+ compatibility not documented in spec. **Fixed:** Added to approved dependencies table.
- Zod 4 DO NOT GENERATE entries missing. **Fixed:** Five Zod 4 patterns added.

---

## Pass 3 — Strategic and structural review

**Files reviewed:** All files after Pass 2 fixes applied. External research context provided (Gemini Deep Research on 2026 agentic DevSecOps landscape, adjacent academic literature survey).

**Structural findings:**

1. **Duplicate DECISIONS row.** "Per-job timeout-minutes required" appeared twice identically. **Fixed:** Duplicate removed.

2. **Stage 1 miscitation.** `(R06 — RESEARCH.md)` on the Gitleaks shallow-checkout note. R06 is the grype/esbuild entry. **Fixed:** Citation removed from Stage 1.

3. **Admin-bypass mechanics described backwards in three locations.** With "Do not allow bypassing" off, admins can direct-push — GitHub warns but permits. The spec described it as "direct pushes to main are blocked" which was incorrect and weaker than the actual threat model. **Fixed:** Accurate framing in all four locations: protection gates non-admin actors, owner retains admin bypass.

4. **Canon-verify script in two copies with already-diverged comments.** iterum.md and pipeline-os.md Stage 0 both carried the script. **Fixed:** Stage 0 designated as single source of truth; iterum.md references it rather than duplicating.

5. **Dotless artifact name stated as a platform rule without attribution.** "Artifact names must not contain dots" with no source. **Fixed:** Attributed as "discovered in build" inline and in DECISIONS table.

6. **14-stage vs Stage 0–14 count ambiguity.** Fifteen stage sections exist (Stage 0 through 14) but "14-stage" appears in README and pipeline-os.md without explanation. **Fixed:** Convention documented: "canon-fetch is Stage 0; stages 1–14 follow."

7. **Two design seams not documented as known gaps.** Stage 12 ZAP fails open when `DAST_TARGET_URL` unset; Stage 9 scorecard "not a hard gate" contradicts the "every stage runs only after all prior stages pass" claim. **Fixed:** Both added to Known Gaps with accurate framing and resolution path.

**Strategic findings (inform v1.0 positioning):**

- "Unoccupied construction layer" is too wide. Backstage, Cortex, Port, StepSecurity secure-repo, and the Wiz rules files the spec itself fetches are all construction-layer. The defensible claim is narrower: external human-sealed artifact authority gating both generation and CI runtime, combined with a pre-generation threat model gate.
- "Blue ocean" is false at the research layer. Four papers in twelve months converging on deterministic control planes is validation, not empty water. The defensible position is a practitioner-oriented implementation aligned with recent literature on deterministic control planes.
- README "What gets generated" led with Angular/Node/Azure, burying the core object. **Fixed:** Authority model leads; scaffold repositioned as the reference implementation demonstrating it.
- The audit trail itself (three external passes, findings documented, corrections tracked) is portfolio material that was invisible in the repo. **Fixed:** This AUDIT.md.

---

## Upgrades applied post-Pass 3

**RESEARCH.md claim upgrades (research-verified):**

- R07 (`trivy-action` vs `docker run`): PARTIALLY VERIFIED → VERIFIED. Trivy CI/CD docs confirm `trivy-action` as the official GitHub Actions integration.
- R09 (OIDC job-scoped): PARTIALLY VERIFIED → VERIFIED. GitHub docs confirm "only valid for a single job, then automatically expires."
- R16 (SPVS notation): PARTIALLY VERIFIED → VERIFIED. Both CSVs verified, no incorrect IDs found.

**Count:** 19 claims total. 18 VERIFIED. 1 PARTIALLY VERIFIED (R12 — SPVS 1.5 draft status is a factual condition, not a verification gap).

---

## Re-verification schedule

| Item | Trigger |
|------|---------|
| R12 SPVS 1.5 controls | When SPVS 2.0 ships (target October 2026) |
| R13 harden-runner bypass advisories | On each harden-runner release with security notes |
| R03 `pull_request_target` partial mitigation | GitHub may strengthen enforcement in future releases |
| canon.json values | Every 90 days or on any upstream release announcement |

---

*the pipeline cannot catch decisions that were never made.*
*the architecture is the security.*
