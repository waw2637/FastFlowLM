# Model-template boundary assessment

## Verdict

This is a medium-sized refactor, not a rewrite. The template engine and model-file
loading already exist in `AutoModel`; the broken boundary is the policy wrapped
around them. A safe implementation is roughly three reviewable changes plus a
template conformance suite. It should not be combined with inference-kernel work.

## Current boundary leaks

1. `AutoModel::_shared_setup_tokenizer()` owns template discovery, file precedence,
   parsing, and process-fatal error handling.
2. Every concrete model rebuilds `minja::chat_template_inputs`. Most copies differ
   only in whether tools and a few context flags are forwarded.
3. Request semantics leak into model classes. Tool eligibility, thinking flags,
   generation-prompt behavior, and Minja compatibility options are selected inside
   individual backends rather than by a declared template policy.
4. Template rendering is coupled to insertion/tokenization. There is no small API
   that can render and validate a template without loading weights or an NPU engine.
5. Template type is used by prompt caching, while actual rendering behavior is
   independently encoded in model subclasses. Those two descriptions can drift.
6. Missing or invalid template assets terminate the process with `exit(1)` instead
   of returning a model-load error at the server boundary.

## Proposed boundary

Introduce a model-independent `ChatTemplateRenderer` owned by `AutoModel`:

```text
REST/CLI message contract
        |
        v
TemplateRequest { messages, tools, add_generation_prompt, context }
        |
        v
ChatTemplateRenderer { loaded template + Minja options }
        |
        v
RenderedPrompt
        |
        v
Tokenizer -> model-specific token adjustments -> inference engine
```

Concrete models should declare only policy/capabilities, for example:

- supports tools;
- default thinking mode and extra context;
- Minja compatibility options;
- any explicit post-render token adjustment.

They should not assemble generic `chat_template_inputs` or decide asset-loading
precedence themselves.

## Delivery slices

### 1. Characterization tests (low risk)

Create CPU-only tests that load template text directly and compare rendered output
for every registered model family. Cover system/user/assistant messages, tool calls,
tool results, empty tools, thinking on/off, malformed templates, and precedence of
`chat_template.jinja` over `tokenizer_config.json`.

### 2. Extract renderer and typed errors (moderate risk)

Move template discovery/compilation/rendering behind `ChatTemplateRenderer`. Return
typed errors instead of calling `exit(1)`. Keep existing model overrides as adapters
so output stays byte-for-byte stable.

### 3. Replace per-model boilerplate with policy (moderate risk)

Add a `ChatTemplatePolicy` value to each model family and make the base implementation
of `apply_chat_template()` concrete. Delete overrides only after their characterization
tests pass. Keep genuinely model-specific post-render token behavior outside the
renderer.

### 4. Reconcile prompt-cache identity (moderate risk)

Derive cache compatibility from renderer/template identity plus relevant policy,
rather than a separately maintained `chat_template_type_t` default. Changing this
without tests risks reusing KV state across prompts rendered under different flags.

## Effort and risk

- Boundary extraction: about 1-2 focused engineering days.
- Characterization matrix and fixtures: about 1-2 days.
- Migrating all model families and removing duplicates: about 2-3 days.
- Hardware/OpenAI compatibility validation: environment-dependent.

The main difficulty is not C++ mechanics. It is proving that rendered prompts remain
byte-for-byte compatible across model families, especially tool and thinking modes.
With tests first, this is tractable; without them, a broad cleanup can silently change
token sequences and model behavior.

## Non-goals

- Do not put image/audio message normalization into the renderer. That belongs at the
  API-to-domain boundary before template rendering.
- Do not put tokenizer or inference-engine behavior into template policy.
- Do not make model-list metadata execute arbitrary template policy.
- Do not mix this refactor with the streaming control-flow fix.
