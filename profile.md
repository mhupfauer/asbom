# CycloneDX Agent System Profile

**Status:** Draft v0.1
**Target spec:** CycloneDX 1.7
**Author:** Markus Hupfauer
**Date:** 2026-05-25

## 1. Scope

This profile defines a usage convention on top of CycloneDX 1.7 for describing **agent systems**: bounded compositions of one or more AI models, a host application ("harness") that drives them, zero or more installed skills, zero or more invoked MCP servers, a runtime/sandbox environment, and the identities under which each of those components executes.

It is a *profile*, not a new specification. With one optional extension (the [`principal` proposal](./proposal-principal-field.md)) it is expressible entirely in CycloneDX 1.7 primitives. The profile defines:

- a component taxonomy mapping agent-system parts to existing CycloneDX `component` and `service` types,
- a composition pattern using `compositions` and `formulation`,
- attestation rules using `declarations`,
- signing requirements,
- and naming conventions for the `properties` bag.

It does not introduce new top-level fields. It does cite the proposed `principal` block for component/service identity; consumers that have not adopted that proposal can substitute a `properties` namespace as described in §5.4.

## 2. Motivation

CycloneDX ML-BOM describes the *model*. CycloneDX SaaSBOM describes a *service surface*. Neither describes the **running composition** that determines what an agent can actually do: which model, loaded in which harness, with which skills installed, executing in which runtime, under which credentials.

That composition is the unit operators must audit, sign, attest, and respond to in an incident. Today it has no home in the schema. This profile gives it one without inventing a new format.

## 3. Conformance

A document conforms to this profile if:

1. It is a valid CycloneDX 1.7 BOM.
2. Its `metadata.properties` contains `cdx:profile = "agent-system"` with the profile version, e.g., `cdx:profile:agent-system:version = "0.1"`.
3. It contains at least one `composition` of `aggregate: complete` (or `complete_with_exceptions`) that enumerates the agent system's runtime parts.
4. Each component or service that *executes code under some identity* carries either a `principal` block (preferred) or a `cdx:agent:principal:*` properties namespace (fallback; see §5.4).

A document MAY include additional CycloneDX content unrelated to the agent system; only the conforming composition needs to satisfy this profile.

## 4. Component taxonomy

The profile maps agent-system parts to existing CycloneDX 1.7 types as follows. Where multiple types would be defensible, the profile picks one to keep producers and consumers aligned.

### 4.1 Model

CycloneDX `component.type = "machine-learning-model"` with a `modelCard`. No new convention. The model is named by its vendor and version. If the model is consumed via API rather than executed locally, the `component` represents the model itself and a *separate* `service` represents the API endpoint (see §4.5).

### 4.2 Harness (agent host application)

The CLI, IDE, or desktop application that loads the model, loads skills, mediates tool calls, and presents the UI to the user. Modeled as `component.type = "application"` with:

- `properties."cdx:agent:role" = "harness"`
- `properties."cdx:agent:harness:vendor"` (e.g., `anthropic`, `cursor`, `github`)

Examples: Claude Code CLI, Cursor, Gemini CLI, the GitHub CLI in `gh copilot` mode.

The harness MUST declare a `principal` describing the OS-level identity it runs under.

### 4.3 Skill

An installed agent capability — a SKILL.md package, an Anthropic plugin, a Cursor extension, etc. Modeled as `component.type = "application"` with:

- `properties."cdx:agent:role" = "skill"`
- `properties."cdx:agent:skill:format"` (e.g., `agentskills.io/0.1`, `anthropic-plugin/1.0`)
- `properties."cdx:agent:skill:source"` (URL or registry URI the skill was installed from)
- `properties."cdx:agent:skill:digest"` (full content hash; SHOULD use `hashes` array as well)

A skill MAY declare a `principal`. Skills that execute under the harness's ambient identity SHOULD set `principal.mode = "delegated-ambient"` and `principal.subject.ref` to the harness's `bom-ref` — making the inheritance explicit rather than implicit.

### 4.4 MCP server

An MCP server invoked by the harness or by a skill. Modeled as `service`, not `component`, even when colocated with the harness, because MCP is fundamentally a protocol-mediated service surface. Properties:

- `properties."cdx:agent:role" = "mcp-server"`
- `properties."cdx:agent:mcp:transport"` (e.g., `stdio`, `http+sse`, `streamable-http`)
- `endpoints` MUST list every endpoint the server exposes; if `stdio`, use a synthetic `stdio://` URI naming the binary.

An MCP server MUST declare a `principal`. This is the field that distinguishes a token-passthrough shim from a properly OBO-flow server (see [the principal proposal §3](./proposal-principal-field.md#axis-1--attribution-mode-who-the-audit-log-records-as-the-actor)).

### 4.5 External AI / data services

Any external service the model, harness, skill, or MCP server calls — including the model's own API endpoint when the model is hosted. Modeled as `service` with:

- `properties."cdx:agent:role" = "external-api"`
- `services.endpoints` listing the calling endpoint(s)
- `services.data` describing classification and direction
- `services.trustZone` set to a meaningful name (e.g., `public-internet`, `vendor-saas`, `internal`)

These services SHOULD NOT carry a `principal` block — the principal lives on whichever component or service *calls* them. (Modeling it on both ends is allowed but creates ambiguity; prefer the caller.)

### 4.6 Runtime environment

The sandbox, user account, or container under which the harness executes. Modeled as `component.type = "platform"` (1.7 explicitly clarifies that `platform` covers interpreters and runtime hosts). Properties:

- `properties."cdx:agent:role" = "runtime"`
- `properties."cdx:agent:runtime:kind"` — one of:
  - `host-user-session` — running in the interactive user's OS session under the user's account
  - `agent-os-account` — running under a dedicated OS account (e.g., Windows 11 Agent Workspace)
  - `container` — running in a container with declared isolation
  - `vm` — running in a VM
  - `cloud-managed` — running in a vendor-managed agent runtime (Anthropic Managed Agents, etc.)
- `properties."cdx:agent:runtime:isolation"` — free text, e.g., `none`, `os-account+acl`, `app-container`, `oci-container:user-namespace`, `hyperv-vm`, `vendor-attested`.

The runtime component MUST carry a `principal` whose `mode` is one of `agent-account-os`, `service-principal`, or `none`. The `mode = none` value is appropriate for `host-user-session` runtimes whose attribution is the user's existing OS identity rather than a distinct agent principal.

### 4.7 Credentials and identities

Credentials are *not* modeled as standalone components. They are modeled as properties of a `principal` block attached to whichever component or service holds them (§4.2–§4.4). This avoids the trap of inventorying secret material directly and keeps the schema's grain at the principal rather than at the credential.

When the same credential is held by multiple components (e.g., a shared OAuth refresh token), each component's `principal` block describes its own access path to that credential; the credential itself is not a referenceable BOM object.

## 5. Composition pattern

### 5.1 The agent-system composition

Every conforming BOM MUST contain exactly one `composition` representing the agent system as a whole:

```jsonc
{
  "compositions": [
    {
      "bom-ref": "comp:agent-system:claude-code-with-salesforce-skill",
      "aggregate": "complete",
      "assemblies": [
        "comp:model:claude-opus-4-7",
        "comp:harness:claude-code-1.2.0",
        "comp:skill:salesforce-export-1.4.2",
        "comp:runtime:macos-user-session",
        "svc:mcp:salesforce-mcp-0.9.1"
      ]
    }
  ]
}
```

The `aggregate` value MUST be `complete` if the producer asserts the listed assemblies fully describe the runtime composition. Use `complete_with_exceptions` when transitive dependencies of the listed parts are not enumerated; use `incomplete_first_party_only` when only the producer's own components are listed.

The composition's `signature` (JSF) is what a fleet admin or registry attests when approving the composition.

### 5.2 Dependencies

`dependencies` SHOULD record the call relationships among agent-system parts:

- harness `depends-on` model and skills
- skills `depends-on` MCP servers they invoke
- MCP servers `depends-on` external APIs they call

This produces a directed acyclic graph operators can walk for blast-radius queries.

### 5.3 Formulation: how this composition was assembled

The agent system's `formulation` captures the assembly process. The 1.7 release extended formulation to apply to "any referencable object within the BOM, including … the BOM itself," which is exactly the level this profile uses. Recommended structure:

```jsonc
{
  "formulation": [{
    "bom-ref": "formula:claude-code-session-2026-05-25",
    "workflows": [{
      "bom-ref": "wf:session-bootstrap",
      "uid": "session-2026-05-25-...",
      "name": "Agent session bootstrap",
      "taskTypes": ["build", "deploy"],
      "tasks": [
        {
          "uid": "task:load-model",
          "name": "Load model via vendor API",
          "resourceReferences": [{ "ref": "comp:model:claude-opus-4-7" }, { "ref": "svc:api:anthropic" }]
        },
        {
          "uid": "task:install-skill",
          "name": "Install skill from registry",
          "resourceReferences": [{ "ref": "comp:skill:salesforce-export-1.4.2" }],
          "inputs": [{ "source": { "ref": "svc:registry:internal-skills" } }]
        },
        {
          "uid": "task:bind-mcp",
          "name": "Bind MCP server with OBO identity",
          "resourceReferences": [{ "ref": "svc:mcp:salesforce-mcp-0.9.1" }]
        }
      ],
      "runtimeTopology": [
        { "ref": "comp:harness:claude-code-1.2.0", "dependsOn": ["comp:runtime:macos-user-session"] },
        { "ref": "comp:skill:salesforce-export-1.4.2", "dependsOn": ["comp:harness:claude-code-1.2.0"] },
        { "ref": "svc:mcp:salesforce-mcp-0.9.1",     "dependsOn": ["comp:harness:claude-code-1.2.0"] }
      ]
    }]
  }]
}
```

The `runtimeTopology` field is the load-bearing one: it lets a consumer reconstruct what was *executing under what* at the moment the BOM was produced, which is precisely the information that disappears between an SBOM's "what is installed" and a SaaSBOM's "what services exist."

### 5.4 Property-bag fallback for `principal`

For BOMs targeting consumers that have not adopted the [`principal` proposal](./proposal-principal-field.md), the same fields MAY be expressed under the `cdx:agent:principal:*` property namespace:

```jsonc
"properties": [
  { "name": "cdx:agent:principal:mode", "value": "delegated-obo" },
  { "name": "cdx:agent:principal:subject:kind", "value": "compound-user-and-client" },
  { "name": "cdx:agent:principal:subject:identifier", "value": "alice@example.com + client:salesforce-mcp" },
  { "name": "cdx:agent:principal:provider:name", "value": "salesforce" },
  { "name": "cdx:agent:principal:provider:issuer", "value": "https://login.salesforce.com" },
  { "name": "cdx:agent:principal:scopes", "value": "api refresh_token id" },
  { "name": "cdx:agent:principal:credential:storage", "value": "os-keychain" },
  { "name": "cdx:agent:principal:credential:lifetime", "value": "short-lived-oauth" },
  { "name": "cdx:agent:principal:credential:replayProtection", "value": "dpop-bound" },
  { "name": "cdx:agent:principal:consent:binding", "value": "per-grant" },
  { "name": "cdx:agent:principal:attestation:level", "value": "idp-attested" }
]
```

Producers SHOULD prefer the structured `principal` block once the proposal lands. Consumers SHOULD accept both for the transition period.

## 6. Attestation via `declarations`

The profile uses CycloneDX 1.7 `declarations` for the third-party-assertion layer. Trust in an agent-system BOM derives from who signed which assertion about it, not from the artifact's self-description.

Minimum recommended declarations for a fleet-approved composition:

| Claim | Asserted by | Evidence |
|---|---|---|
| Composition matches the org's approved-skill policy | Fleet admin (third-party assessor) | Signature over the `composition.bom-ref` and the resolved skill digests |
| Every skill in the composition was installed via the org's policy-aware registry | Registry operator | Registry-side install log referenced by URI |
| MCP servers in the composition use OBO flow, not token passthrough | MCP server vendor | OAuth client registration evidence |
| The runtime principal is OS-attested | OS / endpoint mgmt | Intune/MDM device-compliance posture, EDR runtime trace, etc. |

Each `declaration` carries:

- `assessors[].thirdParty: true` for assertions made by parties other than the BOM producer
- `attestations[].map` linking the claim to the relevant `bom-ref` (component, service, or composition)
- A JSF signature over the attestation

The point is: **a BOM is governance only to the extent that its claims are signed by parties whose authority over the claim is operationally meaningful**. A `principal.mode: delegated-obo` assertion in a self-signed BOM tells the consumer the producer *says* the MCP server does OBO; an attestation signed by the registry operator after running an actual OAuth-flow probe tells the consumer the registry *verified* it.

## 7. Signing

Every conforming BOM MUST be signed (JSF, enveloped, at the top level). The signer is the BOM producer.

A fleet-approved composition SHOULD additionally carry:

- a signed `composition` (JSF over the composition object) by the fleet admin's signing key
- signed `declarations.attestations[]` for at least the policy-conformance and registry-provenance claims above

Verification flow at runtime (e.g., in `npx skills`, a managed harness, or a gateway):

1. Verify the envelope signature against the producer's key.
2. Verify the composition signature against the fleet admin's key from the org's trust policy.
3. For each declared attestation, verify the assessor's signature against the assessor's published key.
4. Refuse the composition if any required attestation is missing or fails.

## 8. Worked example

See [`example-claude-code-session.cdx.json`](./example-claude-code-session.cdx.json) for a complete BOM modeling a Claude Code session with one skill, one MCP server, and a Windows 11 Agent Workspace runtime, including the `principal` block on every runtime actor and a fleet-admin attestation over the composition.

## 9. Implementation guidance

### 9.1 Producers

- A policy-aware registry CLI (e.g., `npx skills` with [vercel-labs/skills#1254](https://github.com/vercel-labs/skills/pull/1254)) SHOULD emit a conforming BOM at install time, attaching it to the install log and (optionally) writing it to a stable on-disk path for fleet tooling to read.
- A harness SHOULD emit a conforming BOM at session start, capturing the current runtime composition and any user-consent events that bound the session's `principal` blocks.

### 9.2 Consumers

- A fleet-management agent (Intune script, JAMF script, osquery extension) SHOULD verify the BOM signature chain and refuse compositions missing required attestations.
- An MCP gateway SHOULD ingest the conforming BOM and use the declared `principal` to decide whether to require additional consent, route through a token-exchange broker, or refuse the call entirely.
- An incident-response query SHOULD be able to answer, against a corpus of conforming BOMs, "which compositions on which hosts hold a `credential.lifetime: long-lived-pat` with `credential.storage: env-var` or `file`."

### 9.3 Where this profile does not help yet

- Runtime data-flow / value provenance (the "no egress after sensitive read" policy from the source blog post) is *not* modeled here. That belongs in the gateway/runtime, not the BOM. The BOM names the principals and trust zones the gateway then enforces against.
- The transitive permission graph beyond the second hop (skill → MCP server → external API → further APIs) is modeled only to the depth the producer can observe. Deep transitive analysis remains an out-of-band concern.

## 10. Open questions

1. Should the profile mandate a specific `bom-ref` naming convention (e.g., `comp:role:name:version`)? Suggest: SHOULD, not MUST, with the convention above as the recommended form.
2. How should multi-model compositions be modeled (a session that uses both Opus and Haiku)? Probably as two `machine-learning-model` components both listed in the composition's `assemblies`, with `formulation.workflow.tasks` indicating which task used which model. Worked example to follow.
3. How should ephemeral compositions be expressed — e.g., a tool call that briefly invokes a sub-agent? Probably a nested `workflow` within the same `formulation`. Worked example to follow.

## Appendix A: Mapping to existing standards

| Concept | This profile | SBOM (CycloneDX core) | ML-BOM | SaaSBOM | SLSA | in-toto |
|---|---|---|---|---|---|---|
| Inventory of software parts | components | ✓ | ✓ | ✗ | ✗ | ✗ |
| Model card | model component | ✗ | ✓ | ✗ | ✗ | ✗ |
| Service surface | mcp-server / external-api | ✗ | ✗ | ✓ | ✗ | ✗ |
| Runtime composition (what's running together) | composition + formulation.runtimeTopology | partial | ✗ | ✗ | ✗ | partial |
| Build provenance | declarations + formulation | partial | ✗ | ✗ | ✓ | ✓ |
| Runtime identity / principal | `principal` (proposed) | ✗ | ✗ | partial | ✗ | ✓ (predicate) |
| Third-party attestation | declarations | ✓ | ✓ | ✓ | ✓ | ✓ |
| Non-repudiation properties | `principal.attestation` (proposed) | ✗ | ✗ | ✗ | ✗ | partial |

The profile is best understood as the CycloneDX-side wiring that lets an agent system carry the equivalent of an in-toto principal predicate as an inline schema-validated block, signed alongside everything else in the BOM.
