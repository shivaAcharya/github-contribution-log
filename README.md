# Contribution 1: Media upload + additional metadata for traces on Langfuse

**Contribution Number:** 1

**Student:** Shiva Acharya

**Issue:** https://github.com/pipecat-ai/pipecat/issues/4212

**Status:** Phase III (In Progress)

---

## Why I Chose This Issue

I chose this issue because it directly overlaps with my hands-on experience building with ElevenLabs TTS, OpenAI LLM streaming, and Deepgram STT at PlaceOrder AI. I've personally hit this inconsistency when debugging latency in voice pipelines, so the problem feels concrete rather than theoretical — and that makes it motivating to fix properly.

It's also a great entry point into Pipecat's core pipeline architecture. Tracing how metrics are instrumented across service types will give me a solid understanding of the frame processor lifecycle, which I can build on for future contributions.

---

## Understanding the Issue

### Problem Description

Pipecat's OpenTelemetry tracing integration emits spans for STT and TTS operations but never attaches the audio itself to those spans. Looking at add_tts_span_attributes() (service_attributes.py:66-119) and add_stt_span_attributes() (service_attributes.py:121-182) — neither function has an audio_data parameter. There is also no metadata parameter for attaching custom key-value pairs to a span. The only escape hatch is **kwargs at lines 116–118 and 180–182, which silently drops anything that isn't a str, int, float, or bool — so passing audio bytes or a dict through kwargs fails silently.

### Expected Behavior

 + TTS spans should have the synthesized output audio attached as a Langfuse media object, allowing you to listen to what the bot said directly in the Langfuse trace view.
 + STT spans should have the input audio attached as a media object, so you can hear the user's exact speech for that turn.
 + Both span types should accept a metadata: dict of custom key-value pairs (e.g. user_id, session_type, call_id) that surface as filterable span attributes in Langfuse.

### Current Behavior

TTS spans show text, voice_id, metrics.character_count, metrics.ttfb, and flattened settings.* attributes — no audio. STT spans show transcript, language, vad_enabled, metrics.ttfb, and settings.* — no audio. There is no supported metadata parameter; the **kwargs fallback at the bottom of both functions accepts primitives only and is undocumented as a metadata pathway.

### Affected Components

+ service_attributes.py:66–182 — add_tts_span_attributes() and add_stt_span_attributes() are missing the parameters entirely
+ service_decorators.py:268–287 — traced_create_audio_context in traced_tts never accumulates TTSAudioRawFrame bytes
+ service_decorators.py:496–517 — open_span in traced_stt never accumulates input AudioRawFrame bytes
+ service_decorators.py:300–319 — traced_push_frame already intercepts all pushed frames but ignores TTSAudioRawFrame
---

## Reproduction Process

### Environment Setup

```bash
```git clone https://github.com/<your-fork>/pipecat
cd pipecat
uv sync --group dev --all-extras --no-extra gstreamer --no-extra local
uv run pre-commit install
cp examples/open-telemetry/langfuse/.env.example examples/open-telemetry/langfuse/.env
# Fill in LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY, DEEPGRAM_API_KEY, ELEVENLABS_API_KEY, OPENAI_API_KEY
# Note: OTLP header requires base64("pk-...:sk-...") — misconfiguring this produces silent trace loss```
```

The main friction point is getting the base64-encoded Langfuse credentials correct for the OTLP exporter header — the README explains this but it's easy to misconfigure.

### Steps to Reproduce

1. Run the Langfuse OTel example: cd examples/open-telemetry/langfuse && python bot.py
2. Speak into the bot and complete a turn (STT → LLM → TTS)
3. Open cloud.langfuse.com → find the trace → expand the stt_deepgramsttservice span
4. Observed: The span shows transcript, language, metrics.ttfb, and settings — no audio media object, no custom metadata field
5. Expand the tts_elevenlabsttsservice span
5. Observed: The span shows text, voice_id, metrics.character_count — no audio output, no custom metadata field

## My Findings

The gap is confirmed by reading the source:
 + service_attributes.py:116–118: **kwargs loop filters to isinstance(value, (str, int, float, bool)) — passing audio bytes via kwargs silently drops them
 + service_decorators.py:300–319: traced_push_frame in traced_tts only looks for MetricsFrame; TTSAudioRawFrame frames pass through completely unobserved
 + service_decorators.py:627–644: patched_push_frame in traced_stt handles VADUserStartedSpeakingFrame, TranscriptionFrame, UserStoppedSpeakingFrame, and MetricsFrame — input AudioRawFrame is never touched

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/shivaAcharya/pipecat/tree/feature/4212-langfuse-media-metadata
- **Screenshots/logs:** [If applicable]
- **My findings:** 
The gap is confirmed by reading add_stt_span_attributes() and add_tts_span_attributes() in service_attributes.py — neither function has an audio_data or metadata parameter. The **kwargs handler at the bottom of each function only accepts str/int/float/bool primitives, so even if you passed audio bytes via kwargs they'd be silently dropped.

---

## Solution Approach

### Analysis

The root cause is that add_tts_span_attributes() and add_stt_span_attributes() in src/pipecat/utils/tracing/service_attributes.py were designed to carry text and metric data only. Audio bytes were never part of the original API. The decorator layer (service_decorators.py) wraps run_tts() and _handle_transcription() and is where span attributes get collected after execution — but it currently only reads self._metrics.ttfb and character count. There's no accumulation of the yielded TTSAudioRawFrame bytes, and no pathway for user-defined metadata to reach the span.
The **kwargs escape hatch exists but isn't the right mechanism: it's untyped, silently drops non-primitives, and requires callers to know to pass things as flat keyword arguments rather than a structured dict.

### Proposed Solution

Audio: Add audio_data: bytes | None = None to add_tts_span_attributes() and add_stt_span_attributes(). When present:

 + Set audio.data_size_bytes (consistent with the existing pattern in add_gemini_live_span_attributes() at line 339 and add_openai_realtime_span_attributes() at line 431)
 + Base64-encode and set langfuse.media as a JSON string in Langfuse's expected format: {"type": "audio", "data": "<base64>", "mediaType": "audio/wav"}
 + Set output (TTS) or input (STT) to the same base64 string (mirrors how add_llm_span_attributes() uses input/output at lines 228–231)

At the decorator level:

+ TTS: extend the entry dict in _tts_spans to include audio_chunks: list[bytes]. In traced_push_frame (lines 300–319), add a branch for TTSAudioRawFrame that appends frame.audio to entry["audio_chunks"]. When the span closes (in end_tts_span), concatenate and pass to add_tts_span_attributes().
+ STT: patch process_frame on the owner class (analogous to how push_frame is already patched) to accumulate incoming AudioRawFrame.audio bytes into _stt_span_state["audio_chunks"]. Pass concatenated bytes to add_stt_span_attributes() when the span closes.

Metadata: Add metadata: dict[str, Any] | None = None to both functions. Flatten as metadata.<key> span attributes, skipping non-primitives. Thread the parameter up through traced_create_audio_context (TTS) and open_span (STT) so services can supply it. Optionally expose a constructor-level tracing_metadata: dict | None on TTSService / STTService base classes, so users configure it once per service instance rather than per-call.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:**
Pipecat emits OTel spans for STT and TTS via add_stt_span_attributes() and add_tts_span_attributes() in service_attributes.py. These functions have no parameters for audio bytes or user metadata, so Langfuse traces for audio services are text-only and not user-customizable.

**Match:**
 + add_gemini_live_span_attributes() (line 292) and add_openai_realtime_span_attributes() (line 395) both have audio_data_size: int | None → span.set_attribute("audio.data_size_bytes", ...). This is the established pattern for audio metadata on spans.
 + add_llm_span_attributes() uses input and output as the canonical attribute keys Langfuse maps to its input/output fields (lines 228–231).
 + add_openai_realtime_span_attributes() flattens a session_properties dict as session.<key> (lines 463–474) — the model for flattening a metadata dict as metadata.<key>.
 + The _tts_spans[context_id] entry dict already accumulates per-context state ({"span", "ttfb_recorded"}); adding "audio_chunks": [] follows that pattern.
 + The _stt_span_state dict (line 629) already tracks span, segment_start_time, segments; adding "audio_chunks": [] follows that pattern.
 + Tests use unittest.TestCase + @unittest.skipUnless(HAS_OPENTELEMETRY, ...) + InMemorySpanExporter, as established in test_tracing_context.py.

**Plan:** [Step-by-step implementation plan]
1. service_attributes.py — add audio_data: bytes | None = None and metadata: dict[str, Any] | None = None to add_tts_span_attributes() (line 66). When audio_data is present: set audio.data_size_bytes, base64-encode and set langfuse.media, set output. For metadata: iterate and set metadata.<key> for each primitive value.
2. service_attributes.py — same changes to add_stt_span_attributes() (line 121), with audio going to input and langfuse.media instead of output.
3. service_decorators.py — in traced_tts, extend the _tts_spans entry dict to include "audio_chunks": []. In traced_push_frame (lines 300–319), add a branch for TTSAudioRawFrame that appends frame.audio. In end_tts_span (lines 208–219), concatenate audio_chunks and pass as audio_data to add_tts_span_attributes() (called from traced_create_audio_context, lines 277–284).
4. service_decorators.py — in traced_stt, add "audio_chunks": [] to the _stt_span_state dict. Add a patch_process_frame(owner) function (parallel to patch_push_frame) that wraps owner.process_frame to accumulate AudioRawFrame.audio bytes when a span is open. Pass concatenated bytes to add_stt_span_attributes() when the span closes.
5. Add tracing_metadata: dict[str, Any] | None = None constructor param to TTSService and STTService base classes. Thread it through traced_create_audio_context and open_span respectively so users can configure it once per service instance.
6. Create changelog/4212.added.md with a towncrier entry describing the new capability.

**Implement:** https://github.com/shivaAcharya/pipecat/tree/feature/4212-langfuse-media-metadata

**Review:**
 - [ ] Docstrings follow Google convention, per pyproject.toml: convention = "google"
 - [ ] New parameters have type annotations; audio_data and metadata default to None — no breaking changes to existing callers
 - [ ] changelog/4212.added.md created (naming per CONTRIBUTING.md: <PR_number>.added.md)
 No new required dependencies — base64 is stdlib, Langfuse reads the span attribute via the existing OTLP exporter
 ruff check and ruff format --check pass (line length 100)
 PR description references Fixes #4212

**Evaluate:**

Unit tests in tests/test_tracing_service_attributes.py using the same pattern as test_tracing_context.py:

1. test_tts_span_with_audio_data -> audio.data_size_bytes is set correctly; langfuse.media is valid base64 JSON with type: audio; output is set
2. test_stt_span_with_audio_data -> input is set; langfuse.media is valid base64 JSON
3. test_tts_span_with_metadata -> metadata.session_type, metadata.user_tier appear as span attributes
4. test_stt_span_with_metadata -> Same for STT
5. test_metadata_skips_non_primitives -> Passing {"nested": {"a": 1}} doesn't raise; no metadata.nested attribute set
6. test_no_audio_no_media_attribute -> Without audio_data, langfuse.media is never set (regression guard)
7. test_existing_tts_attributes_unchanged -> Existing callers passing only text, voice_id, model still work — no regression

Run full suite with pytest tests/ -x -q to confirm no regressions. Manual confirmation via the Langfuse OTel example: after the fix, STT and TTS spans should show audio media objects and any custom metadata keys in the Langfuse UI.

---

## Testing Strategy
Run with:
`uv run pytest tests/test_tracing_service_attributes.py -v`

### Unit Tests

1. test_tts_span_with_audio_data — verifies that when audio_data is passed to add_tts_span_attributes(), the span has audio.data_size_bytes set to the correct length, output set to the base64 string, and langfuse.media is valid JSON with type: audio and a matching data field.

2. test_stt_span_with_audio_data — same as above for add_stt_span_attributes(), but checks input instead of output.

3. test_tts_span_with_metadata — verifies that a metadata dict with primitive values (session_type, user_tier) produces metadata.session_type and metadata.user_tier span attributes on a TTS span.

4. test_stt_span_with_metadata — same metadata flattening check for an STT span.

5. test_metadata_skips_non_primitives — passes a metadata dict containing a nested dict ({"nested": {"a": 1}}) alongside a valid primitive. Asserts that metadata.nested is not set but metadata.primitive is, confirming non-primitive values are silently dropped without raising an error.

6. test_no_audio_no_media_attribute — calls add_tts_span_attributes() without audio_data and asserts that audio.data_size_bytes, langfuse.media, and output are all absent from the finished span (regression guard for existing callers).

7. test_existing_tts_attributes_unchanged — calls add_tts_span_attributes() with the original parameters (text, voice_id, model, character_count, ttfb) and verifies all pre-existing span attributes (gen_ai.provider.name, gen_ai.request.model, voice_id, text, metrics.character_count, metrics.ttfb) are still set correctly.

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [3] Progress

Two new optional parameters were added to the span attribute functions in service_attributes.py:
audio_data: bytes | None = None
When raw audio bytes are provided, three span attributes are set:
 + audio.data_size_bytes — the byte length, consistent with the existing pattern in add_gemini_live_span_attributes() and add_openai_realtime_span_attributes()
 + langfuse.media — a JSON-encoded media object ({"type": "audio", "data": "<base64>", "mediaType": "audio/wav"}) that Langfuse uses to render a playable audio player in the trace view
 + output (TTS) or input (STT) — the raw base64 string, following the input/output semantic convention used by add_llm_span_attributes()

metadata: dict[str, Any] | None = None
Each key-value pair in the dict is flattened as a metadata.<key> span attribute. Only primitive values (str, int, float, bool) are written; non-primitives are silently skipped, matching the defensive pattern used throughout the file for settings.*, session.*, and extra.* attributes. import base64 and import json were added at the top of the file. All new parameters default to None, so all existing callers are unaffected.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:**
 + src/pipecat/utils/tracing/service_attributes.py — added audio_data and metadata params to add_tts_span_attributes() and add_stt_span_attributes(); added import base64 and import json
 + tests/test_tracing_service_attributes.py — new test file with 7 unit tests covering both functions

- **Key commits:**
  https://github.com/shivaAcharya/pipecat/commit/c7a5b3fa4cb5f96a3cb8a3b6eb926852f3964e6c
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
