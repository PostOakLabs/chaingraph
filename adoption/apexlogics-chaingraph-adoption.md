# ApexLogics × ChainGraph Standard — Adoption Spec

**Project:** ApexLogics — 146 deterministic, privacy-first edtech / careertech tools at `apexlogics.org`.
**Target:** conform the ApexLogics suite to **The ChainGraph Standard v0.1** ([spec](https://postoaklabs.github.io/chaingraph/)) so its artifacts are agent-verifiable and chainable across vendors (AINumbers, OCS).
**Date:** 2026-06-14 · **Maintainer:** Post Oak Labs.
**Status:** PROPOSED. This is a drop-in spec for the ApexLogics repo — staged here because that repo isn't in the current workspace. Verify the current envelope field names against a live artifact before applying (see §1).

---

## 0 · Why ApexLogics is already most of the way there

From the live MCP server (`list_apexlogics_tools`): 146 tools, every one `ap2_export: true`, each with an `al_id` (AL-01 …), a category, and a deep-link under `apexlogics.org/tools/…`. The suite already has the AINumbers shape — a catalog, AP2 exports, and a `build_workflow_links`-style MCP tool. **What's missing for ChainGraph conformance is small and mechanical:** a versioned namespace, populated `mandate_type` values, the verify/build graph bindings, and a published graph index + agent card.

**Categories in scope** (drive the mandate-type vocabulary, §3): `education_path_roi`, `compensation_offers`, `career_transition_mobility`, `licensing_credentials`, `student_finance_debt`, `workforce_development`, `immigration_visa`, `selected_studies`, `hr_analytics`, `obbba_student_loans`, `equity_tax`, `freelance_tax`.

---

## 1 · FIRST: verify the current envelope (no fabrication)

Before changing anything, fetch one real ApexLogics artifact (export from any AL tool, or call its MCP emit path) and record the actual field names. The `list_*` response shows `ap2_export: true` and an `ap2_type` field (currently `null` in sampled tools). Confirm:

- Top-level version field name (likely `ap2_version` — Standard's compat alias).
- Whether a `chain` block, `execution_hash`, `policy_parameters`, `output_payload`, `compliance_flags`, and `audit_signature` are present, and their exact shapes.
- How the execution hash (if any) is currently computed.

Fill the deltas in §4 against what you actually find — do not assume.

---

## 2 · Mandate namespace

ApexLogics decisions (career ROI, compensation, credential value) **do not map to the Standard's core finance-regulatory types**, so they MUST be namespaced (Standard §7.3). Use the domain-based reverse-DNS namespace:

> **`org.apexlogics`**

Register it by opening an issue on the ChainGraph repo (`github.com/PostOakLabs/chaingraph`) to avoid collisions.

---

## 3 · Mandate-type vocabulary (proposed)

One namespaced type per decision family. Map each tool's category → type; populate the currently-null `ap2_type`/`mandate_type` field accordingly.

| Category | `mandate_type` |
|---|---|
| compensation_offers | `org.apexlogics/compensation_assessment` |
| education_path_roi | `org.apexlogics/education_roi` |
| career_transition_mobility | `org.apexlogics/career_mobility` |
| licensing_credentials | `org.apexlogics/credential_assessment` |
| student_finance_debt · obbba_student_loans | `org.apexlogics/student_finance` |
| workforce_development · hr_analytics | `org.apexlogics/workforce_analytics` |
| immigration_visa | `org.apexlogics/immigration_assessment` |
| equity_tax · freelance_tax | `org.apexlogics/tax_projection` |
| selected_studies | `org.apexlogics/research_estimate` |

These are non-core types — consumers treat them as opaque but carry them through the chain. Keep the list versioned in the ApexLogics repo (a `mandate-types.md` or registry file).

---

## 4 · Conformance deltas (L1 → L4)

**L1 — Artifact envelope (§5).** Emit `chaingraph_version: "0.1.0"` (keep `ap2_version` as a compat alias if already present). Ensure every artifact carries `mandate_type` (§3, no longer null), `tool_id` (use the `al_id` or slug), `tool_version`, `execution_hash`, `policy_parameters`, `output_payload`, `compliance_flags` (MAY be empty), `audit_signature` (`client_side_executed`, `zero_pii_verified`, `deterministic_run`). Keep real PII out of `policy_parameters`/`output_payload` (the privacy-first posture already aligns).

**L1 — Execution hash (§6).** Compute `execution_hash = "sha256:" + hex(SHA-256(canonical_json({policy_parameters, output_payload})))` with recursive key-sort + whitespace-stripped JSON + preserved array order. Use the WebCrypto reference in the Standard. Tools must be deterministic; seed any RNG from `policy_parameters`. **Quantize derived monetary/NPV outputs to a fixed precision before hashing** (these tools are full of floats — document the precision in `policy_parameters`).

**L2 — Chain block (§7).** Add `chain: { parent_hashes, parent_tool_ids, chain_depth }`. Where a workflow already feeds one tool's output into another (e.g. credential ROI → compensation → geo-fiscal arbitrage), copy the parent's `execution_hash` into `parent_hashes`. This turns the existing `build_workflow_links` chains into verifiable DAGs.

**L3 — MCP graph bindings.** Add to the ApexLogics MCP server (mirror the AINumbers worker — code in §5):
- `verify_execution_hash` — recompute + compare (lets agents verify ApexLogics artifacts, including third-party).
- `build_chaingraph` — hash-aware DAG builder over the ApexLogics graph index.
- `emit_apexlogics_artifact` — return the signed envelope as `structuredContent` (so an agent gets the artifact, not just a deep-link).

**L4 — Discovery.** Publish `apexlogics.org/chaingraph.json` (graph index, §8.3) and `apexlogics.org/.well-known/agent-card.json` (A2A card declaring `x-chaingraph`, §6 below). Link both from `apexlogics.org/llms.txt`.

---

## 5 · MCP bindings — reuse the AINumbers worker code

`verify_execution_hash` and `build_chaingraph` are vendor-neutral; lift them verbatim from `ainumbers-mcp-apps/worker.mjs` (the canonicalization helper + both `registerTool` blocks). Only change: `build_chaingraph` reads the **ApexLogics** graph index, and `emit_apexlogics_artifact` replaces `emit_chaingraph_artifact`. The canonicalization (`cgCanon`) and `cgExecutionHash` are identical across all ChainGraph vendors — that's the point: one hash procedure, verifiable everywhere.

---

## 6 · A2A agent card (`x-chaingraph`)

Serve `apexlogics.org/.well-known/agent-card.json` declaring the suite's skills (the category families) and the ChainGraph extension:

```json
"capabilities": {
  "extensions": [{
    "uri": "https://postoaklabs.github.io/chaingraph/ext/x-chaingraph/v0.1",
    "description": "Emits ChainGraph verifiable decision artifacts (hash-anchored, chainable).",
    "params": {
      "conformance_level": "L4",
      "graph_index": "https://apexlogics.org/chaingraph.json",
      "verify_endpoint": "verify_execution_hash",
      "hash_alg": "sha256",
      "mandate_namespace": "org.apexlogics"
    }
  }]
}
```

Once ApexLogics, AINumbers, and OCS all declare `x-chaingraph`, an orchestrating agent can chain artifacts across all three and verify every link through each vendor's `verify_execution_hash`.

---

## 7 · Definition of done

- A live ApexLogics artifact emits `chaingraph_version`, a namespaced `mandate_type`, a reproducible `execution_hash`, and the `chain` block; an independent recompute of the hash matches.
- `verify_execution_hash`, `build_chaingraph`, `emit_apexlogics_artifact` registered on the MCP server; `verify_execution_hash` round-trips a known artifact.
- `chaingraph.json` graph index + `.well-known/agent-card.json` published and linked from `llms.txt`.
- `org.apexlogics` namespace + the §3 type list committed to the ApexLogics repo and registered upstream.
- No fabricated values: every populated `mandate_type` and every hash verified against a real artifact.

---

*ApexLogics × ChainGraph Standard v0.1 · Post Oak Labs · 2026-06-14 · Drop-in adoption spec. Apply inside the ApexLogics repo; companion: `ocs-chaingraph-adoption.md`, `../README.md` (the Standard).*
