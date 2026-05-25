# Agent System BOM — CycloneDX Profile & `principal` field proposal

Working drafts for extending [CycloneDX](https://cyclonedx.org/) to describe agent systems — bounded compositions of one or more AI models, a host application ("harness"), installed skills, MCP servers, a runtime environment, and the identities under which each part executes.

The single substantive schema addition is a `principal` block that precisely models *who the audit-log actor is, what credential backs that identity, where the credential lives, and whether a human consent event is bound to its use* — four orthogonal axes that today collapse into `services.authenticated: true`.

## Contents

- **[`proposal-principal-field.md`](./proposal-principal-field.md)** — Schema PR draft. Decomposes credential modeling into four orthogonal axes (attribution mode, storage, lifetime, consent binding), frames the gap around non-repudiation, ships a JSON Schema fragment and worked diff examples (OBO MCP server vs. PAT shim).
- **[`profile.md`](./profile.md)** — CycloneDX Agent System Profile v0.1. Usage convention on top of CycloneDX 1.7 primitives (`compositions`, `formulation.runtimeTopology`, `declarations`). Conformance rules, taxonomy, attestation, signing.
- **[`example-claude-code-session.cdx.json`](./example-claude-code-session.cdx.json)** — Valid CycloneDX 1.7 BOM applying the profile to a realistic scenario: Claude Code in Win11 Agent Workspace, one skill installed via internal registry, one MCP server using delegated-OBO with DPoP-bound OAuth tokens in OS keychain, plus signed fleet-admin and registry-operator attestations.

## Related

- Blog post motivating the proposal: [The registry is the control plane](https://hupfauer.one/posts/the-registry-is-the-control-plane/)
- Reference implementation seed for policy-aware skill installation: [vercel-labs/skills#1254](https://github.com/vercel-labs/skills/pull/1254)

## Status

Draft v0.1. The proposal will be opened as a discussion under CycloneDX/specification → Ideas, Proposals, RFCs.

## License

CC0-1.0. These are spec drafts intended to be incorporated into upstream standards work; no rights reserved.
