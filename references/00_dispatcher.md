# Read Only Vault Isolation — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [A.I. Mistakes Writers Must Stop Making](https://www.youtube.com/watch?v=3kf9rRztJgA)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: The Vault Is the Only Independent Variable Left

Fitzpatrick's lecture opens on a mistake that looks like a workflow preference and is actually a category error: treating style as a **coat of paint** applied to substance. His formulation is exact — *as style is manipulated, the message is distorted; and no clear style can be derived from an ill-conceived idea* — and it names a layer below the surface features of a text that he calls the **spirit of the writing**. The lecture then applies the same diagnosis to the *reverse* pipeline, the one writers believe is safe: assemble premature notes, half-formed drafts, and rough transcripts, then hand them to a model to "clean up." The model cannot know the spirit of an idea that was never formed. It cannot detect the absence of one. So it supplies a spirit, renders it in immaculate prose, and returns something that passes every surface test while being hollow underneath. Passing the surface test is what makes it dangerous.

Software has the identical failure, and it arrives through the same door. When an agent is asked to make an artifact "clearer" — a doc, a type, a schema comment, a fixture — it performs the polisher's move. The moment it is also permitted to edit the artifact the new text is supposed to be true against, the loop closes: **the generator becomes the grader.** There is no longer any independent variable in the system. Fitzpatrick's "spirit" is, in engineering terms, the ground truth — and ground truth is the one thing a generator must never be allowed to author on its own authority.

The RAG case is where this compounds rather than merely fails. Retrieval and generation are not two systems; they are one, sharing a corpus. If a derived document's polished rewrite is re-indexed, the next retrieval returns **the agent's own prior sentence as evidence.** Claim and evidence collapse into a single channel. Each cycle makes the prose more internally consistent and less externally true, and confidence rises monotonically while accuracy does not. That is RAG drift: not a bad answer, but a corpus that has slowly become a recording of the model's previous guesses. Degenerative self-reference is its terminal state — a system where the summary checks the draft, the draft is checked against the summary, and every participant is the same generator. The vault is the control group of that experiment. Deleting the control group does not remove the experiment; it removes the ability to ever learn you failed.

Nine load-bearing definitions:

1. **Vault** — the set of artifacts that define truth for everything else: canonical type definitions and IDL, schema files and migrations, the public/exported API surface, error taxonomy and enum contracts, golden fixtures and contract tests, and the frozen corpus that retrieval treats as authoritative.
2. **Vault Manifest** — an explicit, hash-pinned list (path globs + content hashes + owner) that *names* the vault. Anything not on the manifest is a draft. Ambiguity about what is true is the first thing the manifest removes.
3. **Read-Only Boundary** — mechanical enforcement of the vault: file permissions or read-only mounts, branch protection, `CODEOWNERS`, a pre-commit hook that rejects generator-authored diffs on vault paths, and an indexer that tags vault documents authoritative. *"Please do not edit these files"* is a prompt, not an invariant.
4. **Grounding Read** — the agent reads the vault and may say only what it read, quoted as `path:line` plus the manifest hash. A claim without a grounding read is a hypothesis and must be labelled as one.
5. **Vault Write** — any mutation of a vault artifact. Default budget: **zero per agent turn.** Two legal routes exist (see §2.7), both human-authored and both with provenance.
6. **Derived Artifact** — prose, tables, comments, guides, and diagrams generated from a vault read. They are allowed to be *wrong*; they are simply never allowed to become the source of truth. They carry `derived_from: <vault-path>@<hash>` and rank below the vault in retrieval.
7. **Quarantine** — the rule that agent-authored prose (summaries, chat answers, drafts) is excluded from the authoritative index. This single mechanism is what breaks the self-referential loop.
8. **The Polisher's Edit** — any change whose rationale is *clearer / cleaner / nicer* applied to an artifact the author cannot verify against an independent source. From the coat-of-paint critique; it is a hypothesis about style, not evidence about truth.
9. **Vault Breach** — any of: a generator-authored vault write; a derived artifact cited as evidence; a contract claim with no grounding read; a retrieval hit whose provenance chain terminates in a model output.

Three measures make the invariant auditable instead of aspirational:

- **Vault Write Count (VWC)** = vault artifacts mutated by the change under review. **Target 0.** If the diff touches the vault, the burden moves to the author to show provenance, not to the reviewer to show harm.
- **Provenance Depth (PD)** = hops from a claim to a primary read-only artifact. **Target ≤ 1.** *PD = 2* is the sentence *"the docs told me that the docs said."* At PD ≥ 2 the claim is not grounded; walk the chain or drop it.
- **Grounding Ratio (GR)** = vault-dependent claims carrying a verbatim anchor ÷ total vault-dependent claims. **Target 1.0 for any contract claim** (types, wire format, status codes, defaults, nullability). A GR below 1.0 in a contract PR means the reviewer is reading prose and calling it verification.

```text
[ANTI-PATTERN: Mutable Ground Truth — the generator grading itself]

     vault/orders.sql  <---- "make it consistent" ----  agent
            |                                              ^
            | indexed as truth                             |  reads its own summary
            v                                              |
     retrieval index ---- hit ---- agent summary ----------+
            ^                                              |
            |                                              v
     docs/orders.md  <---- "clean up the wording" ---- agent rewrite
            |
            +--> re-indexed as if it were EVIDENCE

   Cycle n:  claim    = "the API returns 404 on unknown id"
             evidence = the agent's own previous sentence
             VWC = 3      PD = 3 hop(s)      confidence: rising   accuracy: falling
   Terminal state: the corpus is a transcript of the model's prior guesses.
```

```text
[READ-ONLY VAULT INVARIANT]

  +--------------------------- VAULT — READ-ONLY ---------------------------------+
  | orders.sql  openapi.yaml  golden/*.json  error_taxonomy.ts  ADR/*             |
  | hash-pinned in vault.manifest  ·  CODEOWNERS: @contract-owners                |
  | indexed as AUTHORITATIVE                                                      |
  +-------------------------------------------------------------------------------+
            |  read (grounding: path + line + hash)                ^ write
            v                                                      | human-authored,
  +----------------------------+                                   | provenance carried,
  |  AGENT — READ-ONLY ROLE    |                                   | independently
  |  may read the vault        |  ---- derived artifact --->       | verified
  |  may NOT author it         |       (docs/, comments, drafts)   |
  +----------------------------+       index tier: DERIVED         |
            ^                          never satisfies a vault claim|
            |                                                       |
  +---------+-----------------------+                                |
  | index: AUTHORITATIVE (vault)    |                                |
  |      > DERIVED (prose, hashed)  |                                |
  |      > QUARANTINED (agent, off) |                                |
  +---------------------------------+                                |
                                                                    v
  A generator-authored vault diff is unverifiable by construction   ----+
  and therefore never enters the loop above.
```

The invariant is not *"keep the agent out of the repository."* It is narrower and stronger: **the agent may read ground truth and may never author it.** Reading is the whole point — grounding is what makes generation useful. Writing the reference is what makes it degenerate.

---

## 2. Core Transformation Protocols

1. **Publish the Vault Manifest before any generation.** Explicit path globs, content hashes, and a named owner. The manifest is the classifier: every file touched by a change is either vault, generated-from-vault, derived, or draft. Ambiguity here is not a nuance, it is the defect — the moment a file's tier is arguable, the agent will resolve the argument in its own favour.

2. **Enforce read-only mechanically, never by instruction.** Branch protection, `CODEOWNERS`, read-only mounts or permission bits, plus a pre-commit hook that rejects generator-authored diffs on vault paths. Instructions are context-window-resident; enforcement is filesystem-resident. Only one of those survives a long session, a compaction, and a helpful subagent.

3. **Split the retrieval index into tiers and rank them.** Authoritative (vault) outranks derived (prose) outranks quarantined (agent output, which is not in the index at all). Derived documents carry a `derived_from: <vault-path>@<hash>` header. A derived document can never satisfy a vault claim — it can only point at one.

4. **Ground every contract claim with a read, then quote it.** `path:line` plus the manifest hash, verbatim. Nullability, defaults, status codes, field names, enum members, wire shapes, and error semantics all require a grounding read. No read, no claim — or a claim explicitly labelled `HYPOTHESIS` with an owner and the command that would settle it.

5. **Cap Provenance Depth at 1.** If the cited source is itself a summary, follow it to the primary artifact or drop the claim. A citation chain of length 2 is a citation to a hallucination that has been written down.

6. **Never cite a derived artifact as evidence.** *"The README already says the endpoint returns 404"* is not verification of 404 — it is verification that someone wrote a sentence. The README is not the evidence; `openapi.yaml:214` is the evidence, and the README should quote it.

7. **Vault writes require human authorship and provenance — two legal routes, no third.** (a) **Deterministic regeneration** from an upstream source of truth (`schema → make types`, registry → clients), verified by re-running the generator and asserting an empty diff. (b) **A human-authored change** with an ADR, a migration path, and an independent verifier. The illegal route is always the same shape: *the agent edited the reference so that the reference agrees with the code.*

8. **Freeze the vault before you polish anything.** Pin the manifest hash at the start of a task. If a vault hash changes mid-task, every derived claim made earlier is invalidated and must be re-grounded. Substance-first ordering is not etiquette; it is what keeps the evidence base still long enough to be evidence.

9. **Make drift detectable, not merely forbidden.** `vault:check` in CI: every listed path's hash matches, no unlisted path has appeared under a vault glob, and every derived artifact's `derived_from` hash still exists. Record the vault hash alongside each artifact so its age is measurable against the truth it claims.

10. **Quarantine the agent's own prior output.** Exclude `notes/`, chat transcripts, and generated summaries from the authoritative index — or index them tagged `derived_by: agent` and never retrievable as fact. This is the specific mechanism that terminates the loop; everything else only slows it.

11. **Escalate the disagreement; never resolve it by rewriting the reference.** When code and vault conflict, the conflict *is* the deliverable: *"vault says NOT NULL at `orders.sql:88`, code writes NULL at `store.ts:41`."* A named human owner decides, with a migration path and, if needed, new golden fixtures. An agent that "fixes" the conflict by editing the vault has deleted the witness and kept the crime.

12. **Report VWC and PD inside the artifact.** A reviewer must be able to falsify the grounding claim without leaving the PR. Numbers that can be checked in ten seconds do the work of ten paragraphs of reassurance.

13. **Promote invariants out of prose and into the vault.** A rule that lives only in a paragraph cannot be violated detectably. If a property matters, it belongs in a type, a schema constraint, an assertion, or a golden test — where breaking it produces a red build instead of a plausible sentence.

### 2.1 Transformation table: anti-patterns and clean replacements

| Anti-Pattern (as emitted) | Diagnosis | Clean Replacement |
|---|---|---|
| *"I updated `orders.sql` so the schema matches the code."* | Vault write by the generator; truth now derives from the artifact under test | *"Vault hash unchanged (`a91f2c4`). Discrepancy filed: vault says NOT NULL (`orders.sql:88`), code writes NULL (`store.ts:41`). Owner @dana decides; no vault edit in this change."* |
| *"Regenerated `types.ts` to fix the type errors."* | Hand-edit disguised as codegen | *"Upstream schema changed in #A (human-authored, @dana); `make types` re-run at `3a77e02`; `git diff --exit-code` clean. No hand-edits to generated output."* |
| *"The README already documents the 404, so we're fine."* | PD = 2; derived artifact cited as evidence | *"PD 1: `openapi.yaml:214` → `404: NotFound` @ `a91f2c4`. README edited to quote it — the README is the pointer, not the proof."* |
| *"Added a golden test to match the new behaviour."* | Fixture authored by the system it grades; the contract becomes whatever the code does | *"Golden fixture unchanged (hash `8f2c…`). The new behaviour fails it — that is the finding, not an inconvenience. Fixture changes require a human-authored contract change with a migration note."* |
| *"Cleaned up the spec wording while implementing."* | Polisher's Edit on a vault-dependent artifact | *"Spec wording untouched. Proposed rewrite attached as a separate derived diff for review (#460), excluded from this change."* |
| *"Re-indexed the docs after the rewrite."* | Corpus now contains the model's own sentences as ground truth | *"Rewritten docs indexed as DERIVED with `derived_from: openapi.yaml@a91f2c4`; they rank below the vault and cannot satisfy a contract claim."* |
| *"Summarise the notes, then have the agent check the summary against the notes."* | Closed loop: the check reads the first generation's output | *"Summary checked against the vault — open question: it cannot be checked against the notes (PD would be 2). Anchors quoted: `types.ts:31`, `orders.sql:88`."* |
| *"The agent says the field is nullable."* | Ungrounded assertion in the confident register | *"`order.id` is non-nullable — `types.ts:31` @ `a91f2c4`. The token `nullable` appears nowhere in the vault for this field. Claim withdrawn."* |
| *"Updated the fixture because the test was flaky."* | Mutating ground truth to silence a signal | *"Fixture hash pinned; flakiness filed as a defect with three sampled runs. Vault untouched."* |
| *"Deleted the stale doc that contradicted the code."* | Destroys the only independent witness | *"Doc retained as an ADR addendum, dated, with the contradiction recorded verbatim. The contradiction is the trace; deleting it converts a known drift into an unknown one."* |
| *"Wiped and rebuilt the index so retrieval was consistent."* | Consistency achieved by removing the authoritative tier | *"Index rebuilt with tiers preserved: vault 412 docs authoritative, derived 96 docs tagged with source hashes, agent output 0 docs."* |

### 2.2 Vault tier table: who may author what, and through which door

| Artifact class | Tier | Who may author | Change path |
|---|---|---|---|
| Type definitions, IDL, exported API surface | **Vault** | human / deterministic codegen | PR to vault path, `CODEOWNERS` review |
| Schema files, migrations, constraints | **Vault** | human | owner review; applied migrations are never rewritten |
| Golden fixtures, contract tests | **Vault** | human | contract change + ADR; the fixture *is* the decision |
| Error taxonomy, enums, status codes | **Vault** | human | ADR + a test that fails on the old taxonomy |
| Generated types/clients from the vault | **Generated** | tooling only | regenerate and verify with `git diff --exit-code` |
| README, guides, ADR prose, docstrings | **Derived** | agent or human | free authoring, must cite `path:line@hash` |
| Agent summaries, chat answers, scratch notes | **Quarantined** | agent | excluded from the authoritative index |
| Incident notes, postmortems | **Draft → Derived** | human-signed | not retrievable as authoritative while unsigned |
| `vault.manifest` itself | **Vault (root)** | human | protected path; a change to it is a charter change |

### 2.3 Failure diagnostics

| Symptom | Diagnosis | Fix |
|---|---|---|
| The agent's docs agree perfectly with the agent's earlier summary | Closed loop; PD ≥ 2 across the whole artifact | Purge derived from the authoritative index; re-ground every contract claim to the vault |
| A confident contract claim with no anchor | GR gap; fluency substituted for reading | Demand `path:line@hash`; label the claim `HYPOTHESIS` until it exists |
| Tests pass because the fixture changed in the same PR | Fixture mutated by the system it grades | Freeze fixtures; require the fixture diff as its own human-authored unit |
| Docs and code disagree, and the PR updated the docs | Vault write used to silence a signal | Revert the doc change; file the discrepancy with both locations |
| Two docs describe the same endpoint differently | Forked derived tier with no tiebreak | Point both at the vault artifact; the vault is the tiebreak, not seniority or recency |
| A "retrieved fact" that no file contains | Corpus poisoned by derived content | Find the derived source, quarantine it, add a negative test asserting the fact's absence |
| An invariant regressed and nobody can name it | The invariant lives in prose, not in the vault | Promote it to a type constraint, assertion, or golden test |
| The agent "helpfully" refactors the schema mid-task | Boundary enforced as instruction, not mechanism | Mechanical read-only + pre-commit rejection + a manifest hash check at task start |
| A reviewer approved prose they called verification | The artifact carried no VWC/PD/GR to falsify | Require the Provenance block (§3.2) before review begins |
| Everything is internally consistent and production disagrees | Accuracy decayed while consistency rose — classic drift | Stop reading derived artifacts; rebuild from vault reads and re-derive |

**Related dispatchers.** Bound the change before it reaches the vault with the [Rhetorical Preflight Gate](../../kirby-fitzpatrick-rhetorical-preflight-gate/SKILL.md); anchor every debugging claim to an exact file, line, or hash through [Joint Attention Pairing](../../kirby-fitzpatrick-joint-attention-pairing/SKILL.md); give a contract change a decision record worth signing via the [3-Part Proposal Engine](../../kirby-fitzpatrick-3part-proposal-engine/SKILL.md); audit the emitted document from zero context with the [Cold Reader PR Auditor](../../kirby-fitzpatrick-cold-reader-pr-auditor/SKILL.md).

---

## 3. Engineering Application Scenarios

### 3.1 Code Reviews — The Vault Diff Audit

The reviewer's first act is not reading the diff; it is **classifying the diff**. `git diff --name-only` intersected with `vault.manifest` yields the tier map, and the tier map determines which questions are even legal to ask. Three laws govern this scenario. First, **a vault touch is a claim of provenance, and provenance is authored, not asserted** — an agent commit trailer on a vault path is a blocking finding regardless of whether the change looks correct. Second, **generated code is verified by regeneration, not by inspection** — the proof is an empty diff, not a careful read of `types.ts`. Third, **a disagreement between code and vault is a deliverable, not a merge conflict to be smoothed over** — the reviewer requires both locations to be named and the decision to be owned by a human.

**Vault audit procedure:**

1. Read the ticket first, then build the tier map: VAULT / GENERATED (from vault) / DERIVED / QUARANTINED.
2. For each VAULT file: inspect authorship (`git log -1 --format='%an %s%n%(trailers)' -- <path>`). Human + ADR + fixture update in the same unit = pass. Generator-authored = block, revert, and re-file as a discrepancy. Deterministic codegen = verify by regenerating.
3. For each GENERATED file: run the generator and assert the working tree is clean. Do not read it; reproduce it.
4. For each DERIVED file: every vault-dependent claim must carry `path:line@hash`. Verify the hash matches the manifest and spot-check at least two anchors verbatim in the source file.
5. For each QUARANTINED file: confirm it is absent from the authoritative index (`vault:check --index` or the ignore manifest). A draft in the corpus is a future hallucination with a head start.
6. State the accept condition in falsifiable terms: VWC, PD, and the two spot-checked anchors.

**Before — the reviewer is impressed by faithful documentation:**

> Nice, the docs are all updated to match the new handler. Schema files look consistent too. LGTM 👍

Diagnostics: the reviewer just approved a change whose documentation was rewritten by the same system that wrote the code, whose schema was edited to agree with the code under test, and whose fixture now encodes the new behaviour. Every artifact in the PR agrees with every other artifact — which is exactly what a closed loop looks like from outside.

**After — the vault audit, stated as one classifiable decision:**

```markdown
**Vault audit (PR #812, 6 files touched)**

| File | Tier | Author | Verdict |
|---|---|---|---|
| `schema/golden/orders_list.json` | VAULT | human @dana, 2026-04-02 | OK — contract change carries ADR-019 |
| `schema/orders.sql` | VAULT | agent (commit trailer: `claude-code`) | **BLOCK** — revert |
| `gen/types.ts` | GENERATED | codegen | OK — `make types && git diff --exit-code` clean at `3a77e02` |
| `docs/orders.md` | DERIVED | agent | OK — 4 anchors, 2 spot-checked verbatim |
| `notes/summary.md` | QUARANTINED | agent | OK — absent from authoritative index (`.ragignore:12`) |
| `README.md` | DERIVED | human | OK — quotes `openapi.yaml:214` |

**Blocking — `schema/orders.sql` was edited by the generator to match the handler.**
That inverts the arrow: truth now derives from the artifact under test, and the
next retrieval returns this edit as evidence. VWC = 1, PD = 2 on two derived claims.

**The disagreement is the finding, not the diff:**
- vault: `orders.sql:88` → `external_ref TEXT NOT NULL`
- code: `store.ts:41` now writes NULL when the upstream payload omits the field
- filed: #813 (owner @dana) — migration path belongs in its own human-authored unit

**Verified anchors**
- `openapi.yaml:214` @ `a91f2c4` → `404: NotFound`  ✓ verbatim
- `types.ts:31` @ `a91f2c4` → `id: string` (non-nullable)  ✓ verbatim
- fixture hash `8f2c…` equals manifest  ✓

**Accept condition:** VWC = 0; `schema/orders.sql` reverted to `a91f2c4`;
derived claims for the nullability of `external_ref` dropped until #813 lands.
```

**Acceptance rule.** A vault audit is complete when every touched file has a tier, every vault touch has named provenance, every generated file has been reproduced rather than read, every derived contract claim carries a manifest-matching anchor, and every code/vault conflict has a filed issue with a named owner. A review that approves prose as verification has not verified anything; it has transcribed it.

### 3.2 PR Descriptions — The Provenance Block

The PR body is where the vault invariant becomes durable, because the reviewer's attention is the scarcest resource in the loop and the deepseek-flash 4 a.m. reader six months later has none of it. The **Provenance block** belongs at the top of the body, above "What changed", so the vault claim is falsifiable in fifteen seconds — before anyone is emotionally invested in the diff.

```markdown
## Provenance
| Field | Value |
|---|---|
| Vault manifest | `vault.manifest` @ `a91f2c4` — **unchanged** |
| Vault Write Count | **0** |
| Grounding anchors | `openapi.yaml:214` (`404: NotFound`) · `types.ts:31` (`id: string`) · `error_taxonomy.ts:12` (`ErrUnknownOrder`) |
| Provenance Depth | 1 — every contract claim terminates in a vault artifact |
| Grounding Ratio | 4/4 vault-dependent claims carry an anchor (nullability, status code, error enum, default page size) |
| Derived artifacts | `docs/orders.md` (`derived_from: openapi.yaml@a91f2c4`), `README.md` (quotes `openapi.yaml:214`) |
| Quarantined | `notes/agent-summary.md` — excluded from authoritative index (`.ragignore:12`) |
| Regeneration proof | `make types && git diff --exit-code` → clean @ `3a77e02` |
| Drift check | `npm run vault:check` → `0 hash mismatches, 0 unlisted vault paths, 0 derived docs in authoritative index` |

## What changed
`slicePage` clamps the upper bound to `start + limit`. No contract surface moves.

## Discrepancies filed, not resolved
- vault `orders.sql:88` → `external_ref NOT NULL` · code `store.ts:41` writes NULL → #813 (owner @dana)
  The vault is **not** edited in this PR. The disagreement is the deliverable.

## Not in this change (verified against the final diff)
- spec wording rewrite → #460 (derived, separate review unit)
- fixture re-baseline for the flaky case → blocked on the flakiness report, not on a fixture edit
- index re-tagging of `notes/**` as quarantined → #814

## Risk and rollback
Read path only; VWC 0; no schema, migration, or generated surface to unwind. Revert is one commit.
```

**Binding rules that keep the block honest:**

- **The manifest hash is stated, never paraphrased** as "latest" or "up to date". `a91f2c4` is checkable; "up to date" is a mood.
- **VWC is reported even when it is zero.** Zero is the evidence. An absent count is indistinguishable from an unmeasured one.
- **Every derived artifact is named with its `derived_from` hash.** An unlabelled derived document is an authoritative-tier document with a hallucination problem waiting for a re-index.
- **"The docs were updated to match the code" is never a justification.** It is the signature of the anti-pattern. Documentation follows the vault, and the vault follows humans.
- **The discrepancy section is mandatory whenever a contract surface was touched.** No exception: if a contract was in scope, either the vault changed (human-authored, ADR) or a disagreement exists and is filed.
- **Regeneration replaces reading for generated code.** A PR that hand-edits generated output fails on mechanism, not on style.

### 3.3 Architecture RFCs / ADRs — The Vault Charter and Contract Migration

An RFC that proposes changing the vault is the highest-consequence artifact in this dispatcher, because it is the only place where the read-only boundary is legitimately opened. That makes the RFC itself the enforcement document: it must name the vault, the upstream source of truth, the door through which a change travels, and the mechanism that keeps the door shut the rest of the time. An RFC without a **vault charter** is not a decision — it is an invitation for the next agent session to improvise the boundary.

| Vault axis | RFC/ADR section | Fails when |
|---|---|---|
| **Vault charter** (what is immutable, to whom, enforced how) | Context + Decision | Enforcement is "we agreed not to" |
| **Source of truth and codegen direction** | Decision | The generator's output can be hand-edited without a diff check |
| **Migration and fixture unit** | Consequences → migration surface | Fixtures are re-baselined silently; the contract change is invisible |
| **Independent verification** | Verification → anchor + red/green proof | "The agent confirmed it" or "the docs now match" |
| **Reversal and expiry** | Consequences → reversibility + expiry clause | No operation undoes the decision; no reader is named to notice it going stale |

```markdown
# ADR-021 — Vault charter for the orders contract

## Vault charter
- **In the vault (read-only to every generator):** `schema/orders.sql`, `schema/golden/**`,
  `openapi.yaml`, `error_taxonomy.ts`, and the generated `gen/types.ts`
- **Source of truth:** `schema/orders.sql` + `openapi.yaml` (human-authored)
- **Codegen direction:** vault → `gen/types.ts` via `make types` (never the reverse)
- **Never generator-authored:** every path above, plus `vault.manifest` itself
- **Derived tier:** `docs/**`, `README.md` — free authoring, must cite `path:line@hash`
- **Quarantined tier:** `notes/**`, chat transcripts, agent summaries — excluded from the
  authoritative index; this is the mechanism that terminates the self-referential loop
- **Enforcement:** branch protection + `CODEOWNERS @contract-owners` on vault paths;
  pre-commit rejects generator-authored vault diffs; CI `vault:check` asserts hash matches,
  no unlisted vault paths, and zero derived docs in the authoritative index;
  `make types && git diff --exit-code` asserts codegen determinism

## Context / Trigger
INC-4477's root-cause narrative changed three times across the derived docs while the
vault never changed. Two of the three versions were generated from the previous
generated version (PD = 2). The vault was not wrong; the vault was the only thing
that stayed still, and nobody was reading it.

## Decision
The vault is read-only to generation. Contract changes travel human-authored, in units
that carry the ADR, the schema change, and the golden fixture update together —
changing a fixture *is* the contract decision, not a test-maintenance chore.
Derived artifacts carry `derived_from` hashes and rank below the vault in retrieval.
The authoritative index excludes quarantine-tier content by construction.

## Consequences
- Measured cost: one human-authored unit per contract change — median 40 min, 11 changes/yr.
- Measured cost of inaction: PD ≥ 2 in 3 of 3 postmortems this quarter; two incidents
  where the "documented" behaviour existed only in generated prose.
- Verification: the new contract is enforced by a golden test that fails on the old
  contract (`TestExternalRefNotNull` — red on `main @ 4f9c1ab`, green on `3a77e02`);
  codegen determinism verified by `git diff --exit-code`.
- Reverses if: contract changes exceed ~4 per quarter and the human-authored unit
  becomes the bottleneck — in which case split the vault into independently versioned
  contracts rather than relaxing read-only.
- Expiry: re-evaluate 2026-10-01 (@dana), or immediately if `vault:check` reports
  > 0 mismatches for two consecutive weeks.
```

**Rules for the RFC side of the invariant:**

- **Name the enforcement mechanism per path.** *"We will be disciplined"* is not a boundary; `CODEOWNERS` plus a pre-commit rejection plus a CI hash check is a boundary.
- **State the codegen direction explicitly and verify it by regeneration.** Any pipeline where generated output *can* be hand-edited without an empty-diff check is a pipeline where the vault silently migrates into the code.
- **The golden fixture belongs to the decision, not to test maintenance.** A fixture updated to match the code is the anti-pattern wearing a green checkmark; a fixture updated as part of an ADR is the decision being made.
- **Grounding is the RFC's evidence base.** Every load-bearing claim about current behaviour cites `path:line@hash`. An RFC whose Context section is a paraphrased summary of other summaries has a Provenance Depth problem before it has a decision problem.
- **Reversibility and expiry are first-class consequences.** Name the operation that undoes the decision and its cost, plus the reader who will notice the assumption going stale. A charter with no expiry becomes folklore that outlives its own rationale.
- **Non-goals are mandatory.** An RFC without a non-goals list invites every reviewer to import a scope — and the first scope that gets imported is the derived documentation tier, which is precisely what must stay out.

---

## 4. Verification Checklist

- [ ] **The vault is named, hash-pinned, and mechanically read-only.** A `vault.manifest` exists with path globs, content hashes, and an owner; enforcement is mechanical (protected paths, `CODEOWNERS`, pre-commit rejection, read-only mount) rather than an instruction in a prompt; and every file touched by this change has been classified as VAULT, GENERATED, DERIVED, or QUARANTINED. **Vault Write Count = 0 for all generator-authored work.**
- [ ] **No derived artifact is used as evidence, and Provenance Depth ≤ 1.** Every vault-dependent claim (nullability, defaults, status codes, field names, enum members, wire shape, error semantics) carries a verbatim `path:line@<manifest-hash>`, at least two anchors were spot-checked in the source file, and no citation terminates in a model output, a summary, or another document. Grounding Ratio = 1.0 for contract claims, or the shortfall is labelled `HYPOTHESIS` with an owner.
- [ ] **The retrieval corpus was inspected for its own prior output.** Quarantine is real, not aspirational: agent-authored prose, chat transcripts, and scratch notes are excluded from the authoritative index (and any derived document is indexed as DERIVED with a `derived_from` hash that still exists). Drift check ran with a reported result: `0 hash mismatches, 0 unlisted vault paths, 0 derived docs in the authoritative index`.
- [ ] **Any vault change was human-authored with provenance, and verified independently.** Change travelled one of the two legal routes — deterministic regeneration from an upstream source of truth (proved by re-running the generator and asserting an empty diff) or a human-authored change with an ADR and an independent verifier. No fixture was updated to match the behaviour under test, no schema was edited to silence a failing contract test, and no vault path carries a generator commit trailer.
- [ ] **Every conflict was escalated, not resolved by rewriting the reference.** Each code/vault disagreement is reported with both locations (`path:line` for the vault, `path:line` for the code), filed with an identifier and a named human owner, and carries a stated migration or decision path. No vault path was deleted, softened, or re-baselined in order to make the artifact internally consistent — because internal consistency is the symptom, not the goal.