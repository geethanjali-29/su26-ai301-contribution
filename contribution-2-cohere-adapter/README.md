# Contribution 2: Adapter: Cohere Model Provider

**Contribution Number:** 2
**Student:** Geethanjali Nagaboina
**Issue:** https://github.com/orthogonalhq/nous-core/issues/313
**Status:** Phase IV — PR submitted, maintainer review received (changes requested, revisions in progress)

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

   *(Update, Phase IV: both of these pre-existing failures were independently fixed upstream between my initial branch-off point and the PR's final rebase — see Testing Strategy below.)*

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
Branch: https://github.com/geethanjali-29/nous-core/tree/fix-issue-313

**Review:**
Checked `CONTRIBUTING.md` for commit message and PR conventions before opening the PR; confirmed the new leaf follows the same shape as existing leaves (matching Anthropic's file structure and export contract). Rebased onto the latest integration branch before opening the PR to pick up two other provider leaves (`azure-openai`, `dashscope`) that landed upstream while this work was in progress, and resolved the resulting merge conflicts across 5 test-fixture files plus 3 generated catalog files (regenerated fresh rather than hand-merged).

Received maintainer review from @atlamors five days after opening the PR (changes requested). See **Maintainer Feedback** and the revision plan below for the full response.

**Evaluate:**
Re-ran the focused test suite used for the baseline above, then the full suite, after rebasing onto the latest integration branch. Ran `typecheck` and `check:generated` to confirm the leaf integrates cleanly with codegen. See Testing Strategy below for final numbers as of the initial PR submission; this same phase will add a new evaluation pass once the requested changes land.

---

## Testing Strategy

**Full test suite result (Phase III, before final rebase):**
```bash
pnpm --filter @nous/subcortex-providers exec vitest run --config vitest.config.ts
```
Result: **524 passed, 2 failed, 4 skipped** (530 total). The 2 failures are the same pre-existing, unrelated failures documented in the Phase II baseline (`vllm`/`qwen-code` filesystem ordering, and a Mistral env-var stubbing issue) — confirmed unaffected by this change.

**Full test suite result (Phase IV, after rebasing onto latest integration branch pre-PR):**
```bash
pnpm --filter @nous/subcortex-providers exec vitest run --config vitest.config.ts
```
Result: **569 passed, 0 failed, 4 skipped** (573 total). Both previously pre-existing, unrelated failures are now gone — they were independently fixed upstream in commits that landed on the integration branch between my initial branch-off and the final rebase. The higher test count reflects two new provider leaves (`azure-openai`, `dashscope`) that also landed upstream during that window and are now exercised alongside Cohere in the shared pipeline-integration tests.

**What was tested (as of PR submission):**
- `pnpm --filter @nous/subcortex-providers run typecheck` — clean pass (includes `check:generated` as its first step, plus `tsc --build --force` and `tsc --noEmit`).
- `check:generated` — confirms `provider-definitions.ts`, `provider-adapters.ts`, and `provider-factories.ts` stay in sync with the leaf via `generate:providers`; verified both before and after the rebase (regenerated fresh after rebase rather than hand-merging the 3-way conflicts in these generated files).
- Full vitest suite, including `provider-pipeline-integration.test.ts`'s registry construction test, which successfully constructs and registers a `CohereProvider` instance end-to-end alongside all other vendors (including the newly-landed `azure-openai` and `dashscope`).

**What was not yet tested at PR submission (now addressed by maintainer review, see below):**
- No dedicated `cohere-provider.test.ts` or `cohere-adapter.test.ts` unit tests existed yet, unlike `anthropic-provider.test.ts` / `adapters/anthropic-adapter.test.ts`. The maintainer's review confirmed this gap and specifically requested these tests (see below) — the shared fixtures alone did not exercise Cohere's custom request/response, streaming, or error-handling paths, which is exactly where the maintainer found the behavioral issues listed above.
- Live/manual verification against the real Cohere API (no `COHERE_API_KEY` used) — the maintainer confirmed this is **not required** for this PR.

---

## Maintainer Feedback

@atlamors reviewed the PR and requested changes. Summary of the review, five days after submission:

**Overall assessment:** The provider leaf is organized clearly and the Cohere-specific protocol is correctly isolated. The maintainer noted that reviewing this PR also surfaced a gap in the Anthropic reference implementation itself (it makes native tool support look more complete than it currently is across the codebase) — the maintainer was explicit that fixing that reference is the project's responsibility, not something expected of this PR.

The maintainer ran the provider build, generated-file checks, the relevant tests, and did manual request/response reproductions against the Cohere paths. Four categories of requested changes came out of that:

**1. Native tool-use capability is not yet real and should not be advertised**
The leaf currently declares `nativeToolUse: true` in both the adapter and the provider definition, but a full tool round-trip fails: a tool-only assistant message fails input validation, and when assistant text is present, the assistant's `tool_calls` are stripped and the following tool result is discarded before the request reaches Cohere. Requested fix: set `nativeToolUse: false` in both places (parsing helpers can stay). The shared tool-call/tool-result bridge is being tracked separately in issue #390, and providers should only advertise the capability once that round trip works end to end.

**2. Several Cohere-specific request/response behaviors need correction**
- **Token limit not forwarded:** a configured `maxTokens` is currently dropped rather than being sent to Cohere as `max_tokens`; it should be forwarded when configured and omitted (letting Cohere use its default) when not.
- **Multi-block responses truncated:** `invoke()` currently reduces a Cohere response to its first content block before the adapter sees it, even though the adapter already knows how to join multiple text blocks. Given a response with two text blocks, the provider currently returns only the first; it should return the joined text.
- **System messages must stay as a single leading message:** the adapter currently emits the composed system prompt as one message and then appends context frames (e.g. short-term-memory summaries) as additional system messages. Cohere's API rejects any system turn that isn't first in the message list. These frames need to be folded into the single leading system message instead.
- **HTTP 498 misclassified:** Cohere documents 498 as an invalid-token response, but the leaf currently classifies it as `PROVIDER_UNAVAILABLE`. It should be classified alongside 401/403 as `PROVIDER_AUTH_FAILED` / `PRV-AUTH-FAILURE`, so the user is told to fix their credential rather than wait for the provider to recover.

**3. No endpoint for credential validation ("Save & Test")**
The provider definition has neither a `healthCheckEndpoint` nor a `modelListEndpoint`, so the shared Settings credential test rejects every Cohere key immediately without making a request. Requested fix: add `healthCheckEndpoint: '/v1/models'` (an authenticated endpoint Cohere exposes). `modelListing: false` is confirmed correct to leave as-is — it lets the credential test validate the key without claiming the shared model-discovery parser understands Cohere's response shape.

**4. Dedicated Cohere provider/adapter tests are needed**
The shared pipeline tests only confirm Cohere is registered in the catalog and resolvable through the factory — they don't exercise Cohere's custom request, response, streaming, or error-handling paths. The maintainer asked for focused, mocked, Cohere-specific tests covering: the `/v2/chat` URL and Bearer header; message serialization, streaming, and configured `max_tokens`; omission of `max_tokens` when unconfigured; multi-text-block non-streaming responses; `content-delta`/`message-end` streaming events and usage; auth-failure classification including HTTP 498; adapter formatting when a composed system prompt is accompanied by a system context frame; the disabled native-tool capability; and the `/v1/models` health-check metadata. A live `COHERE_API_KEY` test is explicitly **not** required.

**Explicitly out of scope for this PR (maintainer-owned, tracked separately):**
- The shared native-tool bridge — issue #390
- Model-discovery format support for Cohere's model-list shape, plus shared timeout/malformed-response lifecycle behavior — issue #413
- Generated catalog and central roster churn — issue #414

The maintainer closed by inviting pushback on any of the above if I disagreed, which I don't — all four items are concrete, reproducible, and scoped clearly to this leaf.

---

## Phase IV (continued): Addressing Review Feedback

Working through the maintainer's requested changes in order. Status per item:

| # | Requested change | Status |
|---|---|---|
| 1 | Set `nativeToolUse: false` in adapter + definition | In progress |
| 2a | Forward `config.maxTokens` → Cohere `max_tokens`; omit when unset | In progress |
| 2b | Stop truncating multi-block responses in `invoke()`; return joined text from adapter | In progress |
| 2c | Fold system context frames into a single leading system message | In progress |
| 2d | Reclassify HTTP 498 as `PROVIDER_AUTH_FAILED` / `PRV-AUTH-FAILURE` | In progress |
| 3 | Add `healthCheckEndpoint: '/v1/models'` to `providerDefinition` | In progress |
| 4 | Add dedicated `cohere-provider.test.ts` / `cohere-adapter.test.ts` covering the maintainer's checklist | In progress |

**Plan for this phase:**
1. Fix the four request/response behaviors in `implementation.ts` and `adapter.ts` (items 2a–2d) together, since all four touch the same request-building / response-parsing paths.
2. Flip `nativeToolUse` to `false` in `definition.ts` and `adapter.ts` (item 1), keeping the existing tool-call parsing helpers in place per the maintainer's note.
3. Add `healthCheckEndpoint: '/v1/models'` to `definition.ts` (item 3) — small, isolated change.
4. Write the new `cohere-provider.test.ts` and `cohere-adapter.test.ts` files against the maintainer's explicit checklist (item 4), using mocked fixtures only.
5. Regenerate catalogs (`generate:providers`) and re-run `check:generated` and `typecheck`, since `healthCheckEndpoint` and the `nativeToolUse` flip both touch generated output.
6. Re-run the full test suite and update the Testing Strategy section with new pass/fail numbers.
7. Push the revision commits to `fix-issue-313` and reply to @atlamors on the PR thread summarizing what changed against each numbered item.

**Not doing in this phase (per maintainer's explicit scoping):**
- Building the shared tool-call/tool-result bridge (#390)
- Implementing Cohere model-listing support (#413)
- Any shared timeout/malformed-response lifecycle work (#413)
- Touching the generated catalog/central-roster mechanism itself (#414)

---

## Implementation Notes

**What was built:** A complete Cohere provider leaf at `self/subcortex/providers/src/providers/cohere/` (`definition.ts`, `adapter.ts`, `provider.ts`, `implementation.ts`, `index.ts`), modeled structurally on the Anthropic leaf but adapted for Cohere's v2 Chat API (`POST /v2/chat`, `Authorization: Bearer` auth, native tool-calling via `tool_calls`, SSE streaming with `content-delta`/`message-end` event types).

**Key decisions:**
1. **`protocol: 'cohere-chat'` and `adapterKey: 'cohere'`** use the schema's `(string & {})` escape hatch rather than requiring an enum change, matching how other non-`chat-completions` protocols are declared.
2. **`capabilities.cacheControl` and `extendedThinking` are both `false`** — Cohere's Chat API has no equivalent to Anthropic's cache-control segments or extended-thinking blocks, so these were left honestly `false` rather than approximated.
3. **`modelListing: false`** — while Cohere does expose a models endpoint, I didn't confirm its response shape matches either `anthropic-models` or `openai-models` (the only two formats the schema's `ProviderModelListFormatSchema` currently supports), so I left model listing unimplemented rather than declaring an unverified format. *(Confirmed correct by maintainer review — see below.)*
4. **`nativeToolUse: true` at PR submission** — this was flagged by maintainer review as premature; the full tool round-trip does not yet work end to end, so this is being flipped to `false` in this revision pass while the shared bridge (#390) is built separately.
5. **Registered `COHERE_API_KEY` test fixture consistently** across `provider-pipeline-integration.test.ts` so the registry-construction integration test builds a real `CohereProvider` instance the same way it does for every other vendor.
6. **Kept the vendor-order arrays in true alphabetical order** during the final rebase's conflict resolution (`azure-openai` → `codex-cli` → `cohere` → `dashscope` → `deepinfra`...) rather than preserving either side's ordering verbatim, matching the convention used throughout the rest of the fixtures.

**Challenges encountered and how I resolved them:**
1. **Hardcoded vendor-key arrays in test fixtures.** Several tests (`provider-codegen.test.ts`, `provider-definitions.test.ts`, `provider-pipeline-integration.test.ts`, `adapter-resolver.test.ts`) hardcode the full list of vendor keys as an explicit array/union type rather than deriving it dynamically. Adding a new leaf requires updating each of these — a good adjacent-issue candidate to file upstream (dynamic derivation would prevent this class of edit entirely).
2. **A metadata-only definitions test reads provider source files by hardcoded relative path** (`provider-definitions.test.ts`'s `providerFiles` array), matched against the pattern `_PROVIDER_DEFINITION = {`. I initially guessed the wrong file for a sibling entry while doing a bulk find/replace and had to trace it back via `grep` on the actual export locations (`definition.ts` vs `implementation.ts` differ per provider depending on where each defines its named constant).
3. **A broad `sed` find/replace edited more than intended**, inserting a stray line into an unrelated test fixture object because the matched text pattern wasn't unique enough. Fixed by using more context-specific multi-line matches (`perl -0777` with surrounding-line anchors) for subsequent edits.
4. **Three-way merge conflicts during the final rebase.** Rebasing onto the latest integration branch surfaced conflicts in 5 files: 2 real test fixtures (`provider-definition-types.test.ts`, `provider-pipeline-integration.test.ts`) that needed both sides' additions merged (my `cohere` entries plus upstream's newly-landed `azure-openai`/`dashscope` entries), and 3 generated catalog files that should never be hand-merged. For the generated files, resolved conflicts by taking either side as a placeholder (`git checkout --ours`) and then running `generate:providers` fresh afterward, rather than manually reconciling the generated diff — consistent with the project's "don't hand-edit generated files" convention. For the real test fixtures, merged both sides by hand (or via a small Python script for the longer array literals) to preserve alphabetical ordering across all three new vendors.
5. **`git rebase --continue` opening vim mid-terminal-restart.** After closing the terminal mid-rebase, resumed cleanly since rebase state persists on disk in `.git`; used `GIT_EDITOR=true git rebase --continue` to skip the interactive commit-message editor and avoid repeated vim sessions during a multi-commit rebase with several conflict-resolution rounds.

### Files Changed (as of PR submission)
```
self/subcortex/providers/
└── src/
    ├── providers/
    │   └── cohere/                          ← new leaf
    │       ├── adapter.ts
    │       ├── definition.ts
    │       ├── implementation.ts
    │       ├── index.ts
    │       └── provider.ts
    ├── index.ts                             ← added CohereProvider export
    ├── provider-definitions.ts               ← regenerated
    ├── provider-adapters.ts                  ← regenerated
    ├── provider-factories.ts                 ← regenerated
    └── __tests__/
        ├── provider-codegen.test.ts                ← added 'cohere' to vendor order fixture
        ├── adapter-resolver.test.ts                ← added 'cohere' to adapter module order
        ├── provider-definitions/
        │   ├── provider-definition-types.test.ts   ← added 'cohere' to vendor-key unions
        │   └── provider-definitions.test.ts        ← added cohere roster entry + expectedDefinitions + file-list entry
        └── provider-pipeline-integration.test.ts   ← added 'cohere', COHERE_API_KEY fixture, CohereProvider import/map entry
```

**Additional files expected in this revision pass:**
```
self/subcortex/providers/
└── src/
    ├── providers/
    │   └── cohere/
    │       ├── adapter.ts                    ← nativeToolUse: false; system-frame folding; response join fix
    │       ├── definition.ts                 ← nativeToolUse: false; healthCheckEndpoint added
    │       └── implementation.ts             ← max_tokens forwarding; 498 auth classification; response block handling
    ├── provider-definitions.ts               ← regenerated (healthCheckEndpoint, nativeToolUse)
    ├── provider-adapters.ts                  ← regenerated (nativeToolUse)
    └── __tests__/
        └── providers/cohere/                 ← new: cohere-provider.test.ts, cohere-adapter.test.ts
```

### Code Changes
- **Branch:** https://github.com/geethanjali-29/nous-core/tree/fix-issue-313
- **Base branch:** `feat/contributor-friendly-inference-provider-surface` (per issue's 2026-06-18 scope update)
- **Key commits (through PR submission):**
  - `feat(providers): add Cohere provider leaf` — the 5 new leaf files + regenerated catalogs + initial test fixture updates
  - `test(providers): register cohere in vendor-key fixtures and definition-file roster` — follow-up fixture corrections
  - `test(providers): register cohere in provider-codegen vendor order fixture`
  - `chore(providers): regenerate catalogs after rebase onto azure-openai/dashscope` — post-rebase catalog regeneration
- **Test result at PR submission:** 569 passed / 0 failed / 4 skipped (573 total)
- **Revision commits (in progress):** to be added as each requested change lands

---

## Pull Request

- **PR Link:** https://github.com/orthogonalhq/nous-core/pull/430
- **PR Description:** Adds a certified Cohere provider leaf (definition/adapter/factory + Chat API implementation) on `feat/contributor-friendly-inference-provider-surface`, closing #313. Rebased cleanly alongside the newly-merged `azure-openai` and `dashscope` leaves; full test suite passes at 569/0/4 (passed/failed/skipped).
- **Base branch:** `orthogonalhq:feat/contributor-friendly-inference-provider-surface` ← `geethanjali-29:fix-issue-313` (4 commits at submission)
- **Maintainer Feedback:** Changes requested by @atlamors — see **Maintainer Feedback** section above for full detail.
- **Status:** Addressing requested changes (in progress)

---

## Learnings & Reflections

_To be completed at end of program._

_Interim note: this review is a useful example of a maintainer distinguishing between what a contributor should fix (leaf-specific request/response correctness, missing tests, an over-eager capability flag) and what the maintainer owns (a systemic gap in the reference implementation and shared-surface work tracked in separate issues). Worth carrying forward: advertise a capability only once it's been exercised end-to-end, not just once parsing looks right._

---

## Resources Used

- https://github.com/orthogonalhq/nous-core/issues/313
- https://github.com/orthogonalhq/nous-core/pull/430
- https://docs.nue.orthg.nl/docs/development/provider-adapters/quickstart
- https://docs.nue.orthg.nl/docs/development/provider-adapters/provider-leaf-anatomy
- https://docs.nue.orthg.nl/docs/development/provider-adapters/schemas-abi-reference
- https://docs.nue.orthg.nl/docs/development/provider-adapters/testing-checklist
- https://docs.nue.orthg.nl/docs/development/provider-adapters/anthropic-reference
- https://docs.cohere.com/v2/reference/errors
- https://docs.cohere.com/v2/reference/chat
- https://docs.cohere.com/reference/list-models
