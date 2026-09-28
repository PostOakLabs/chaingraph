# The ChainGraph Standard

**An open, vendor-neutral specification for verifiable, chainable, agent-callable decision artifacts.**

[![Spec](https://img.shields.io/badge/artifact%20format-v0.4-14B8A6)](https://postoaklabs.github.io/chaingraph/) [![License](https://img.shields.io/badge/license-CC%20BY%204.0-D4A847)](https://creativecommons.org/licenses/by/4.0/)

> **Two version axes, don't conflate them.** The artifact *format* and schema are pinned at **v0.4** (`chaingraph_version` 0.4.x; the execution-hash preimage is frozen, so artifacts keep verifying forever). The specification *document* evolves additively around that frozen format: `SPEC.md` frontmatter `spec_version` carries the exact current version. The badge tracks the format axis because it is the stable one; never hand-type a spec patch version here — it drifts as fast as any other count.

> A decision tool returns a **verdict, score, or verified mandate** — not reference data. ChainGraph defines a small, transport-neutral envelope for that output: it carries a **reproducible execution hash**, may cite the hashes of the artifacts it consumed (a verifiable provenance DAG), and is exposed to AI agents over standard protocols (MCP, A2A). One common receipt format, so decision tools from different vendors can be chained, audited, and independently verified by any agent.

> Naming: **OpenChainGraph** is the standard's formal name (see `SPEC.md`); **ChainGraph** is the short name its surfaces and this mirror use.

**ChainGraph is not** a payments protocol, an identity protocol, or an inference framework. It is the connective envelope and provenance model beneath them — deliberately compatible with AP2, ACP, x402, A2A, KYA-OS, and MCP without replacing any of them.

📖 **Read the full spec:** <https://postoaklabs.github.io/chaingraph/>

---

## At a glance

**The four adoption levels** — each includes the lower ones. This ladder is adoption guidance defined here in this README (the artifact field `ocg:conformance_level` records where a suite sits):

| Level | Name | Requirement |
|---|---|---|
| **L1** | Artifact-emitting | Emits the envelope with a reproducible execution hash. |
| **L2** | Provenance-anchored | Implements the `chain` block; cites consumed artifacts' hashes; re-verifiable. |
| **L3** | Agent-callable | Exposes the tool over MCP `tools/call` and/or an A2A skill, returning the artifact as structured content. |
| **L4** | Discoverable | Publishes a graph index of tools + chain edges, linked from a discovery surface. |

**How the spec grades conformance.** The spec's normative conformance surface is the **§15 gate suite** (conformance-by-construction: gates run over artifacts, so a conformant artifact is one the gates pass). Separately, the **strength-of-verifiable ladder (§18.4)** grades how strongly an artifact is verifiable: L1 §4 hash (tamper-evidence) → L2 §16 proof binding (authenticated attestation) → L3 §18 compute receipt (succinct compute-integrity). Note the two ladders use L-numbers with different meanings; this README keeps the adoption ladder's numbering because artifacts already carry it.

**The execution hash** — `sha256:` + SHA-256 over the RFC 8785 (JCS) canonical JSON of `{policy_parameters, output_payload}`. Any party recomputes it to verify an artifact — no trusted signer. Tools MUST be deterministic (seed RNG from `policy_parameters`). See `SPEC.md` §4 for the normative preimage rule — a plain recursive-key-sort canonicalizer is a close cousin of JCS but not identical on edge cases (number formatting, escaping); implementations MUST use a real RFC 8785 canonicalizer, not hand-roll one.

---

## Repository layout

```
chaingraph/
├── index.html                              # Landing page (mirror-owned, hand-edited here)
├── SPEC.md                                 # Normative spec, MARKDOWN SOURCE (synced — see below)
├── spec.html                               # Normative spec, RENDERED (synced — see below)
├── schema/
│   ├── openchain-graph-v0.4.schema.json    # JSON Schema for the artifact envelope (synced)
│   └── chaingraph-v0.1.json                # Legacy v0.1 envelope schema (superseded, kept for history)
├── ext/
│   └── x-chaingraph/v0.1.json              # A2A capability-extension descriptor (mirror-owned)
├── profiles/                               # Compute Binding / Export Profile docs (mirror-owned)
├── namespaces/                             # Registered mandate_namespace values (mirror-owned)
├── adoption/                               # Third-party adoption writeups (mirror-owned)
└── README.md                               # This file (mirror-owned)
```

**This is a mirror, not the SSOT.** `SPEC.md`, `spec.html`, and `schema/openchain-graph-v0.4.schema.json` are
auto-synced on every push from the site repo's `PostOakLabs/ainumbers` `chaingraph/standard/` — a GitHub Actions
workflow there pushes here via a short-lived GitHub App token whenever those files change. **Never hand-edit
those three files in this repo** — edit `repo/chaingraph/standard/SPEC.md` (+ `openchain-graph-spec.html`) in the
site repo instead and let the sync propagate. Everything else in this repo (`index.html`, `profiles/`,
`namespaces/`, `adoption/`) is mirror-owned and edited directly here. When committing here, use explicit paths
(`git add index.html profiles/... `) — **never `git add -A`**, which has previously swept uncommitted WIP into a
spec-sync commit and deleted published profile files.

---

## Minimal conformant artifact (L1)

```json
{
  "chaingraph_version": "0.4.0",
  "mandate_type": "compliance_mandate",
  "tool_id": "example-sanctions-screen",
  "tool_version": "1.0.0",
  "generated_at": "2026-06-14T00:00:00Z",
  "execution_hash": "sha256:…",
  "chain": { "parent_hashes": [], "parent_tool_ids": [], "chain_depth": 0 },
  "policy_parameters": { "execution_backend": "js", "input_parameters": { "name": "ACME SYNTHETIC LTD", "list": "OFAC-SDN" } },
  "output_payload": { "match": false, "score": 0.02, "decision": "clear" },
  "compliance_flags": [],
  "audit_signature": { "client_side_executed": true, "zero_pii_verified": true, "deterministic_run": true }
}
```

## Verify an artifact (browser, no library)

```js
async function executionHash(policy_parameters, output_payload) {
  const canon = (v) => Array.isArray(v) ? v.map(canon)
    : (v && typeof v === 'object')
      ? Object.keys(v).sort().reduce((o,k)=>(o[k]=canon(v[k]),o),{})
      : v;
  const preimage = JSON.stringify(canon({ policy_parameters, output_payload }));
  const buf = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(preimage));
  return 'sha256:' + [...new Uint8Array(buf)].map(b=>b.toString(16).padStart(2,'0')).join('');
}
// valid === (await executionHash(a.policy_parameters, a.output_payload)) === a.execution_hash
```

This snippet is a simplified illustration (recursive key-sort only) — it agrees with full RFC 8785 (JCS) on plain
ASCII JSON but diverges on edge cases (number formatting, Unicode escaping). For a spec-compliant reference
implementation, see [`chaingraph/kernels/_hash.mjs`](https://github.com/PostOakLabs/ainumbers/blob/main/repo/chaingraph/kernels/_hash.mjs)
in the reference implementation repo.

---

## Adopting ChainGraph in your own suite

To make an existing decision-tool suite conformant, walk the adoption levels:

1. **L1** — wrap outputs in the envelope and compute the execution hash. Usually a serialization change, not a logic change. Pick your `mandate_namespace` (e.g. `org.apexlogics`, `me.omegacentauri`).
2. **L2** — add the `chain` block: copy each consumed artifact's `execution_hash` into `parent_hashes`.
3. **L3** — expose `tools/call` + `verify_execution_hash` (+ `emit_<vendor>_artifact`, `build_chaingraph`).
4. **L4** — publish a graph index and an A2A agent card declaring the `x-chaingraph` extension; link both from `llms.txt`.

Once two suites are L4 with `x-chaingraph`, an orchestrating agent can chain artifacts **across vendors** and verify every link through each vendor's `verify_execution_hash`. That cross-suite provenance graph is the point of a shared standard.

**Register your namespace.** Open an issue to register your `mandate_namespace` and avoid collisions.

---

## Reference implementation

The [AINumbers ChainGraph Suite](https://ainumbers.co/chaingraph/chaingraph-hub.html) (Post Oak Labs) is the canonical reference implementation — hundreds of hash-anchored tools (current count in the hub's live stats, never hand-typed here since it drifts), an MCP server with `verify_execution_hash` and `build_chaingraph`, and a published graph index at [`chaingraph.json`](https://ainumbers.co/chaingraph/chaingraph.json).

## License

Text and reference schemas are licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — reuse and extend with attribution to Post Oak Labs.

---

*The ChainGraph Standard v0.4 · Editor: [Post Oak Labs](https://postoaklabs.com) · see `SPEC.md` frontmatter for the exact patch and date*
