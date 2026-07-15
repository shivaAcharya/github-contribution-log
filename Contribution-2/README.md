# Contribution 2: feat: add should_summarize_callback param to LLMContextSummarizer

**Contribution Number:** 2 

**Student:** Shiva Acharya 

**Issue:** https://github.com/pipecat-ai/pipecat/issues/4795 

**Status:** Phase II Complete

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

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

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
