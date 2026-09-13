# Testing

[English](testing.md) | [Lietuvių](testing_LT.md)  
[README (EN)](../README.md) | [README (LT)](../README_LT.md)

## Overview

Testing was performed continuously throughout the development of the Real-Time Q&A Assistant.

Because the application combines audio capture, network APIs, asynchronous workers, semantic processing, and a desktop GUI, testing was not limited to isolated functions.

The project used a combination of:

- isolated component tests;
- mock-based tests;
- controlled audio tests;
- real end-to-end runtime tests;
- regression testing after architectural changes;
- manual usability observation.

The main objective was not only to verify that individual components worked, but also to confirm that the complete real-time processing pipeline behaved correctly under realistic conditions.

## Testing Approach

The project followed an incremental testing strategy.

A typical development cycle was:

```text
implement
→ test isolated behaviour
→ integrate
→ run controlled scenario
→ observe
→ correct
→ regression test
```

This approach helped identify problems close to the stage where they were introduced.

Large groups of untested changes were intentionally avoided where practical.

## Audio Capture Testing

The first testing stage focused on Windows system-audio capture.

The objective was to confirm that the application could reliably obtain the audio being played by the computer through WASAPI loopback.

The test verified:

```text
WASAPI loopback device discovery
audio stream initialization
correct channel configuration
sample rate handling
continuous frame capture
```

The tested environment used the Windows loopback device associated with the active Realtek audio output.

Successful capture established the foundation for the remaining pipeline.

## Microphone Test

Microphone capture was also tested separately.

A small standalone test confirmed that the built-in Realtek microphone could be detected and used successfully.

However, microphone integration was intentionally not added to the main application.

The current workflow is based on system audio, and microphone support was not required for the primary use case.

This was therefore treated as a verified future capability rather than a current product feature.

## Speech-to-Text Testing

Speech-to-text testing verified that captured audio could be:

```text
converted to an in-memory WAV buffer
→ sent to the STT API
→ returned as usable text
```

The tests covered both normal speech and silent periods.

The transcription stage was later extended with pure-silence skipping, so tests also verified that silent audio could bypass unnecessary STT calls without breaking downstream processing.

## Chunk Processing Tests

Continuous audio is processed in chunks.

Testing focused on several practical risks:

```text
speech crossing chunk boundaries
missing words between chunks
duplicated words caused by overlap
silence occurring at chunk boundaries
```

The application uses approximately one second of audio overlap between consecutive chunks.

Testing confirmed that this reduced the risk of losing speech at chunk boundaries.

## Transcript Merge Testing

Audio overlap naturally creates duplicated transcript content.

The merge logic was therefore tested with consecutive transcript fragments containing repeated or partially repeated phrases.

Example:

```text
Previous:
"...how did you process the large amount of data"

New:
"the large amount of data and how did you validate it"
```

The objective was to preserve the continuation:

```text
"...how did you process the large amount of data and how did you validate it"
```

without producing duplicated phrases.

Both exact and fuzzy overlap behaviour were tested.

## Known Transcript Imperfections

Speech-to-text output is not always linguistically perfect.

During testing, occasional artifacts such as:

```text
"Could you tell me about. a time..."
```

or:

```text
"what steps you took? to improve..."
```

were observed.

These artifacts did not prevent the semantic question detector or answer-generation model from correctly understanding the question.

For that reason, additional aggressive transcript normalization was not introduced.

The current behaviour was considered acceptable unless future testing demonstrates a real user-facing problem.

## Silence Detection Testing

Silence detection was tested using the RMS audio level.

The objective was to distinguish between:

```text
active speech
short natural pauses
confirmed silence
```

The application should not evaluate every short pause as the end of a question.

A silence period must remain below the configured threshold for the configured confirmation duration before a question-boundary check is triggered.

Testing confirmed that the silence state resets correctly when speech resumes.

## Semantic Question Detection Testing

Semantic question detection was tested using spoken questions containing natural pauses.

One important scenario was a multi-part question.

Example flow:

```text
first part of question
→ silence
→ WAIT

speaker continues
→ silence
→ YES
```

This behaviour confirmed that the system did not automatically treat every pause as the end of a question.

The combination of:

```text
confirmed silence
+
semantic YES / WAIT / NO detection
```

proved more reliable than silence-only segmentation.

## Current Turn Testing

The `current_turn` state was tested to confirm that transcript fragments remain grouped until a question is semantically complete.

Important behaviours included:

```text
new transcript fragments append to the current turn
WAIT preserves the same turn
YES creates a question snapshot
continued silence does not create a new empty turn
new real text starts the next turn
```

This prevented incorrect splitting of long or multi-part spoken questions.

## Question Snapshot Testing

When a question is classified as complete, its text is frozen.

Testing confirmed that later incoming speech does not modify the already detected question.

This was important because transcription continues while answer generation may still be running.

## Question ID Testing

Each completed question receives a unique numeric ID.

Testing verified that the ID remains associated with the same question through the complete asynchronous pipeline:

```text
question detection
→ GUI entry
→ answer queue
→ answer generation
→ GUI update
```

This became particularly important once question history and navigation were introduced.

## Mock GUI Testing

Before relying only on live audio, GUI history behaviour was tested using mock events.

Mock testing made it possible to simulate:

```text
multiple detected questions
answers arriving later
navigation to previous questions
new questions arriving while viewing history
returning to the latest question
```

This allowed GUI state logic to be validated independently from speech-to-text and network timing.

## History Navigation Testing

The GUI provides:

```text
Previous
Next
Latest
Question X / Y
```

Testing verified that:

- previous questions remain accessible;
- newer questions are appended correctly;
- navigation does not corrupt history state;
- answers update the correct question;
- the interface can return to the latest question;
- new questions do not automatically interrupt the user while reviewing older history.

The `follow_latest` state was introduced and tested specifically for this behaviour.

## Asynchronous Answer Testing

Answer generation is performed by a dedicated worker.

Tests confirmed that the transcription pipeline can continue while the answer worker is processing a previously detected question.

This was important for avoiding a design where slow model responses could block incoming audio processing.

## Translation and Answer Testing

Question translation and suggested-answer generation are performed in one LLM request.

Testing verified that the response can be parsed into:

```text
translated question
suggested answer
```

The answer generation was also checked against supplied contextual information.

The goal was to generate useful responses while avoiding unsupported facts or invented user experience.

## Pure-Silence STT Regression Test

One of the final optimizations was skipping STT calls for chunks containing no detected speech.

This optimization required careful regression testing.

A critical requirement was:

> Skipping STT must not skip question-boundary processing.

A real test showed that silent chunks produced:

```text
STT SKIP
```

while the semantic question-state flow continued normally.

The test confirmed that the application could still progress through:

```text
WAIT
→ additional speech
→ YES
```

even when intermediate silence-only chunks did not call the STT API.

## Single-Instance Testing

A single-instance lock was added after development experience showed that multiple accidentally running application processes could duplicate audio processing and API usage.

Testing confirmed that:

```text
first application instance
→ acquires the lock

second application instance
→ cannot acquire the lock
→ does not start another processing pipeline
```

This reduced the risk of accidental duplicate API consumption.

## Shutdown Testing

Shutdown behaviour was tested after replacing blocking thread joins in the Tkinter main thread.

The final shutdown process was checked to confirm that:

```text
stop request is registered
audio capture stops
workers finish cleanly
remaining queued work is handled
audio resources are released
GUI closes
single-instance lock is released
```

The observed runtime sequence completed normally without leaving the application visibly frozen.

## Final End-to-End Regression Test

A final real-world regression test was performed after the optimization and cleanup phase.

The scenario contained two separate spoken questions.

The first question required multiple audio chunks and semantic continuation.

Observed flow:

```text
speech
→ transcription
→ WAIT
→ continued speech
→ WAIT
→ YES
→ answer generated
```

Silence-only chunks were observed during the same session and correctly skipped unnecessary STT calls.

A second question was then processed independently:

```text
new speech
→ new question
→ YES
→ separate answer
```

The complete test verified:

```text
system audio capture
continuous chunking
audio overlap
transcript overlap removal
current_turn accumulation
confirmed silence handling
semantic WAIT / YES states
question snapshot creation
question_id assignment
answer generation
question translation
GUI updates
history navigation
pure-silence STT skipping
clean shutdown
```

No removed debug-only functionality was required for successful operation.

## API Usage Testing

API consumption was treated as part of system reliability testing.

An earlier development incident revealed that repeated growing context and multiple simultaneously running application processes could create unexpectedly large model input usage.

After the cause was identified, testing focused on ensuring that the architecture controlled API usage through:

```text
bounded current-turn context
bounded detector input
bounded answer input
single-instance protection
pure-silence STT skipping
one combined translation + answer LLM request
```

The purpose was not to minimize every possible request, but to remove requests and context growth that had no practical value.

## Manual Usability Observation

Some aspects of the system are more useful to evaluate interactively than with isolated automated tests.

Manual runtime observation was used to evaluate:

```text
perceived response delay
GUI responsiveness
question-history behaviour
whether incoming audio appeared to queue excessively
whether question detection felt natural
shutdown behaviour
```

During the final test phase, response latency was considered acceptable for the prototype.

No additional latency instrumentation was added because there was no observed queue backlog or user-facing performance problem requiring deeper measurement.

## Testing Philosophy

Testing decisions followed several practical principles.

### Test the Real Pipeline

Mock tests are useful for isolated state logic, but audio, network, timing, and threading behaviour also require real runtime testing.

### Regression-Test Architectural Changes

Optimizations such as silence-based STT skipping can affect apparently unrelated logic.

Changes to the processing pipeline therefore require re-testing the full question-detection flow.

### Do Not Optimize Without Evidence

A technically possible optimization was not considered sufficient reason to change stable behaviour.

Changes were prioritized when testing exposed a concrete problem.

### Preserve Known Working Behaviour

Cleanup and refactoring should not silently alter question detection, history behaviour, or asynchronous processing.

### Cost Is Part of Reliability

Unexpected API consumption is not only a financial issue.

It can indicate uncontrolled application behaviour and should therefore be treated as an engineering reliability problem.

## Current Test Status

The current Windows prototype has successfully completed an end-to-end regression covering its primary workflow.

Verified areas include:

```text
Windows system-audio capture
speech-to-text
chunk overlap
transcript merge
silence detection
semantic question detection
multi-part question handling
question snapshots
question IDs
context-aware answer generation
translation
asynchronous workers
GUI history
navigation
silence-call optimization
single-instance protection
shutdown
```

Microphone capture has been tested separately but remains intentionally outside the primary application workflow.

## Remaining Testing Opportunities

Future development may justify additional testing in areas such as:

- longer continuous runtime sessions;
- different Windows audio devices;
- different microphones if microphone support is integrated;
- varying background-noise conditions;
- different speaking styles and accents;
- network interruption handling;
- API timeout and retry behaviour;
- packaged standalone executable behaviour;
- non-technical user testing;
- performance measurements across different hardware.

These tests are not required to demonstrate the current prototype, but they would become increasingly important if the project moves from a technical prototype toward wider distribution.

## Summary

Testing evolved together with the application.

The project moved from isolated component checks to full real-time regression scenarios involving:

```text
audio
→ transcription
→ state
→ semantic detection
→ asynchronous generation
→ GUI
→ shutdown
```

The final tests confirmed that the current architecture can process multiple spoken questions in sequence while remaining responsive, maintaining correct question history, avoiding unnecessary silence-related API calls, and shutting down cleanly.