# Contribution 2: feat: add should_summarize_callback param to LLMContextSummarizer

**Contribution Number:** 2 

**Student:** Shiva Acharya 

**Issue:** https://github.com/pipecat-ai/pipecat/issues/4795 

**Status:** Phase III In Progress

---

## Why I Chose This Issue

I chose issue #4795, which adds a should_summarize_callback parameter to LLMContextSummarizer in pipecat-ai/pipecat, as my next contribution after wrapping up PR work on #4212 (Langfuse tracing for STT/TTS spans). What drew me to it is that it sits squarely in the conversation-management layer of a voice AI pipeline — deciding when and how context gets summarized directly affects response latency and quality in real-time agents, which connects to the production concerns I dealt with using ElevenLabs TTS, OpenAI LLM streaming, and Deepgram STT at PlaceOrder AI. It's also a clean, additive feature request rather than a bugfix requiring root-cause spelunking, which makes it a good pace-setter after the more involved async/tracing work in #4212.

I'm hoping to use this issue to build depth in a part of the pipecat codebase I haven't touched yet — the context-summarization logic — and to practice designing a public API surface (a callback parameter) that other contributors and users will build on, rather than just patching internal behavior. Since the issue is well-specified but leaves the implementation details open, I also want to get more comfortable proposing a concrete design in a PR description and inviting maintainer feedback, rather than working from a fix that's already fully scoped out for me.

---

## Understanding the Issue

### Problem Description
`LLMContextSummarizer._should_summarize()` is invoked on every single `LLMFullResponseStartFrame` — i.e., on every assistant turn — and unconditionally runs `LLMContextSummarizationUtil.estimate_context_tokens()` to check the configured thresholds, even when the conversation is nowhere near `max_context_tokens` or `max_unsummarized_messages`. That's wasted work on every turn for the common case. On top of the overhead, there's no supported way to override this trigger logic: the only path today is subclassing `LLMContextSummarizer` and overriding `_should_summarize()`, which works only because the method happens to use a single underscore rather than being a truly private/name-mangled method.

### Expected Behavior

`LLMContextSummarizer` should accept an optional `should_summarize_callback` parameter. When provided, it replaces the built-in threshold-checking logic entirely — the callback becomes the sole authority on whether to summarize, and (per the proposal) the framework should be able to skip the token-estimation work when a callback is supplied and it's cheap to determine "not yet" without estimating tokens. When no callback is provided, behavior stays exactly as it is today (thresholds checked, token estimation performed as needed).

### Current Behavior
There is no `should_summarize_callback` parameter. The only way to customize trigger behavior is to subclass `LLMContextSummarizer` and override the private `_should_summarize()` method — fragile because it's not a documented extension point, could silently break on internal changes, and (critically) there's no supported way to get a custom subclass injected into `LLMAssistantAggregator` in the first place, since the aggregator constructs its own `LLMContextSummarizer` instance internally.

### Affected Components

+ `LLMContextSummarizer` and its `_should_summarize()` method (the trigger-check entry point, called on every `LLMFullResponseStartFrame`)
+ `LLMContextSummarizationUtil.estimate_context_tokens()` (the token-estimation work we want to be able to skip)
+ `LLMAssistantAggregator` / `LLMAssistantAggregatorParams` — needs a way to pass `should_summarize_callback` through to the `LLMContextSummarizer` it constructs, since there's currently no injection point for a custom summarizer instance
+ Likely `LLMAutoContextSummarizationConfig`, as the natural place to carry the new parameter alongside the existing thresholds — though this is one of the open design questions (see below)

---

## Reproduction Process

### Environment Setup

+ Cloned the repo and set up the dev environment with `uv` per `CONTRIBUTING.md`
+ Installed the `openai` extra so I could exercise `LLMContextAggregatorPair` with `enable_auto_context_summarization=True` end-to-end against a real (or mocked) `OpenAILLMService`
+ Since this is a "missing capability / avoidable overhead" issue rather than a crash, reproduction here means demonstrating the two claims in the issue: (1) `_should_summarize()` runs and estimates tokens on every turn regardless of how far the thresholds are, and (2) there's no supported hook to change that

### Steps to Reproduce

1. Clone `pipecat-ai/pipecat` and install with the openai extra so `OpenAILLMService` (or a mock) is available.
2. Build an `LLMContext` and create context aggregators via `LLMContextAggregatorPair`, passing `LLMAssistantAggregatorParams(enable_auto_context_summarization=True, auto_context_summarization_config=LLMAutoContextSummarizationConfig(max_context_tokens=50000, max_unsummarized_messages=200))` — thresholds set deliberately high so summarization should never trigger in a short test run.
3. Patch or wrap `LLMContextSummarizationUtil.estimate_context_tokens` with a counter (e.g., `unittest.mock.patch` with `wraps=` the original function) before running the pipeline.
4. Run a short scripted conversation through the pipeline — enough to produce 5-10 `LLMFullResponseStartFrames` (i.e., 5-10 assistant turns).
5. Observed result: the counter shows `estimate_context_tokens` was called once per assistant turn (5-10 times), even though neither threshold was remotely close to being met. Inspecting `LLMAssistantAggregatorParams` and `LLMContextSummarizer.__init__` further confirms there is no parameter anywhere in that chain to skip or override this check short of subclassing `LLMContextSummarizer` and overriding the private `_should_summarize()`.

### Reproduction Evidence

- **My findings:**
Confirmed the per-turn overhead is real and unconditional, and confirmed (by reading the current constructor signatures) that there is no parameter anywhere in the aggregator → summarizer chain to override or skip the check without subclassing a private method
---

## Solution Approach

### Analysis

The root cause is twofold: (1) the trigger check is unconditional — it always pays the token-estimation cost even when a cheap early-exit would do — and (2) the trigger logic is not a documented extension point, so the only override path is fragile subclassing of a private method, with no supported way to inject that subclass into `LLMAssistantAggregator` in the first place. A callback parameter that the aggregator constructs the summarizer with directly addresses both: it gives full control over when to summarize, and — because the callback is checked first — the framework can skip `estimate_context_tokens()` entirely when a callback is present and says "not yet."

### Proposed Solution

Add `should_summarize_callback` as a parameter that, when supplied, replaces the built-in threshold checks in `_should_summarize()` (per the issue's stated proposal — this is the maintainer-facing default I'm proposing, pending their answer to the open questions below). Thread it through so it can be set on `LLMAssistantAggregatorParams` and passed down into the `LLMContextSummarizer` the aggregator constructs internally, since today there's no way to inject a custom summarizer instance at all.

Open design questions (posted on the issue; no maintainer response yet):
1. Replace vs. additional gate — the issue text says "replaces... entirely," but it's worth confirming this is really the intended behavior rather than "existing checks AND callback."
2. Return value contract — plain bool, or something richer that could also adjust summarization config per-decision
3. Sync vs. async callable support.
4. Confirming that omitting the callback preserves today's default behavior exactly.

Since there's no reply yet, I'm treating "replace, bool return, both sync and async supported, default behavior unchanged when omitted" as my working assumption for design purposes below, but I'm holding off on opening a PR until at least the replace-vs-gate and return-value questions are answered — those two change the method signature and call site, so starting implementation now risks rework.

### Implementation Plan

+ Add `should_summarize_callback:` `Optional[Callable]` to the summarizer's constructor (and/or `LLMAutoContextSummarizationConfig`, depending on maintainer preference on where config should live)
+ Provide a way to pass this through `LLMAssistantAggregatorParams` down to the `LLMContextSummarizer` instance the aggregator constructs — this likely requires the aggregator to accept a summarizer-construction parameter it doesn't currently expose
+ Update `_should_summarize()` to check for the callback first and short-circuit before calling `estimate_context_tokens()` when one is present
+ Support both sync and async callables (e.g., via a small `maybe_await` helper, consistent with other optional-async-hook patterns in the codebase)
+ Add unit tests: callback returning `True/False`, async callback, no-callback default-behavior parity, and a test asserting `estimate_context_tokens()` is not called when a callback is present (this is the perf claim from the issue, so it should be tested explicitly)
+ Add Google-style docstrings for the new parameter
+ Add a towncrier changelog fragment (`changelog/4795.added`) once implementation lands

Using UMPIRE framework (adapted):

**Understand:** 

`LLMContextSummarizer._should_summarize()`, part of `pipecat.processors.aggregators.llm_context_summarizer`, fires on every `LLMFullResponseStartFrame` and always runs `estimate_context_tokens()` against the thresholds in `LLMAutoContextSummarizationConfig` (`max_context_tokens`, default 8000; `max_unsummarized_messages`, default 20). There's no supported way to change this decision short of subclassing `LLMContextSummarizer` and overriding the private `_should_summarize()`. The ask is a `should_summarize_callback` parameter that, when supplied, replaces this threshold logic entirely and lets the token-estimation work be skipped.

**Match:** 

A few existing patterns in the codebase are directly relevant:

+ Config-level optional override, verified in the docs: `LLMContextSummaryConfig.llm` is exactly this shape already — an `Optional[LLMService]` field, defaulting to `None`, where "if set, use this instead of the pipeline default; if None, fall back to default behavior." `should_summarize_callback` can follow the identical pattern on `LLMAutoContextSummarizationConfig: Optional[Callable]`, default `None`, same fallback contract.
+ Event-handler / pyee `EventEmitter` pattern: the existing `on_summary_applied` hook is registered via `@summarizer.event_handler("on_summary_applied")` (and mirrored on the aggregator via `@assistant_aggregator.event_handler(...)`). This confirms the project already has an established, non-subclassing way to expose customization points — worth raising with the maintainer as a possible alternative shape (an `on_should_summarize` event with a settable return, vs. a plain constructor callback), though the issue as written asks specifically for a constructor parameter.
+ On-demand override precedent: `LLMSummarizeContextFrame(config=LLMContextSummaryConfig(...))` already lets a caller override generation settings per-request without touching the constructor. Less directly applicable to a "should I trigger" decision (that check has already happened by the time a frame reaches the pipeline), but it's evidence the maintainers are comfortable with per-call override objects generally.
+ Backward-compat renaming precedent: the recent `enable_context_summarization → enable_auto_context_summarization` rename kept the old name working with a DeprecationWarning rather than a hard break. If the maintainer wants the callback somewhere other than where I've assumed (e.g., directly on `LLMAssistantAggregatorParams` instead of nested in `LLMAutoContextSummarizationConfig`), this precedent suggests they'd expect a soft-deprecation path rather than a breaking move — good to keep in mind when proposing the parameter's home.

**Plan:**
1. Add `should_summarize_callback: Optional[Callable] = None` to `LLMAutoContextSummarizationConfig` in `src/pipecat/utils/context/llm_context_summarization.py` (path to confirm locally — inferred from the import shown in docs, not yet grepped).
2. In `LLMContextSummarizer._should_summarize()` (`src/pipecat/processors/aggregators/llm_context_summarizer.py`), check for the callback first; if present, call it (supporting both sync and async — via a small `maybe_await` helper, adding one if the codebase doesn't already have an equivalent utility) and return its result directly, without calling `estimate_context_tokens()` — this is what satisfies the stated perf goal.
3. Thread the parameter through so it's visible wherever `LLMAutoContextSummarizationConfig` is already accepted — no new plumbing needed here since the config object already flows from `LLMAssistantAggregatorParams` into `LLMContextSummarizer` today.
4. Add/extend unit tests (exact test file to confirm — likely alongside other summarizer tests, following the project's existing test-layout convention).
5. Update the two docs pages I fetched (`/pipecat/fundamentals/context-summarization` and the API reference page) to document the new field — flagging that docs may live in a separate repo from `pipecat-ai/pipecat` itself, which I haven't confirmed.
6. Add a changelog fragment per the project's towncrier convention.

**Implement:** 

https://github.com/shivaAcharya/pipecat/tree/feature/4795/should_summarize_callback-param-in-LLMContextSummarizer

**Review:**

Self-review checklist before opening a PR (per `CONTRIBUTING.md` conventions used elsewhere in the project):

- [ ] New parameter has a Google-style docstring with type, default, and behavior description (matching the style of existing `ParamField` entries like llm on `LLMContextSummaryConfig`)
- [ ] Backward compatibility verified: omitting `should_summarize_callback` produces byte-identical behavior to today (existing tests for default threshold behavior still pass unmodified)
- [ ] No new required constructor args — everything is `Optional` with a safe default, consistent with how every other field on `LLMAutoContextSummarizationConfig` is optional
- [ ] Both sync and async callables tested explicitly, not just one
- [ ] Changelog fragment added following the existing `changelog/NNNN.added` format seen in the CHANGELOG history
- [ ] No unrelated formatting/lint changes bundled into the diff
- [ ] PR description explicitly calls out the replace-vs-gate decision and any other assumption made, per this project's pattern of surfacing non-obvious design calls to reviewers (as I did on PR #4212)

**Evaluate:**

 - Run the full existing summarizer test suite to confirm no regressions in default (no-callback) behavior
 - Add a targeted test asserting `estimate_context_tokens` is never called when a callback is supplied (this is the concrete, testable form of the issue's performance claim)
 - Manually exercise both an always-`True` and always-`False` callback against a live (or mocked) `LLMContextAggregatorPair` and confirm summarization fires / never fires accordingly
 - Manually test an async callback to confirm no event-loop issues (e.g., accidentally calling it without awaiting)

---

## Testing Strategy

### Unit Tests

- [ ] Callback provided, returns `True` → summarization triggers on the next `LLMFullResponseStartFrame`, regardless of token/message counts
- [ ] Callback provided, returns `False` → summarization does not trigger even when both built-in thresholds are far exceeded
- [ ] Callback provided (sync) vs. callback provided (async) → both are correctly invoked and awaited where necessary; assert no `RuntimeWarning` about un-awaited coroutines
- [ ] No callback provided → behavior is identical to current default threshold logic (regression/parity test against existing behavior)
- [ ] Callback provided → `estimate_context_tokens()` is asserted not called (via `unittest.mock.patch`), directly verifying the stated performance improvement
- [ ] Callback raises an exception → confirm this fails loudly/predictably rather than silently falling back to threshold logic (behavior here should be explicitly decided, not accidental)

### Integration Tests

- [ ] Full pipeline run with `LLMContextAggregatorPair`, `enable_auto_context_summarization=True`, and a callback that gates on a custom condition (e.g., a counter reaching a specific turn number) — confirm the resulting context after summarization matches expectations (system message preserved, recent messages preserved, summary inserted)
- [ ] Full pipeline run confirming `on_summary_applied` still fires correctly when the callback (rather than thresholds) is what triggered the summarization — i.e., the event-emission path downstream of the decision is unaffected by which mechanism made the decision

### Manual Testing

Once maintainer input unblocks implementation: run a short local voice-agent example (adapting the existing context-summarization example script) with a callback that logs every time it's consulted, and manually verify in the logs that (a) it's called every turn, (b) `estimate_context_tokens` is not called alongside it, and (c) summarization behavior matches the callback's return values turn-by-turn. Results to be recorded here once run.

---

## Implementation Notes

### Week 7 Progress

Not started — blocked on maintainer confirmation of the four open design questions (replace-vs-gate, return contract, sync/async, default-parity), since points 1-2 above depend on the answers. Branch: feature/4795/should_summarize_callback-param-in-LLMContextSummarizer. Will link individual commits here as they land.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
