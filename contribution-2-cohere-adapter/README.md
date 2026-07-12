# Contribution 2: Adapter: Cohere Model Provider

**Contribution Number:** 2
**Student:** Geethanjali Nagaboina
**Issue:** https://github.com/orthogonalhq/nous-core/issues/313
**Status:** Phase I In Progress

---

## Why I Chose This Issue

I chose this issue because it asks for a self-contained, well-documented unit of work implementing the Cohere model provider as a certified "provider leaf" rather than a change scattered across the codebase. The `nous-core` project maintains detailed contributor docs for this exact task (a quickstart, a provider-leaf anatomy reference, a schemas/ABI reference, and a testing checklist), which gives me a clear on ramp even though I haven't worked in this codebase before. It's also a good stretch from my last contribution: that one was a Python test only change, and this one is TypeScript and touches actual adapter/provider code, so I'll get exposure to a different language and a different kind of contribution (implementing an integration against a documented internal contract, not just writing tests). I'm interested in learning how a project structures a plugin-style adapter system (definition / adapter / factory contracts) so that adding a new vendor doesn't require touching shared code.

---

## Understanding the Issue

### Problem Description

`nous-core` supports multiple model providers (e.g. Anthropic, OpenAI-compatible APIs, Ollama) through a "provider leaf" system, but there is currently no Cohere provider leaf. Without it, the project cannot route requests to Cohere-hosted models the way it can for other supported vendors.

### Expected Behavior

A new certified provider leaf for Cohere should exist at:

```
self/subcortex/providers/src/providers/cohere/
├── definition.ts
├── adapter.ts
├── provider.ts
├── implementation.ts   (if the leaf owns its own request execution)
└── index.ts
```

It should:
- Export `providerDefinition` from `definition.ts` (metadata only — no env reads, network calls, or instantiation)
- Export `providerAdapter` from `adapter.ts`
- Export `providerFactory` from `provider.ts`
- Use a semantic `vendorKey` and **not** hand-author `wellKnownProviderId` (it's derived automatically)
- Declare `auth.purpose: 'api_key'`, `auth.required: true`, `auth.envVar`, `auth.vaultKeyNamespace`, and `auth.header` for API-key auth
- Regenerate the provider catalogs (`generate:providers`) rather than hand-editing generated files

### Current Behavior

No `cohere` directory exists under `self/subcortex/providers/src/providers/`, so Cohere is absent from the generated provider catalogs (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`) and cannot be selected as a model provider.

### Affected Components

- **New directory:** `self/subcortex/providers/src/providers/cohere/` — the new provider leaf
- **Generated (not hand-edited):** `provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts` — regenerated via `pnpm --filter @nous/subcortex-providers run generate:providers`
- **Reference implementation:** the Anthropic provider leaf (documented in the "Anthropic Reference" doc) as the closest real example of a fully implemented, non-shared-protocol leaf
- **Docs consulted:**
  - https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
  - https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
  - https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference
  - https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist

### Important Note on Scope

The issue body itself flags that it was updated on 2026-06-18: the originally-referenced approach (hand-written `IModelProvider` files, e.g. `self/subcortex/providers/src/<vendor>-provider.ts`) is superseded and should not be used. The current integration target is the branch `feat/contributor-friendly-inference-provider-surface`, and work should be based on that branch rather than `main`, once I get to Phase II.

---

## Reproduction Process

_To be completed in Phase II._

---

## Solution Approach

_To be completed in Phase II._

---

## Testing Strategy

_To be completed in Phase III._

---

## Implementation Notes

_To be completed in Phase III._

---

## Pull Request

_To be completed in Phase IV._

---

## Learnings & Reflections

_To be completed at end of program._

---

## Resources Used

- https://github.com/orthogonalhq/nous-core/issues/313
- https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
- https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
- https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference
- https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist
- https://docs.nue.orthg.nl/docs/development/provider-adapters/anthropic-reference
