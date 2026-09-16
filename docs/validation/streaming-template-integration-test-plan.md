# Streaming and model-template integration test plan

## Purpose

Validate the independently branched changes against the same FastFlowLM baseline:

- `fix/streaming-option`: restore generation in the Ollama-compatible streaming chat path;
- `cherry-pick/atomic-germ-template-fix`: preserve Gemma 4 tool results, normalize
  nullable/list schema types, and share the tool-call parser;
- `refactor/model-template-boundary`: proposed boundary for a later template-policy
  extraction (assessment only; no runtime behavior change).

Run the behavioral matrix first on unmodified upstream `main`, then on each fix branch,
then on a temporary local integration merge. Do not merge the implementation branches
just to run the combined matrix.

## Test environments

Use one Windows and one Linux machine supported by the project, with an AMD Ryzen AI
NPU and the normal model assets. Record:

- OS/build preset, CPU/NPU, driver/XRT or HRX version;
- exact FastFlowLM base SHA and tested branch SHA;
- model tag and model-asset revision;
- API request and complete response/event transcript.

Minimum models:

- one plain text model (Qwen family);
- Gemma 4 E2B or E4B;
- Gemma 4 12B;
- one thinking-enabled/tool-enabled model other than Gemma 4, as a regression control.

## Gate 0: build and CPU-only tests

1. Configure and compile the normal server target on Windows and Linux.
2. Build and run `src/test/gemma4_tool_parser/test.cpp` without NPU/model assets.
3. Run the repository's existing CTest suite and record skipped tests.
4. Run formatting/static-analysis checks used by CI.
5. Confirm each fix branch contains exactly its intended commit(s) above the recorded
   upstream base.

Expected: no compiler warnings introduced by the changes; parser checks pass; branch
diffs do not contain branding/path changes from OpenFlowLM.

## Gate 1: streaming option contract

Exercise `/api/chat` for every combination below:

| Case | `stream` value | Expected transport/result |
|---|---:|---|
| omitted | default | One non-streaming JSON response |
| explicit false | false | One non-streaming JSON response |
| explicit true | true | One or more content events, then exactly one final event |

For all three, use the same deterministic sampler settings and prompt. Verify:

- streaming content concatenates to the non-streaming content;
- the prompt is inserted once (prompt token count does not double);
- generated-token count is nonzero for a nonempty answer;
- `done`, `done_reason`, durations, and model name are present and coherent;
- server context is cleared after success;
- a second identical request succeeds and does not inherit unintended state;
- cancellation stops generation and permits the next request;
- max-context and malformed-message failures return once and leave the server usable;
- UTF-8 content split across stream-buffer writes is reconstructed correctly.

Also run `/api/generate` and `/v1/chat/completions` in stream/non-stream modes as
regression controls. The change must not alter their event vocabulary or finalization.

## Gate 2: Gemma 4 tool/template patch

Run the parser unit suite first, then endpoint tests on E2B/E4B and 12B.

### Tool-result continuation

Send a conversation containing a tool result whose identifying value occurs near the
end of a payload longer than the former truncation boundary. Ask the model to repeat
that value. Verify the rendered prompt contains the complete tool result and the model
can use the tail value.

Repeat with:

- short and long text results;
- JSON object/array results;
- newlines, quotes, backslashes, and UTF-8;
- multiple sequential tool calls/results;
- a tool result immediately followed by an assistant continuation.

### Schema normalization

Bind tools whose parameter schemas use:

- `type: ["string", "null"]`;
- `type: ["null", "integer"]`;
- a single string type;
- nested object/array properties;
- omitted type with nullable metadata.

Expected: supported type arrays render without a Minja exception, preserve the first
non-null type, and set nullable semantics. Invalid/ambiguous schemas must fail clearly
rather than silently changing the declared type.

### Tool-call parsing

Cover relaxed Gemma syntax for bare keys, marker-quoted strings, ordinary JSON strings,
nested objects/arrays, booleans, nulls, numbers, Windows paths, escapes, and multiple
tool calls. Verify stream and non-stream responses produce equivalent tool name and
argument JSON. Verify IDs are unique when two calls are emitted in one second.

Cover the generic `tool_call` envelope and verify the real nested tool name is returned.

## Gate 3: template boundary characterization

Before runtime refactoring begins, create golden rendered-prompt fixtures for every
registered model family. Each fixture must call template rendering without loading
weights or an NPU engine and cover:

- system/user/assistant roles;
- empty and populated tools;
- tool call and tool result messages;
- thinking enabled/disabled where supported;
- `add_generation_prompt` behavior;
- `chat_template.jinja` versus embedded tokenizer-config precedence;
- missing, unreadable, syntactically invalid, and runtime-invalid templates;
- multimodal message normalization before the renderer boundary.

Capture both rendered bytes and token IDs. A refactor passes only if intentional
fixture changes are reviewed model by model. Error cases must return typed model-load
or request errors and must not terminate the server process.

## Gate 4: combined integration run

Create a disposable local branch from the recorded upstream SHA and merge the streaming
and Atomic-Germ branches without modifying either source branch. Run Gates 0-3 plus:

1. A streaming Gemma 4 tool call.
2. A streamed tool result continuation with a long tail value.
3. Two tool calls emitted rapidly, checking unique IDs and ordering.
4. Cancellation during tool-call output followed by a clean request.
5. Prompt-cache reuse across identical template policy, followed by a request that
   changes tools or thinking policy and therefore must not reuse incompatible state.

## Acceptance criteria

- All CPU-only and existing project tests pass.
- Streaming chat generates tokens, inserts the prompt once, and finalizes once.
- Stream/non-stream semantic output matches under deterministic sampling.
- Gemma 4 long tool results and supported schema type arrays work on E2B/E4B and 12B.
- Stream/non-stream tool-call parsing is equivalent and call IDs are unique.
- No tested malformed input crashes or terminates the server.
- Golden template output/token IDs remain unchanged unless a separately reviewed fix
  explicitly updates them.
- Hardware results and raw API transcripts are attached to the eventual pull requests.
