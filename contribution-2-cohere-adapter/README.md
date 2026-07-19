# Contribution 2: Adapter: Cohere Model Provider

**Contribution Number:** 2
**Student:** Geethanjali Nagaboina
**Issue:** https://github.com/orthogonalhq/nous-core/issues/313
**Status:** Phase II Complete — Assigned by maintainer

---

## Why I Chose This Issue

I chose this issue because it asks for a self-contained, well-documented unit of work — implementing the Cohere model provider as a certified "provider leaf" — rather than a change scattered across the codebase. The `nous-core` project maintains detailed contributor docs for this exact task (a quickstart, a provider-leaf anatomy reference, a schemas/ABI reference, and a testing checklist), which gives me a clear on-ramp even though I haven't worked in this codebase before. It's also a good stretch from my last contribution: that one was a Python test-only change, and this one is TypeScript and touches actual adapter/provider code, so I'll get exposure to a different language and a different kind of contribution (implementing an integration against a documented internal contract, not just writing tests). I'm interested in learning how a project structures a plugin-style adapter system (definition / adapter / factory contracts) so that adding a new vendor doesn't require touching shared code.

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

The issue body itself flags that it was updated on 2026-06-18: the originally-referenced approach (hand-written `IModelProvider` files, e.g. `self/subcortex/providers/src/<vendor>-provider.ts`) is superseded and should not be used. The current integration target is the branch `feat/contributor-friendly-inference-provider-surface`, and work is based on that branch rather than `main`.

**Maintainer Note:** Assigned by @atlamors following my initial comment expressing interest.

---

## Reproduction Process

### Environment Setup

Forked and cloned the repo, then added the upstream remote to access the integration branch (not on `main`):

```bash
git clone https://github.com/geethanjali-29/nous-core.git
cd nous-core
git remote add upstream https://github.com/orthogonalhq/nous-core.git
git fetch upstream
git checkout -b feat/contributor-friendly-inference-provider-surface upstream/feat/contributor-friendly-inference-provider-surface
```

**Challenges encountered and how I resolved them:**

1. **`corepack enable` failed with `EACCES: permission denied`**
   - *Problem:* Node was installed via a system installer, so `/usr/local/bin` and `/usr/local/lib/node_modules` are root-owned. `corepack enable` couldn't symlink `pnpm` into `/usr/local/bin`.
   - *Resolution:* Installed `nvm` via Homebrew and switched to an nvm-managed Node install in `$HOME`, avoiding root-owned directories entirely. `pnpm --version` then resolved correctly (10.6.2).

2. **`pnpm install` failed building `better-sqlite3` — `gyp ERR! build error`, `make failed with exit code 2`**
   - *Problem:* Was on Node v26.3.0 (Homebrew's latest), which is newer than what `better-sqlite3`'s prebuilt binaries currently support. This forced a from-source native build via `node-gyp`, which failed.
   - *Resolution:* The repo's `package.json` specifies `"engines": { "node": ">=22" }`. Installed Node 22 LTS via `nvm install 22 && nvm use 22` and re-ran `pnpm install`. The native build succeeded cleanly (`better-sqlite3` install script completed in ~1.3s).

3. **`pnpm.onlyBuiltDependencies` warning on every pnpm command**
   - *Problem:* pnpm 10.x printed `[WARN] The "pnpm" field in package.json is no longer read by pnpm...` on every invocation.
   - *Resolution:* Cosmetic only — the repo's own `package.json` uses a pnpm config key that's been relocated in pnpm 10. No action needed; noted here so it isn't mistaken for a real error during grading/review.

**Final working environment:** macOS 15 (Darwin 25.5.0), Node v22 (via nvm), pnpm 10.6.2.

### Steps to Reproduce (Missing Cohere Provider Leaf)

1. Fork and clone the repository, check out `feat/contributor-friendly-inference-provider-surface` per the issue's current integration target (see Environment Setup above).
2. Install dependencies: `pnpm install`
3. List existing provider leaves:
   ```bash
   find self/subcortex/providers/src/providers -maxdepth 1 -type d
   ```
   **Expected result:** A `cohere` directory present among the provider leaves
   **Actual result:** 18 vendor directories returned (`anthropic`, `openai`, `mistral`, `gemini`, `groq`, `xai`, `ollama`, `openrouter`, `perplexity`, `deepinfra`, `huggingface-tgi`, `llama-cpp`, `vllm`, `moonshot`, `qwen-code`, `codex-cli`, `github-copilot-cli`, `openclaw`) — no `cohere`
4. Confirm Cohere is absent from the generated catalog:
   ```bash
   grep -ri cohere self/subcortex/providers/src/provider-definitions.ts
   ```
   **Expected result:** A `cohere` entry in the generated definitions
   **Actual result:** No matches
5. Run the focused provider test suite as a pre-change baseline:
   ```bash
   pnpm --filter @nous/subcortex-providers exec vitest run src/__tests__/provider-codegen.test.ts src/__tests__/public-exports.test.ts src/__tests__/provider-definitions src/__tests__/adapter-resolver.test.ts src/__tests__/provider-pipeline-integration.test.ts --config vitest.config.ts
   ```
   **Result:** 54 passed, 2 failed (out of 56) — both failures are **pre-existing and unrelated to Cohere**, confirmed present on a clean checkout before any code changes:
   - `discovers certified provider leaves in deterministic vendor order` — expects `vllm` before `qwen-code`, got the reverse order; appears to be a filesystem directory-read ordering difference between macOS and the CI environment, not a logic bug.
   - `constructs providers from registry-derived definitions with env-var credentials` — throws `Mistral API key required — set MISTRAL_API_KEY or pass apiKey option`; looks like a test-environment stubbing issue unrelated to this issue's scope.

   These two failures establish my baseline: my eventual Cohere PR should not be expected to fix them, and I'll note that explicitly if they still show up in CI on my PR.

### Reproduction Evidence

- **Branch:** https://github.com/geethanjali-29/nous-core/tree/fix-issue-313
- **Issue confirmed:** No `cohere` directory under `self/subcortex/providers/src/providers/`, no `cohere` references in the generated `provider-definitions.ts`, confirming the gap described in the issue.

---

## Solution Approach

### Implementation Plan (UMPIRE)

**Understand:**
`nous-core` routes model requests through certified "provider leaves" — self-contained vendor folders under `self/subcortex/providers/src/providers/<vendor>/` that export a `providerDefinition`, `providerAdapter`, and `providerFactory`. No leaf exists for Cohere, so Cohere-hosted models cannot currently be selected or routed to. This is a pure gap in provider coverage, not a bug in existing code.

**Match:**
The Anthropic leaf (`src/providers/anthropic/`) is the closest structural template, since Cohere is a native API (not OpenAI Chat Completions-compatible), so it can't simply wrap `src/protocols/openai-api/**` the way `src/providers/openai/` does. Anthropic's `implementation.ts` shows the pattern for owning request execution directly.

**Plan:**
1. Create `self/subcortex/providers/src/providers/cohere/` with `definition.ts`, `adapter.ts`, `provider.ts`, `implementation.ts`, `index.ts`
2. In `definition.ts`, define `providerDefinition` as metadata only (no env reads/network calls), using a semantic `vendorKey: 'cohere'` and letting `wellKnownProviderId` be derived automatically
3. Declare `auth.purpose: 'api_key'`, `auth.required: true`, `auth.envVar` (e.g. `COHERE_API_KEY`), `auth.vaultKeyNamespace`, and `auth.header` per the Schemas ABI Reference
4. Implement request execution against Cohere's Chat API in `implementation.ts`, modeled on Anthropic's leaf
5. Wire up `providerAdapter` in `adapter.ts` and `providerFactory` in `provider.ts`
6. Regenerate catalogs: `pnpm --filter @nous/subcortex-providers run generate:providers`, then verify with `pnpm --filter @nous/subcortex-providers run check:generated`
7. Do not hand-edit the generated files (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`)

**Implement:**
Branch: https://github.com/geethanjali-29/nous-core/tree/fix-issue-313 (Phase III)

**Review:**
Will check `CONTRIBUTING.md` for commit message and PR conventions before opening a PR, and confirm the new leaf follows the same shape as existing leaves (matching Anthropic's file structure and export contract).

**Evaluate:**
Re-run the focused test suite used for the baseline above — expect the same 54 passing tests plus new coverage for the Cohere leaf, with the two pre-existing unrelated failures still isolated and called out. Run `typecheck` and `check:generated` to confirm the leaf integrates cleanly with codegen.

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
