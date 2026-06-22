# OCS × ChainGraph Standard — Adoption Spec

**Project:** Omega Centauri Society (OCS) — interactive astrophysics calculator suite at `omegacentauri.me` (IMBH evidence, black-hole physics, Fermi paradox, SETI, MTH, Kardashev).
**Target:** conform the OCS suite to **The ChainGraph Standard v0.1** ([spec](https://postoaklabs.github.io/chaingraph/)).
**Date:** 2026-06-14 · **Maintainer:** Post Oak Labs.
**Status:** PROPOSED. Drop-in spec for the OCS repo (not in the current workspace). OCS is the **closest existing adopter** — it already emits a hash-anchored ChainGraph-style artifact; this spec is mostly alignment, not new construction.

---

## 0 · Current state (from a live `constraint_stacker` artifact, 2026-06-14)

OCS already emits this envelope — note it is ~**L2** out of the box (envelope + `chain` block + `execution_hash`):

```json
{
  "ocs_version": "1.0.0",
  "mandate_type": "imbh_constraint",
  "tool_id": "constraint-stacker",
  "tool_version": "1.0.0",
  "generated_at": "2026-06-14T17:14:14.931Z",
  "execution_hash": "sha256:8ef505a5…",
  "chain": { "parent_hashes": [], "parent_tool_ids": [], "chain_depth": 0 },
  "policy_parameters": { "epsilon_adaf": 0.001, "rho_inf_kg_m3": 1e-21, "...": "..." },
  "output_payload": { "allowed_window_M_solar": { "lower": 8200, "upper": 709 }, "tension_detected": true, "...": "..." },
  "audit_signature": { "data_sources": ["Häberle et al. 2024 …"], "schema_version": "ocs-chaingraph-1.0.0" }
}
```

This is excellent — the hard parts (envelope, chain block, execution hash, determinism, no-PII domain) are done. **Four deltas** bring it to full v0.1 conformance.

---

## 1 · The four deltas to full conformance

**Δ1 · Version field.** Emit `chaingraph_version: "0.1.0"` alongside (or in place of) `ocs_version`. `ocs_version` MAY remain as a vendor alias; new consumers key on `chaingraph_version`.

**Δ2 · Namespace the mandate type.** `mandate_type: "imbh_constraint"` is an **unprefixed non-core type — invalid under §7.3.** Namespace it:

> `"mandate_type": "me.omegacentauri/imbh_constraint"`

Use the domain-based namespace **`me.omegacentauri`** across the suite (the short registry id `ocs` is an acceptable alternative — pick one and register it on the ChainGraph repo). Type vocabulary by category (§3).

**Δ3 · Add `compliance_flags`.** The Standard REQUIRES the field (MAY be empty). OCS has a natural, valuable use for it: carry the **epistemic register and any analysis tension** as machine-readable flags, e.g.

```json
"compliance_flags": ["register:peer-reviewed", "tension:lower_bound_exceeds_upper_limit"]
```

This makes OCS's "peer-reviewed vs speculative" distinction — and the IMBH mass tension — agent-readable without parsing prose.

**Δ4 · Align `audit_signature`.** The Standard's `audit_signature` is a self-asserted **claims** object: `{client_side_executed, zero_pii_verified, deterministic_run}`. OCS currently uses that key for provenance (`data_sources`, `schema_version`). Keep both — add the three claim booleans (all `true` for OCS: client-side, zero-PII astrophysics, deterministic), and retain `data_sources` + `schema_version` as extra fields (the envelope allows additional properties):

```json
"audit_signature": {
  "client_side_executed": true,
  "zero_pii_verified": true,
  "deterministic_run": true,
  "data_sources": ["Häberle et al. 2024, Nature 631:285", "..."],
  "schema_version": "ocs-chaingraph-1.0.0"
}
```

> Keeping `data_sources` in the artifact is a genuine strength — it's the citation provenance behind the decision. The Standard doesn't define it but explicitly tolerates vendor fields.

---

## 2 · Verify the hash preimage matches §6 (do not assume)

OCS already produces an `execution_hash`. **Confirm its preimage is exactly the Standard's:** SHA-256 over canonical (recursive key-sort, whitespace-stripped, array-order-preserved) JSON of `{policy_parameters, output_payload}` only. If the current OCS hash covers other fields or a different ordering, either (a) re-point it to the §6 preimage, or (b) document the variance in the OCS graph index so cross-vendor `verify_execution_hash` doesn't false-negative. This is the one item that must be checked against the code, not the artifact.

---

## 3 · Mandate-type vocabulary (proposed, `me.omegacentauri/…`)

| Category | `mandate_type` |
|---|---|
| imbh-evidence | `me.omegacentauri/imbh_constraint` |
| bh-physics | `me.omegacentauri/bh_physics` |
| fermi-paradox | `me.omegacentauri/fermi_estimate` |
| fermi-seti | `me.omegacentauri/seti_sensitivity` |
| mth | `me.omegacentauri/transcension_model` |
| kardashev | `me.omegacentauri/kardashev_metric` |

Carry the peer-reviewed / speculative register in `compliance_flags` (Δ3) regardless of type.

---

## 4 · Domain-integrity note (preserve, don't regress)

OCS's standing rule — **the IMBH mass tension is irreconcilable; never collapse Häberle's lower bound and the competing upper limit into one number** — is already honored by the artifact: `output_payload` carries both `lower` and `upper` plus `tension_detected` and named sources. Conforming to ChainGraph MUST NOT flatten this. The `compliance_flags` `tension:*` entry (Δ3) makes the invariant explicit and machine-checkable — a net improvement, not a risk.

---

## 5 · MCP graph bindings (L3) + discovery (L4)

OCS is already L3-adjacent: `constraint_stacker` returns the artifact over MCP, and the server has `list_ocs_tools` + `build_ocs_workflow_links`. Add:

- `verify_execution_hash` and `build_chaingraph` — lift verbatim from `ainumbers-mcp-apps/worker.mjs` (the `cgCanon`/`cgExecutionHash` helpers + both `registerTool` blocks are vendor-neutral). `build_chaingraph` reads the **OCS** graph index.
- Ensure every flagship tool (not just `constraint_stacker`) returns its artifact as `structuredContent`.

**L4 discovery:** publish `omegacentauri.me/chaingraph.json` (graph index, §8.3) and `omegacentauri.me/.well-known/agent-card.json` with the `x-chaingraph` extension:

```json
"params": {
  "conformance_level": "L4",
  "graph_index": "https://omegacentauri.me/chaingraph.json",
  "verify_endpoint": "verify_execution_hash",
  "hash_alg": "sha256",
  "mandate_namespace": "me.omegacentauri"
}
```

Link both from `omegacentauri.me/llms.txt`.

---

## 6 · Definition of done

- A live OCS artifact emits `chaingraph_version`, a namespaced `mandate_type` (`me.omegacentauri/…`), `compliance_flags` (incl. register + any tension), and an `audit_signature` carrying the three claim booleans plus `data_sources`.
- The `execution_hash` preimage is confirmed equal to Standard §6 (or its variance is documented in the graph index); an independent recompute matches.
- `verify_execution_hash` + `build_chaingraph` registered; `verify_execution_hash` round-trips a `constraint_stacker` artifact.
- `chaingraph.json` + `.well-known/agent-card.json` published and linked from `llms.txt`; `me.omegacentauri` namespace registered upstream.
- Mass-tension integrity preserved end-to-end (both bounds + `tension:*` flag).

---

*OCS × ChainGraph Standard v0.1 · Post Oak Labs · 2026-06-14 · Drop-in adoption spec. Apply inside the OCS repo; companion: `apexlogics-chaingraph-adoption.md`, `../README.md` (the Standard).*
