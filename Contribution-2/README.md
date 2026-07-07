# Contribution 2: feat: add should_summarize_callback param to LLMContextSummarizer

**Contribution Number:** 2 

**Student:** Shiva Acharya 

**Issue:** https://github.com/pipecat-ai/pipecat/issues/4795 

**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose issue #4795, which adds a should_summarize_callback parameter to LLMContextSummarizer in pipecat-ai/pipecat, as my next contribution after wrapping up PR work on #4212 (Langfuse tracing for STT/TTS spans). What drew me to it is that it sits squarely in the conversation-management layer of a voice AI pipeline — deciding when and how context gets summarized directly affects response latency and quality in real-time agents, which connects to the production concerns I dealt with using ElevenLabs TTS, OpenAI LLM streaming, and Deepgram STT at PlaceOrder AI. It's also a clean, additive feature request rather than a bugfix requiring root-cause spelunking, which makes it a good pace-setter after the more involved async/tracing work in #4212.

I'm hoping to use this issue to build depth in a part of the pipecat codebase I haven't touched yet — the context-summarization logic — and to practice designing a public API surface (a callback parameter) that other contributors and users will build on, rather than just patching internal behavior. Since the issue is well-specified but leaves the implementation details open, I also want to get more comfortable proposing a concrete design in a PR description and inviting maintainer feedback, rather than working from a fix that's already fully scoped out for me.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

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
