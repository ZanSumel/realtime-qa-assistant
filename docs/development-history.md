# Development History

[English](development-history.md) | [Lietuvių](development-history_LT.md)  
[README (EN)](../README.md) | [README (LT)](../README_LT.md)

## Introduction

Real-Time Q&A Assistant did not start as a complete application.

The project evolved incrementally from a small audio-processing proof of concept into a modular Windows desktop application capable of capturing system audio, transcribing speech, detecting completed questions, generating context-aware answers, and maintaining question history in near real time.

Each major architectural change was introduced in response to a practical problem discovered during development and testing.

This document describes that evolution.

## Phase 1 — Initial Audio Capture

The first objective was simple:

> Capture audio played through Windows and make it available for further processing.

The application uses Windows WASAPI loopback through PyAudioWPatch.

This allows the program to capture audio coming from the computer itself rather than requiring direct microphone recording.

The first successful milestone established that the application could:

```text
Windows audio
→ WASAPI loopback
→ Python
→ raw audio frames
```

At this stage there was no continuous transcription, semantic processing, question detection, or GUI history.

The goal was only to prove that reliable system-audio capture was possible.

## Phase 2 — Speech-to-Text

Once audio capture was working, speech-to-text processing was added.

Captured audio was temporarily converted into WAV format and sent to the speech-to-text API.

An important implementation decision was to avoid writing temporary WAV files to disk.

Instead, the application creates an in-memory audio buffer.

The flow became:

```text
system audio
→ captured frames
→ in-memory WAV
→ speech-to-text
→ text
```

This created the first complete audio-to-text pipeline.

## Phase 3 — Continuous Processing

A single recording followed by a single transcription request was not sufficient for a real-time application.

The next step was continuous audio processing.

Audio was divided into manageable chunks and processed repeatedly while the application remained active.

This introduced a new challenge:

> Audio capture must continue while previous audio is being processed.

A purely sequential design would eventually cause audio processing and network requests to block one another.

This led to the introduction of background processing.

## Phase 4 — Worker Threads and Queues

The application was reorganized around independent worker threads.

The main workers became:

```text
Capture Worker
Transcription Worker
Answer Worker
```

Queues were introduced to transfer data between processing stages.

```text
Capture Worker
    ↓
audio_queue
    ↓
Transcription Worker
    ↓
answer_queue
    ↓
Answer Worker
```

GUI updates were separated through another queue:

```text
gui_queue
```

This architecture allowed each part of the system to operate independently.

Audio capture no longer had to wait for answer generation, and the GUI did not directly depend on background network operations.

## Phase 5 — Audio Overlap

Chunk-based speech processing introduced another practical problem.

Words near the end of one audio chunk could be cut between two chunks.

For example:

```text
Chunk 1:
"...the amount of"

Chunk 2:
"data and how did you..."
```

Without protection, part of the speech could be lost.

To reduce this risk, approximately one second of audio from the end of the previous chunk was included at the beginning of the next chunk.

This improved speech continuity but introduced a new problem:

> Speech-to-text results now contained duplicated words.

## Phase 6 — Transcript Overlap Removal

To remove duplicated text created by audio overlap, transcript comparison logic was added.

The system compares:

```text
end of previous transcript
        ↕
beginning of new transcript
```

Matching words are removed before the new text is appended.

Exact matching alone was not sufficient because speech-to-text results can vary slightly between two transcriptions of the same audio.

For this reason, fuzzy matching was introduced.

The overlap logic became tolerant of small differences while still avoiding aggressive removal of unrelated text.

This stage significantly improved continuity between consecutive speech-to-text results.

## Phase 7 — Conversation Turn State

Continuous transcription alone did not answer an important question:

> Which pieces of transcript belong to the same spoken question?

The application therefore introduced the concept of a current conversation turn.

```text
current_turn
```

New transcript fragments are accumulated into the current turn.

The turn remains active until the system determines that the speaker has completed a question.

This created a clear separation between:

```text
raw transcription fragments
```

and:

```text
a logical spoken question
```

The active conversation turn is also size-limited to prevent uncontrolled context growth during long sessions.

## Phase 8 — Silence Detection

A natural first approach to question boundaries is detecting when the speaker becomes silent.

The audio layer therefore began monitoring RMS volume.

When audio remains below a configured threshold for a defined amount of time, the system produces a confirmed silence boundary.

This helped identify likely moments when a speaker had stopped talking.

However, testing showed that silence alone was not enough.

A speaker may pause while thinking and then continue the same question.

Example:

```text
"Could you tell me about a project where..."
        ↓
pause
        ↓
"...you had to process a large amount of data?"
```

Treating every silence as the end of a question would generate incomplete questions.

A second level of detection was required.

## Phase 9 — Semantic Question Detection

Semantic question-state detection was added after confirmed silence.

Instead of assuming that silence means the question is complete, the accumulated conversation turn is evaluated semantically.

The detector returns one of three states:

```text
YES
WAIT
NO
```

Meaning:

```text
YES  → the question appears complete
WAIT → the speaker probably intends to continue
NO   → the current speech is not a completed question
```

This created an important two-stage model:

```text
silence detection
        ↓
semantic question detection
```

The silence layer determines when it is reasonable to evaluate the text.

The semantic layer determines whether the question is actually complete.

Testing with multi-part questions demonstrated that the `WAIT → YES` flow worked more reliably than simple silence-based segmentation.

## Phase 10 — Completed Question Snapshot

When the semantic detector returns `YES`, the active conversation turn is frozen as a completed-question snapshot.

This ensures that future incoming audio cannot modify a question that has already been recognized.

The application then waits for actual new text before resetting the active conversation turn.

This detail became important during silence-only periods.

Resetting the turn immediately after `YES` could create empty turns while the speaker remained silent.

The final behaviour became:

```text
question completed
→ snapshot stored
→ silence may continue
→ new real text arrives
→ start next conversation turn
```

## Phase 11 — Context-Aware Answer Generation

Once completed questions could be detected reliably, answer generation was added.

The answer-generation stage uses:

```text
detected question
+
user / candidate context
+
scenario / job context
```

The objective was not to generate generic answers.

The system should produce answers grounded in user-provided information and avoid inventing unsupported experience.

Translation and suggested-answer generation were combined into one LLM request.

This avoided sending the same question to the model twice.

The output structure became:

```text
QUESTION_LT
ANSWER_EN
```

The design can later be adapted to different languages or response formats.

## Phase 12 — Independent Answer Worker

Answer generation can take longer than audio transcription.

If answer generation were executed directly inside the transcription worker, incoming speech processing could be delayed.

For this reason, a dedicated answer worker was introduced.

The transcription worker now only needs to send:

```text
completed question
→ answer_queue
```

and can immediately continue processing new audio.

The answer worker handles model generation independently.

This separation became an important part of the application's real-time behaviour.

## Phase 13 — Graphical Interface

A Tkinter GUI was introduced to provide a practical desktop interface.

The interface displays:

```text
detected question
translated question
suggested answer
```

Background workers do not directly modify Tkinter widgets.

Instead, they send structured events through:

```text
gui_queue
```

The GUI thread periodically processes those events.

This keeps Tkinter operations on the main thread and avoids unsafe cross-thread GUI updates.

## Phase 14 — Question History

The original GUI displayed only the latest question.

During practical use, it became clear that users may need to review a previous question while the application continues listening.

A question-history model was therefore introduced.

Each question receives a unique:

```text
question_id
```

The GUI stores separate entries containing:

```text
question
translation
answer
status
```

Navigation controls were added:

```text
Previous
Next
Latest
Question X / Y
```

An additional `follow_latest` state controls whether the interface automatically follows new questions.

If the user is reviewing an older question, new questions continue to be stored without forcing the GUI to jump away from the currently viewed item.

## Phase 15 — Reliable Question and Answer Mapping

Asynchronous answer generation introduced a subtle problem.

An answer may arrive after the user has already navigated to another question.

Display position therefore cannot be used to determine which answer belongs to which question.

The solution was to use the unique question ID throughout the pipeline.

```text
question_id
    ↓
question snapshot
    ↓
answer queue
    ↓
generated answer
    ↓
GUI history entry
```

This created deterministic question-answer mapping independent of timing.

## Phase 16 — API Usage Investigation

During development, unexpectedly high LLM input usage revealed an important reliability issue.

Repeated processing of growing text context, combined with accidentally running multiple application processes, could dramatically increase API usage.

This incident led to several architectural changes rather than only a local fix.

The project introduced:

```text
bounded detector input
bounded answer input
bounded current conversation turn
single-instance protection
reduced redundant processing
```

This became one of the most important optimization lessons in the project:

> In an API-driven real-time application, controlling when a model is called is as important as controlling what is sent to it.

## Phase 17 — Single-Instance Protection

Testing showed that accidentally launching more than one copy of the application could result in duplicated audio processing and duplicated API requests.

A Windows file lock was added.

The application now checks whether another instance is already running.

If the lock cannot be acquired, the second process exits instead of starting another processing pipeline.

This protection is active only while the application process is running.

## Phase 18 — Pure-Silence STT Optimization

The audio system was already detecting silence for question-boundary purposes.

This information was reused to reduce unnecessary speech-to-text calls.

Each newly captured audio chunk records whether speech was detected.

If the chunk contains only silence:

```text
speech_detected = false
```

the speech-to-text request is skipped.

However, the processing cycle continues.

This was important because a silent chunk may be the chunk that confirms the end of a question.

The final logic therefore became:

```text
pure silence
→ skip STT API call
→ keep question-boundary processing active
```

A real regression test confirmed that the optimization preserved `WAIT → YES` question detection while eliminating unnecessary STT calls during silence.

## Phase 19 — Queue and State Cleanup

After the main pipeline became stable, internal queue data and state variables were reviewed.

Several values remained from earlier development stages but were no longer used.

Unused queue metadata and dead state were removed.

The audio queue was simplified to contain only:

```text
chunk number
audio frames
speech detected flag
confirmed-silence flag
```

Removing unused state made the worker interface easier to understand and reduced unnecessary internal complexity.

## Phase 20 — Debug Transcript Cleanup

During development, a full accumulated transcript was useful for debugging overlap behaviour.

The transcript continuously grew and was repeatedly printed to the terminal.

Once the overlap and conversation-turn logic had stabilized, the full transcript was no longer required by the application itself.

It was removed from the working pipeline.

The application now maintains only the state required for actual processing.

This reduced debug noise and avoided unnecessary long-running transcript growth.

## Phase 21 — Shutdown Improvements

The initial shutdown procedure waited for worker threads using blocking `join()` calls directly from the Tkinter thread.

This worked functionally but could make the GUI appear frozen while network-dependent workers were finishing their work.

The shutdown flow was redesigned.

Instead of blocking the GUI thread, Tkinter periodically checks whether background workers have finished.

The shutdown sequence became:

```text
stop capture
→ wait without blocking GUI
→ finish transcription
→ finish queued answers
→ release audio resources
→ destroy GUI
```

The application also ensures that its single-instance lock is released when the GUI runtime finishes.

## Phase 22 — Final Regression Testing

After the optimization and stability changes, a multi-question end-to-end regression scenario was performed.

The test included:

```text
system audio capture
→ multi-chunk transcription
→ audio overlap
→ transcript merge
→ WAIT state
→ continued question
→ YES state
→ answer generation
→ silence STT skip
→ second question
→ second answer
→ GUI history
→ clean shutdown
```

The final test confirmed:

- multiple questions were processed independently;
- silence-only chunks skipped STT requests;
- semantic question detection continued working;
- question IDs correctly separated questions;
- translation and suggested answers were generated;
- history navigation remained functional;
- the application shut down cleanly;
- the removed debug transcript was no longer required.

## Modularization

As the prototype grew, responsibilities were gradually separated into dedicated modules.

The private implementation currently separates concerns around:

```text
audio
transcription
LLM interaction
worker orchestration
GUI
configuration
application lifecycle
context data
```

This modularization was not designed completely in advance.

It emerged from practical development as the responsibilities of individual components became clearer.

## Development Approach

The project followed an incremental development approach.

A typical cycle was:

```text
implement one small capability
        ↓
test it independently
        ↓
integrate it into the pipeline
        ↓
perform a controlled runtime test
        ↓
observe problems
        ↓
refine the architecture
```

Large rewrites were generally avoided when a smaller targeted change could solve the problem.

Features without a clear practical purpose were intentionally postponed or excluded.

For example, direct microphone integration was tested independently but not added to the main application because the primary workflow currently requires system audio.

## Key Lessons

Several broader engineering lessons emerged during development.

### Real-Time Does Not Mean Everything Must Run Simultaneously

A responsive pipeline can be built by separating responsibilities and allowing independent stages to communicate asynchronously.

### Silence Is Useful but Not Sufficient

Audio silence provides a good trigger for semantic analysis, but it does not reliably determine whether a spoken question is complete.

### Overlap Solves One Problem and Creates Another

Audio overlap prevents missing words between chunks but requires transcript de-duplication.

### Explicit State Is Valuable

Variables such as:

```text
current_turn
question_id
follow_latest
silence_confirmed
new_turn_pending
```

make application behaviour easier to reason about than relying on implicit timing.

### API Costs Are an Architectural Concern

Unnecessary model calls, duplicated processes, and uncontrolled context growth can become real system problems.

API usage must therefore be considered during architecture design, not only after implementation.

### Debugging Features Should Not Automatically Become Product Features

The full transcript was useful while developing the overlap logic but became unnecessary once the pipeline was stable.

Development tools and production requirements should remain separate.

### Optimization Should Follow Evidence

The application was not optimized simply because optimization was possible.

Changes were introduced when testing revealed a concrete reason for them.

## Current Development State

The current prototype provides a complete working processing pipeline:

```text
Windows system audio
        ↓
capture
        ↓
speech-to-text
        ↓
overlap removal
        ↓
conversation-turn accumulation
        ↓
silence confirmation
        ↓
semantic question detection
        ↓
completed-question snapshot
        ↓
context-aware answer generation
        ↓
GUI history
```

The system has also completed a dedicated optimization and stability phase.

Current development priorities have shifted from core pipeline functionality toward:

```text
documentation
portfolio presentation
controlled distribution
future user-driven customization
```

## Future Direction

The current prototype is intentionally treated as a working baseline rather than a fixed final product.

Possible future development can be driven by actual user requirements.

Potential areas include:

- standalone Windows packaging;
- simplified configuration for non-technical users;
- secure user-side API-key handling;
- configurable response language and style;
- additional contextual data sources;
- GUI customization;
- scenario-specific question-detection behaviour;
- further performance and usability improvements based on real usage.

Future features should be added only when they provide clear practical value.

## Summary

The project evolved from a basic audio-capture experiment into a modular real-time AI processing application.

The most important development progression was:

```text
capture
→ transcribe
→ continue
→ merge
→ accumulate
→ detect
→ snapshot
→ generate
→ navigate
→ optimize
→ stabilize
```

The final architecture reflects the problems discovered during actual implementation and testing rather than a theoretical design created before development began.