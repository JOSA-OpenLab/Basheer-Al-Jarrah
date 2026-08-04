>[!note] I did not source this file from any external projects. I authored the core rules myself, used Claude to refine the phrasing, and then thoroughly reviewed the final output.

This project is governed under a **BDFL (Benevolent Dictator For Life)** model.

## 1. Decision Authority

**LD-RW** is the BDFL and final decision-maker for this project. All technical, architectural, and process decisions ultimately rest with the BDFL. This includes, but is not limited to:

- Accepting or rejecting contributions
- Setting project direction, scope, and roadmap
- Resolving disputes when consensus cannot be reached
- Naming and revoking any maintainer or reviewer roles

The BDFL role exists to keep the project coherent and moving, not to shut out collaboration. Discussion, review, and community input are welcome and encouraged before a decision is made.

## 2. How Decisions Are Made

A contribution or change is evaluated against the following criteria, in order:

1. **Correctness** — If the contribution doesn't work, or fails any test in the suite, it is **refused**.
2. **Relevance to the current phase** — If the contribution works correctly but doesn't serve the project's current goals, it is **refused**, even if it might be valuable at some future phase. Good ideas that are early are logged for later (see [Section 5](#5-deferred-ideas)), not merged early.
3. **Simplicity** — If the change adds more complexity than it justifies (in code, in dependencies, in cognitive load, in maintenance burden), it is **refused**. Simpler and slightly less capable beats complex and marginally better.

If a contribution clears all three, it moves to standard review (tests, style, docs) before merge.

## 3. Contribution Process

- Open an issue before starting non-trivial work, so scope and phase-fit can be discussed early and effort isn't wasted.
- Follow [Conventional Commits](https://www.conventionalcommits.org/) for commit messages.
- All CI checks (tests, linting, security scanning) must pass before a PR is considered.
- PRs should be scoped to one logical change. Large, multi-purpose PRs will be asked to split.
- Use [Conventional Comments](https://conventionalcomments.org/) in reviews for clarity (`praise:`, `issue:`, `suggestion:`, `question:`, `nitpick:`).

## 4. Security

- Dependencies are kept pinned and monitored (e.g. Dependabot, SHA-pinned Actions where applicable).
- Releases are signed (e.g. keyless signing via cosign) so provenance can be verified.
- An SBOM is generated for releases where practical.
- Security-relevant issues should be reported privately rather than as public issues — see `SECURITY.md` if present, or contact the BDFL directly.

## 5. Deferred Ideas

Contributions refused solely for reason #2 (not useful *right now*) are not lost. They should be:

- Labeled `deferred` / `future-phase` on the issue tracker, with a short note on why now isn't the time.
- Revisited when the project phase changes, rather than requiring the contributor to re-argue from scratch.

## 6. Roles

- **BDFL** — final authority, as above.
- **Maintainers/Reviewers** (if/when appointed) — can review, triage, and recommend, but merge authority and final call remain with the BDFL unless explicitly delegated in writing (e.g. in this file).
- **Contributors** — anyone opening issues or PRs. No special status required to propose changes.

Roles are granted and revoked at the BDFL's discretion, based on sustained, trustworthy contribution — not tenure or seniority elsewhere.

## 7. Conflict Resolution

- Technical disagreements are worked out via discussion on the issue/PR first.
- If no consensus is reached, the BDFL makes the final call and states the reasoning briefly, so the decision is understood even by those who disagree with it.
- Disagreement is fine and welcome; re-litigating a settled decision without new information is not.

## 8. Transparency

- Rejections should come with a brief reason (fails tests / wrong phase / too complex / other), so contributors know whether to revise, wait, or drop it.
- Significant decisions (scope changes, breaking changes, deprecations) are recorded in the changelog or a `DECISIONS.md`/ADR log, not just left in closed PR threads.

## 9. Changing This Document

As BDFL, LD-RW may amend this document at any time. Changes to governance itself should still be recorded via a normal commit with a clear message, so the history of "how the rules changed" stays visible.
