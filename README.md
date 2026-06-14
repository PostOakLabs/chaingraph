# The ChainGraph Standard

**An open, vendor-neutral specification for verifiable, chainable, agent-callable decision artifacts.**

[![Spec](https://img.shields.io/badge/spec-v0.1.0--draft-14B8A6)](https://postoaklabs.github.io/chaingraph/) [![License](https://img.shields.io/badge/license-CC%20BY%204.0-D4A847)](https://creativecommons.org/licenses/by/4.0/)

> A decision tool returns a **verdict, score, or verified mandate** — not reference data. ChainGraph defines a small, transport-neutral envelope for that output: it carries a **reproducible execution hash**, may cite the hashes of the artifacts it consumed (a verifiable provenance DAG), and is exposed to AI agents over standard protocols (MCP, A2A). One common receipt format, so decision tools from different vendors can be chained, audited, and independently verified by any agent.

**ChainGraph is not** a payments protocol, an identity protocol, or an inference framework. It is the connective envelope and provenance model beneath them — deliberately compatible with AP2, ACP, x402, A2A, KYA-OS, and MCP without replacing any of them.

📖 **Read the full spec:** <https://postoaklabs.github.io/chaingraph/>

---

## At a glance

**The four conformance levels** (§3) — each includes the lower ones:

| Level | Name | Requirement |
|---|---|---|
| **L1** | Artifact-emitting | Emits the envelope with a reproducible execution hash. |
| **L2** | Provenance-anchored | Implements the `chain` block; cites consumed artifacts' hashes; re-verifiable. |
| **L3** | Agent-callable | Exposes the tool over MCP `tools/call` and/or an A2A skill, returning the artifact as structured content. |
| **L4** | Discoverable | Publishes a graph index of tools + chain edges, linked from a discovery surface. |

**The Five Tests** (§4) — a tool is conformant iff it passes all five: (1) emits a valid envelope, (2) exposes a typed agent endpoint, (3) exports a decision not context, (4) is chainable, (5) carries a verifiable hash anchor.

**The execution hash** (§6) — `sha256:` + SHA-256 over the canonical (recursively key-sorted, whitespace-stripped, array-order-preserving) JSON of `{policy_parameters, output_payload}`. Any party recomputes it to verify an artifact — no trusted signer. Tools MUST be deterministic (seed RNG from `policy_parameters`).

---

## Repository layout

```
chaingraph/
├── index.html                     # The rendered standard (GitHub Pages)
├── README.md                      # This file
├── schema/
│   └── chaingraph-v0.1.json       # JSON Schema for the artifact envelope (§5)
└── ext/
    └── x-chaingraph/
        └── v0.1.json              # A2A capability-extension descriptor (§8.2)
```

- **Artifact schema:** [`schema/chaingraph-v0.1.json`](schema/chaingraph-v0.1.json)
- **A2A extension descriptor:** [`ext/x-chaingraph/v0.1.json`](ext/x-chaingraph/v0.1.json)

---

## Minimal conformant artifact (L1)

```json
{
  "chaingraph_version": "0.1.0",
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

---

## Adopting ChainGraph in your own suite

To make an existing decision-tool suite conformant (§11):

1. **L1** — wrap outputs in the envelope and compute the execution hash. Usually a serialization change, not a logic change. Pick your `mandate_namespace` (e.g. `io.apexlogics`, `ocs`).
2. **L2** — add the `chain` block: copy each consumed artifact's `execution_hash` into `parent_hashes`.
3. **L3** — expose `tools/call` + `verify_execution_hash` (+ `emit_<vendor>_artifact`, `build_chaingraph`).
4. **L4** — publish a graph index and an A2A agent card declaring the `x-chaingraph` extension; link both from `llms.txt`.

Once two suites are L4 with `x-chaingraph`, an orchestrating agent can chain artifacts **across vendors** and verify every link through each vendor's `verify_execution_hash`. That cross-suite provenance graph is the point of a shared standard.

**Register your namespace.** Open an issue to register your `mandate_namespace` and avoid collisions.

---

## Reference implementation

The [AINumbers ChainGraph Suite](https://ainumbers.co/chaingraph/chaingraph-hub.html) (Post Oak Labs) is the canonical reference implementation — 32+ hash-anchored tools, an MCP server with `verify_execution_hash` and `build_chaingraph`, and a published graph index at [`chaingraph.json`](https://ainumbers.co/chaingraph/chaingraph.json).

## License

Text and reference schemas are licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — reuse and extend with attribution to Post Oak Labs.

---

*The ChainGraph Standard v0.1.0 (DRAFT) · Editor: [Post Oak Labs](https://postoaklabs.com) · 2026-06-14*
