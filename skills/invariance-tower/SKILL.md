---
name: invariance-tower
description: "Harness invariance toolkit: 8-gate dashboard (type audit + 7 ratchets) for apex+xli+qwen+cli-ops. Run BEFORE bumping SDKs / changing wire types / adding settings knobs / absorbing upstream. Surfaces drift that would otherwise silently regress production (sortie-24/26 worked examples)."
---

# Invariance Tower — Verification Toolkit

## TL;DR — the daily move

Before ANY of: SDK bump, wire-type change, settings-knob addition, upstream
absorption, converter refactor — run:

```
invariance_check(action="status")    # 8-gate dashboard, <1s
```

Any 🚨 alarm = look in `cli-ops/sortie-board/apex-harness-invariant/sorties/`
for the named sortie's doctrine doc. The campaign that built each gate is
the campaign that documents how to read its output.

For per-gate detail with top-N gaps:

```
invariance_check(action="ratchet", name="<gate>", top=5)
```

For LIVE structural re-verification (when refactoring types):

```
invariance_check(action="audit")     # ~5s, runs cgm-invariant-audit against indexes
invariance_check(action="drift")     # audit + diff vs last snapshot
```

## What this verifies

The invariance tower ensures that the 3 layers of cross-wire documentation
in `cli-ops/refs/api-references/` remain mechanically grounded in the actual
source code across both harness spokes (apex-ontap + xli).

```
L3  settings-invariants.md     S ──π_wire──→ S_wire ──π_model──→ S_effective(m)
L2  harness-invariants.md      IR^(n) ──τ──→ IR^(n+1)
L1  cross-wire.md              IR ──ε──→ Wire ──δ──→ IR
```

## Gate 1: Type Audit (`invariance_check action=audit`)

**Script:** `~/Projects/code-graph-mcp/scripts/cgm-invariant-audit`

Verifies 22 invariant-critical types exist at their expected file paths
across both spokes using the code-graph-mcp `show` command.

### Three layers × two spokes = 22 checkpoints

| Layer | Apex types | XLI types |
|---|---|---|
| **L3 settings** | WireCapabilities, resolveSamplingParams, ResolvedSamplingParams, buildSamplingParameters, buildRequest, resolveEffort, ModelConfig, ModelCapabilities | ConfiguredModelProvider, AnthropicMessagesProvider, GeminiModelProvider |
| **L1 wire** | AnthropicContentConverter, convertGeminiContentsToResponsesInput, convertResponsesEventToGemini, ResponsesPipeline | build_messages_request, build_responses_request |
| **L2 harness** | AnthropicContentGenerator, OpenAIResponsesContentGenerator | AnthropicMessagesProvider, build_messages_request, build_responses_request, ConfiguredModelProvider |

### Interpreting results

- `✅` = type found at expected file:line
- `❌ MISSING` = type not found → it was renamed or deleted; update the manifest
- `⚠️ MOVED` = type found but at a different file → file was moved; update the manifest

### Flags

```bash
cgm-invariant-audit                    # all 3 layers, both spokes
cgm-invariant-audit --layer settings   # L3 only
cgm-invariant-audit --layer wire       # L1 only
cgm-invariant-audit --drift            # compare against previous snapshot
cgm-invariant-audit --json             # machine-readable output
```

### Snapshots

Saved to `~/Projects/cli-ops/refs/api-references/.audit-snapshots/<timestamp>.txt`.
`--drift` compares against the most recent snapshot.

### When to run

- After every upstream merge (openai/codex, google-gemini/gemini-cli)
- After any sortie that touches wire converter files
- After refactors that rename or move types

## Gates 2–8: Ratchets (`invariance_check action=ratchet`)

**Location:** `~/Projects/cli-ops/refs/api-references/`
**Generate:** `make ratchets` (or `make -C ~/Projects/cli-ops/refs/api-references ratchets`)

### Ratchet 1: SDK Type Coverage (`name=sdk-types`)

**Script:** `scripts/gen-sdk-type-ratchet.py`
**Source:** `refs/anthropic/spec/src/` (anthropic-sdk-typescript submodule)
**Cross-ref:** `cross-wire.md` concept column

Extracts every exported TypeScript type/interface from the Anthropic SDK
and checks which appear in cross-wire.md. Growth in gap count = new Anthropic
feature that neither harness maps yet.

### Ratchet 2: litellm Transform Coverage (`name=litellm-params`)

**Script:** `scripts/gen-litellm-transform-ratchet.py`
**Source:** `upstream-infrastructure/litellm/litellm/llms/anthropic/chat/transformation.py`
**Cross-ref:** `settings-invariants.md` §3 manifest

AST-walks litellm's `map_openai_params()` method — extracts the 18 params
it handles for Anthropic. Cross-references against settings-invariants.md.
Gaps = params litellm passes through that our settings doc doesn't cover.
Asymmetric = params we doc as "dropped" that litellm correctly doesn't handle.

### Ratchet 3: Proxy Model Coverage (`name=proxy-models`)

**Script:** `scripts/gen-proxy-model-ratchet.py`
**Source:** `upstream-infrastructure/llm-proxy/app/api/config_seclab*.yaml`
**Cross-ref:** `~/Projects/apex-ontap/canonical/live/system-settings.json`

Every model alias the proxy serves vs what the apex catalog knows. Gaps =
models users can route to but apex doesn't know their capabilities
(reasoning, effort, caching behavior undefined).

### Ratchet 4: Test Title Coverage (`name=test-titles`)

**Script:** `scripts/gen-test-title-ratchet.py`
**Source:** `harness-invariants-autogen.md` (388 H-INV rows)
**Cross-ref:** actual test files on disk

Verifies every H-INV row's source test file still exists and still contains
the test title string. MISSING = test file deleted (invariant chain broken).
DRIFTED = test title changed (H-INV row stale).

### Ratchet 5: Cursor Pin Coverage (`name=cursor-pins`)

**Script:** `~/Projects/cli-ops/refs/api-references/scripts/gen-cursor-pin-ratchet.py`
**Sources:** apex spoke + cli-ops/refs/cursor recon submodules
**Report:** `cursor-pin-coverage.md`

Surfaces every pin that defines what Cursor SDK we're reading against AND
what apex actually consumes (npm pin / installed / recon-side). Drift in any
row is a re-recon trigger. Built in sortie-19.

### Ratchet 6: Plumbing Integrity (`name=plumbing-integrity`)

**Script:** `~/Projects/cli-ops/refs/api-references/scripts/gen-plumbing-integrity-ratchet.py`
**Sources:** apex spoke (Config, contentGenerator, proxyRouter, AnthropicContentConverter)
   + `refs/anthropic/spec/src/resources/messages/messages.ts`
**Report:** `plumbing-integrity-ratchet.{md,json}` (JSON-first; status reader uses JSON)

The biggest sortie outcome of the campaign. Three sub-checks:

1. **Knob plumbing** — every spec wire-shape field (CacheControlEphemeral, ThinkingConfigParam, OutputConfig) is classified user-settable / hardcoded / ignored AND verified to emit spec-correct on every applicable site.
2. **Constructor contract** — every wire-converter constructor call-site arg count matches the constructor's signature. Catches the sortie-24 `TS2554` bug class (extras silently dropped at runtime).
3. **Layer-4b coverage** (sortie-27 rule #7) — every `Config.get<X>()` getter MUST appear in `buildProxyConfig` as `<knob>: pickerGenerationConfig?.<knob> ?? config?.get<X>?.()`, OR be in `LAYER_4B_PROXY_NA_ALLOWLIST` with explicit rationale, OR pattern-match an auto-allowlist regex.

Status surfaces `layer_4b_uncovered` count — the headline number. At sortie-27 close-out: 0.

Built in sortie-25 (Phase 1 cache_control), sortie-25 Phase 2 (ThinkingConfigParam + OutputConfig), sortie-27 Phase 2 (Layer-4b).

### Ratchet 7: Models Recon — Anthropic (`name=models-recon-anthropic`)

**Script:** `~/Projects/cli-ops/refs/api-references/scripts/gen-model-page-anthropic.py`
**Sources:** litellm price JSON + apex anthropicModelCapabilities.ts + llm-proxy config_seclab*.yaml + Anthropic SDK spec
**Report:** `anthropic/models-recon.{md,json}` (JSON-first; status reader uses JSON)

Per-model reconciliation across three sources. Surfaces:
- ✅ all-sources (model present + agrees across litellm/apex/proxy)
- ⚠️ apex-only (registry knows, litellm missing)
- ❌ no-apex (litellm has it, apex registry missing)
- 🚫 proxy-not-served (apex+litellm know, proxy yaml doesn't)
- 📐 output-mismatch (litellm vs apex max_output_tokens)
- 🔁 alias-drift (dot↔dash convention mismatch)
- 🧊 apex emits 1h cache but litellm lacks 1h tier
- ⏳ deprecation_date set

PLUS: spec-conformance section — every consumer cache_control literal apex emits is classified vs the SDK's accepted enum. Surfaces the sortie-24 `'ephemeral_1h'` bug class.

Built in sortie-22 (POC), sortie-23 (spec-rooted hardening).

## Gate 9: `make verify` (existing pipeline)

**Location:** `~/Projects/cli-ops/refs/api-references/Makefile`

Regenerates all 28 .md files from 26 git submodules (3 SDK specs, litellm AST,
llm-proxy AST, harness test extraction) and `git diff --exit-code`. Any change
= spec submodule or generator drifted.

```bash
make -C ~/Projects/cli-ops/refs/api-references verify     # strict
make -C ~/Projects/cli-ops/refs/api-references verify-xli  # XLI-only gate
make -C ~/Projects/cli-ops/refs/api-references coverage    # advisory gap finder
```

## Full verification cascade

```bash
# Quick health check (reads existing reports, no regeneration)
invariance_check(action="status")

# Deep verification (runs scripts, ~5s + ~10s for ratchets)
invariance_check(action="audit")          # 22 types verified
invariance_check(action="ratchet")        # 7 ratchets, gaps only
invariance_check(action="ratchet", name="plumbing-integrity", top=5)  # Layer-4b drift
invariance_check(action="ratchet", name="models-recon-anthropic", top=5)  # model drift

# After upstream merge (full pipeline, ~2min)
make -C ~/Projects/cli-ops/refs/api-references verify
cgm-invariant-audit --drift
make -C ~/Projects/cli-ops/refs/api-references ratchets
python3 ~/Projects/cli-ops/refs/api-references/scripts/gen-plumbing-integrity-ratchet.py
python3 ~/Projects/cli-ops/refs/api-references/scripts/gen-model-page-anthropic.py
```

## Harness-work routing — when to use which gate

| Operator move | Gate to check first |
|---|---|
| Bumping `@anthropic-ai/sdk` (or any provider SDK) | `sdk_types` + `plumbing_integrity` (catches discriminated-union additions + new fields apex must handle) |
| Adding a `settings.json` knob | `plumbing_integrity` (Layer-4b alarm if knob bypasses `buildProxyConfig`) — also see `.agents/skills/apex-setting-threading/SKILL.md` for the 6-layer template |
| Adding a new model alias to the proxy yaml | `proxy_models` + `models_recon_anthropic` |
| Changing the apex `AnthropicContentConverter` constructor | `plumbing_integrity.constructor_contract` (catches arg-count drift = sortie-24 bug class) |
| Adding a new Cursor SDK version pin | `cursor_pins` |
| Renaming or moving a type used in cross-wire docs | `type_audit` (LIVE — `action=audit`) |
| Refactoring upstream-shaped files (after gemini-cli absorption) | `type_audit` then `plumbing_integrity.layer_4b_contract` (catches new `Config.get<X>()` getters that bypass the proxy fleet) |
| Adding a new wire (responses / messages / something new) | `litellm_params` + `plumbing_integrity` + `models_recon_<provider>` (file a sortie if the gates don't exist for the new provider yet) |

## Key file paths

| What | Path |
|---|---|
| Type audit script | `~/Projects/code-graph-mcp/scripts/cgm-invariant-audit` |
| Init script | `~/Projects/code-graph-mcp/scripts/cgm-init` |
| Ratchet scripts | `~/Projects/cli-ops/refs/api-references/scripts/gen-*-ratchet.py` |
| SDK type ratchet | `~/Projects/cli-ops/refs/api-references/scripts/gen-sdk-type-ratchet.py` |
| Make pipeline | `~/Projects/cli-ops/refs/api-references/Makefile` |
| Audit snapshots | `~/Projects/cli-ops/refs/api-references/.audit-snapshots/` |
| Ratchet reports | `~/Projects/cli-ops/refs/api-references/{sdk-type-coverage,litellm-transform-ratchet,proxy-model-coverage,test-title-coverage}.md` |
| settings-invariants.md | `~/Projects/cli-ops/refs/api-references/settings-invariants.md` |
| cross-wire.md | `~/Projects/cli-ops/refs/api-references/cross-wire.md` |
| harness-invariants.md | `~/Projects/cli-ops/refs/api-references/harness-invariants-curated.md` |
| Design doc | `~/Projects/code-graph-mcp/docs/INVARIANCE-TOOLKIT-DESIGN.md` |

## Known issues

- **sdk_types gate** ✅ fixed in cli-ops sortie-15 (`71f4a1c`) — was 0/0, now 15/160.
- **proxy_models gate** ✅ fixed in cli-ops sortie-16 (`71f4a1c`) — was 32/29, now 32/25 (alias canonicalization recognized).
- **litellm_params gate** ✅ fixed in cli-ops sortie-17 (`71f4a1c`) — was 5/13, now 5/4 (wire-internal vs settings-candidate split).
- **Settings manifest generator** ✅ landed in cli-ops sortie-19 (`gen-settings-manifest.py` + interactive visualizer + cross-harness equivalence table).
- **Plumbing-integrity ratchet** ✅ landed in cli-ops sortie-25 + sortie-27 (Layer-4b coverage at 100% triage; 0 uncovered).
- **Anthropic model page** ✅ landed in cli-ops sortie-22 + sortie-23 (spec-rooted, JSON-first; surfaces 244 models across 3 sources).
- **models_recon_anthropic** currently reports 1 historical spec-violation: `cache_control.type: 'ephemeral_1h'` in a type-union DECL in `converter.ts:716` (not an emission). Cleanup tracked but non-blocking.
