# Agentic-System Bill of Materials (ASBOM)

Working drafts for a new CycloneDX BOM type — **Agentic-System Bill of Materials (ASBOM)** — sitting alongside SBOM, ML-BOM, SaaSBOM, and CBOM. ASBOM inventories the *runtime composition* of an agentic AI deployment: model + harness + skills + MCP servers + runtime + the identities under which each part executes.

ASBOM is expressible entirely in CycloneDX 1.7 primitives with one schema addition: a `principal` block that precisely models *who the audit-log actor is, what credential backs that identity, where the credential lives, and whether a human consent event is bound to its use* — four orthogonal axes that today collapse into `services.authenticated: true`.

## Contents

- **[`profile.md`](./profile.md)** — ASBOM v0.1 specification. Conformance rules, component taxonomy, composition pattern (`compositions`, `formulation.runtimeTopology`), attestation via `declarations`, signing, property-bag fallback. Positions ASBOM as a new BOM type parallel to the existing CycloneDX capability set.
- **[`proposal-principal-field.md`](./proposal-principal-field.md)** — Schema PR draft for the one new field ASBOM requires. Decomposes credential modeling into four orthogonal axes (attribution mode, storage, lifetime, consent binding), frames the gap around non-repudiation, ships a JSON Schema fragment and worked diff examples (OBO MCP server vs. PAT shim). Generally useful beyond ASBOM (CI pipelines, serverless chains, multi-tenant SaaS).
- **[`example-claude-code-session.cdx.json`](./example-claude-code-session.cdx.json)** — Valid CycloneDX 1.7 ASBOM applied to a realistic scenario: Claude Code in Windows 11 Agent Workspace, one skill installed via internal registry, one MCP server using delegated-OBO with DPoP-bound OAuth tokens in OS keychain, plus signed fleet-admin and registry-operator attestations.

## Why a new BOM type

| BOM type | Inventories |
|---|---|
| SBOM | Software components |
| HBOM | Hardware components |
| ML-BOM | Models, training datasets, training methodology |
| SaaSBOM | Service surfaces, endpoints, data flows |
| CBOM | Cryptographic assets |
| **ASBOM** | **Runtime composition of an agentic system + identities under which each part executes** |

Where ML-BOM answers "what is this model and how was it trained" and SaaSBOM answers "what services and data flows exist," ASBOM answers "what composition is executing *right now*, under whose authority, with what reversibility, and with what non-repudiation properties." None of the existing types capture the *running stack of identities* — and that is the unit operators must audit, sign, and respond to in an agentic-AI incident.

## Related

- Blog post motivating the broader argument: [The registry is the control plane](https://hupfauer.one/posts/the-registry-is-the-control-plane/)
- Reference implementation seed for policy-aware skill installation (the natural ASBOM producer): [vercel-labs/skills#1254](https://github.com/vercel-labs/skills/pull/1254)

## Status

Draft v0.1. Will be opened as a proposal under CycloneDX/specification → Ideas, Proposals, RFCs.

## License

CC0-1.0. These are spec drafts intended to be incorporated into upstream standards work; no rights reserved.
