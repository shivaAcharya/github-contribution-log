# Contribution 1: Media upload + additional metadata for traces on Langfuse

**Contribution Number:** 1
**Student:** Shiva Acharya
**Issue:** https://github.com/pipecat-ai/pipecat/issues/4212
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it directly overlaps with my hands-on experience building with ElevenLabs TTS, OpenAI LLM streaming, and Deepgram STT at PlaceOrder AI. I've personally hit this inconsistency when debugging latency in voice pipelines, so the problem feels concrete rather than theoretical — and that makes it motivating to fix properly.

It's also a great entry point into Pipecat's core pipeline architecture. Tracing how metrics are instrumented across service types will give me a solid understanding of the frame processor lifecycle, which I can build on for future contributions.

---

## Understanding the Issue

### Problem Description

Pipecat's OpenTelemetry tracing integration with Langfuse is incomplete for audio services. When STT and TTS spans are emitted, they carry text data (transcripts, synthesized text, metrics) but never the actual audio bytes. There's also no supported way to attach arbitrary user-defined metadata — like a user_id, call_id, or session_type — to a trace beyond the hardcoded conversation_id.

### Expected Behavior

 . STT spans should have the input audio attached as a media object, so you can listen to what the user said directly from the Langfuse trace view.
 . TTS spans should have the output audio attached, so you can hear what the bot said.
 . Both span types should accept a metadata dict of custom key-value pairs that surface as filterable span attributes in Langfuse.

### Current Behavior

STT spans show a transcript string and timing metrics, but no audio. TTS spans show the synthesized text and character count, but no audio. There is no metadata parameter on either span function — the only workaround is the untyped **kwargs catch-all, which accepts primitives but is undocumented and not threaded through the decorator layer.

### Affected Components

src/pipecat/utils/tracing/service_attributes.py — add_tts_span_attributes() and add_stt_span_attributes() are missing the parameters entirely
src/pipecat/utils/tracing/service_decorators.py — @traced_tts and @traced_stt don't accumulate audio bytes to pass downstream
Individual service files (elevenlabs/tts.py, deepgram/stt.py, openai/tts.py) if a service-level tracing_metadata constructor param is added

---

## Reproduction Process

### Environment Setup

git clone https://github.com/<your-fork>/pipecat
cd pipecat
pip install -e ".[tracing,elevenlabs,deepgram,openai]"
cp examples/open-telemetry/langfuse/.env.example examples/open-telemetry/langfuse/.env
# Fill in LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY, DEEPGRAM_API_KEY, ELEVENLABS_API_KEY, OPENAI_API_KEY

The main friction point is getting the base64-encoded Langfuse credentials correct for the OTLP exporter header — the README explains this but it's easy to misconfigure.

### Steps to Reproduce

1. Run the Langfuse OTel example: cd examples/open-telemetry/langfuse && python bot.py
2. Speak into the bot and complete a turn (STT → LLM → TTS)
3. Open cloud.langfuse.com → find the trace → expand the stt_deepgramsttservice span
4. Observed: The span shows transcript, language, metrics.ttfb, and settings — no audio media object, no custom metadata field
5. Expand the tts_elevenlabsttsservice span
5. Observed: The span shows text, voice_id, metrics.character_count — no audio output, no custom metadata field

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

Add audio_data: bytes | None and metadata: dict[str, Any] | None parameters to both add_tts_span_attributes() and add_stt_span_attributes(). When audio is provided, base64-encode it and set langfuse.media in Langfuse's expected JSON format ({"type": "audio", "data": "<base64>", "mediaType": "audio/wav"}), plus audio.data_size_bytes for quick filtering. For metadata, flatten the dict as metadata.<key> span attributes, skipping non-primitives gracefully. Then update @traced_tts in service_decorators.py to accumulate TTSAudioRawFrame bytes as the generator yields and pass the concatenated result to add_tts_span_attributes() after completion.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:**
Pipecat emits OTel spans for STT and TTS via add_stt_span_attributes() and add_tts_span_attributes(). These functions have no parameters for audio bytes or user metadata, so Langfuse traces for audio services are text-only and not customizable.

**Match:**
add_gemini_live_span_attributes() and add_openai_realtime_span_attributes() in the same file already accept audio_data_size: int and set it as audio.data_size_bytes — the pattern for audio metadata on spans is already established. The LLM span function uses input and output as the canonical attribute keys Langfuse maps to its input/output fields — the same convention will apply to STT/TTS audio. The settings.* flattening loop (already in all attribute functions) is the model for flattening a metadata dict.

**Plan:** [Step-by-step implementation plan]
1. In service_attributes.py, add audio_data: bytes | None = None and metadata: dict[str, Any] | None = None to add_tts_span_attributes(). When audio present: set audio.data_size_bytes, base64-encode and set langfuse.media, set output to the base64 string. When metadata present: iterate and set metadata.<key> for each primitive value.

2. Do the same for add_stt_span_attributes() — audio goes to input and langfuse.media, not output.

3. In service_decorators.py, update @traced_tts to accumulate TTSAudioRawFrame.audio bytes from the yielded frames during run_tts() execution, then pass the concatenated bytes as audio_data to add_tts_span_attributes() when closing the span.

4. Add an optional tracing_metadata: dict | None = None constructor parameter to TTSService and STTService base classes (lower priority, can be a follow-up PR). Services would pass self.tracing_metadata into the decorator so users can configure it once at instantiation rather than per call.

5. Add changelog/4212.added with a towncrier-formatted entry describing the new capability.

**Implement:** https://github.com/shivaAcharya/pipecat/tree/feature/4212-langfuse-media-metadata

**Review:**
 [] Docstrings follow Google convention (per pyproject.toml: convention = "google")
 [] New parameters have type annotations and are keyword-only where appropriate
 [] All new parameters default to None — no breaking changes to existing callers
 [] changelog/4212.added created for towncrier
 [] No new required dependencies introduced
 [] PR description references Fixes #4212

**Evaluate:**

Unit tests in tests/test_tracing_service_attributes.py:

1. test_tts_span_with_audio_data -> audio.data_size_bytes set, langfuse.media is valid base64 JSON, output is set
2. test_stt_span_with_audio_data -> input is set, langfuse.media is valid base64 JSON
3. test_tts_span_with_metadata -> metadata.session_type, metadata.user_tier appear as span attributes
4. test_stt_span_with_metadata -> Same for STT
5. test_metadata_skips_non_primitives -> Passing {"nested": {"a": 1}} doesn't raise, no attribute set for that key
6. test_no_audio_no_media_attribute -> Without audio_data, langfuse.media is never set (regression guard)

Run full suite with pytest tests/ -x -q to confirm no regressions. Manual confirmation via the Langfuse OTel example: after the fix, STT and TTS spans should show audio media objects and any custom metadata keys in the Langfuse UI.

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
