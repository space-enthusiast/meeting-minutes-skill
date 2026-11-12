---
name: meeting-minutes
description: Turn a meeting recording (m4a, mp3, wav, mp4, ...) into minutes on the user's own computer. Whisper transcribes, pyannote separates speakers, and the agent writes minutes from the speaker-labelled transcript. Use when the user gives a recording and asks for minutes, a summary, a transcript, speech-to-text, or "who said what". 회의 녹음으로 회의록, 전사본, 화자 분리, STT 를 요청할 때 사용한다. The audio never leaves the computer.
license: MIT
---

# Meeting Minutes

Produce minutes that a person who missed the meeting can trust: every statement traceable to the recording, every speaker label honest about how sure it is.

The recording is the source of truth. A transcript is evidence, and minutes are a reading of that evidence. Notes written by a meeting bot or by an attendee are secondary: use them only to cross-check.

## Operating modes

Loading this skill does not authorize a full run. Pick the smallest mode that answers the request.

- **Guidance mode**: questions about how to transcribe, which model fits a machine, why speaker labels are wrong, how to read an existing transcript. Answer from the relevant file. Do not install anything or create files.
- **Transcript mode**: the user wants the text only. Run setup, transcription, and (unless declined) speaker separation. Stop after the speaker-labelled transcript.
- **Minutes mode**: the user wants minutes. Run every phase and write the files listed under "Outputs".

If the request could mean more than one mode, ask one short question before installing anything.

## Platform terminology

This skill is agent-neutral.

- **Shell** is whatever the platform uses to run commands on the user's computer.
- **Background run** is the platform's way to start a long command and be told when it ends. If there is none, run the command in the foreground and tell the user how long to expect.
- **Environment** is a Python environment that this skill creates once and reuses. `<python>` is its interpreter.
- **Reference script** is a code block in these files. Save it under `<out>/work/` and run it with `<python>`. It is a tested starting point, not a fixed contract: adapt it when the machine, the library version, or the request calls for it, and say what you changed.

## Principles

1. **The audio stays local.** Never upload the recording, or any part of it, to a remote service. Transcription and speaker separation run on this computer.
2. **Be open about where the text goes.** The transcript is text, and the agent writing the minutes reads it. If that agent is a cloud model, the text goes to that provider. Say so once before the first run if the meeting may be sensitive.
3. **Only what was said.** Minutes contain what the transcript supports. No invented owners, dates, numbers, or decisions.
4. **Honest speaker labels.** A number becomes a name only with evidence. A guess is marked as a guess.
5. **One heavy job at a time, at low priority.** Speech models use the whole machine. Never run transcription and speaker separation together. Slower is fine; the user still has to work on this computer.
6. **Ask before installing, never handle secrets.** Show the command before installing a program. The user signs in, accepts model terms, and stores tokens. Do not ask for a password or a token in the conversation.
7. **Recognition errors are expected.** Names, numbers, and jargon are the weak points. Verify what matters instead of smoothing it over.

## Setup values

Resolve these before the first phase. Ask the user only for what you cannot find out yourself.

- **Skill directory**: the absolute directory containing this `SKILL.md`.
- **Recording**: the absolute path of the audio or video file.
- **Output directory `<out>`**: default is a new folder next to the recording, named after it with `-minutes` appended. Use another place when the user names one or the recording sits in a synced or shared folder where working files do not belong.
- **Language**: ask, or let the model detect it.
- **Attendees**: names, and teams or roles if known. Needed to put names on speakers and for the minutes header. If the user does not know, continue with numbered speakers.
- **Title, date and time, purpose**: for the header. Take them from the user or from the file name and metadata; never from guesswork.

## Outputs

All paths are inside `<out>`.

| File | Written in | Content |
|---|---|---|
| `transcript.json` | Phase 2 | Segments with word-level timestamps. Never edited afterwards |
| `transcript.txt` | Phase 2 | The same text, one timestamped line per segment. The uncorrected record |
| `diarization.json` | Phase 3 | Who spoke when, speakers numbered from 1 |
| `speakers.json` | Phase 4 | Speaker number to name, with uncertainty marks |
| `corrections.tsv` | Phase 4 | Recurring recognition errors and their fixes |
| `transcript-speakers.txt` | Phase 4 | The speaker-labelled, corrected transcript. The basis for the minutes |
| `minutes.md` | Phase 5 | The minutes, with the speaker-labelled transcript as an appendix |
| `work/` | all | Reference scripts as they were run, and logs |

## Workflow

| Phase | Read | Result |
|---|---|---|
| 1. Setup | `SETUP.md` | A working environment, and a decision on GPU or CPU and on speaker separation |
| 2. Transcription | `TRANSCRIPTION.md` | `transcript.json`, `transcript.txt` |
| 3. Speaker separation | `DIARIZATION.md` | `diarization.json` |
| 4. Speakers and corrections | `SPEAKERS.md` | `speakers.json`, `corrections.tsv`, `transcript-speakers.txt` |
| 5. Minutes | `MINUTES.md` | `minutes.md` |

Phase 1 is done once per computer; later runs only verify it. Phases 2 and 3 are long: use a background run and continue when it ends. Phase 3 is skipped when the user declines the one-time model access; the rest still works, without names.

Check each phase's result before starting the next one. Each file ends with a "Check" section.

## Reporting to the user

When the run ends, tell the user:

- where `minutes.md` is;
- which speaker names are confirmed and which are guesses;
- what was skipped or changed from the defaults, and why;
- the known weak points of this run (unclear names or numbers, overlapping speech, a stretch of poor audio).

## Anti-patterns

- Summarizing notes written by a meeting bot when the recording is available. Such notes drop stretches of the meeting and merge all voices into one.
- Running transcription and speaker separation at the same time "to save time". It slows both and can exhaust GPU memory.
- Giving the speech model a list of attendee names as a hint. It then writes those names into silent stretches and drops real speech.
- Naming a speaker because a name "fits". One wrong attribution in shared minutes costs more than ten "Speaker 3" labels.
- Fixing the transcript by rewriting sentences. Correct recurring mis-heard words through `corrections.tsv`; leave the wording alone.
- Writing an action item with an owner and a date that nobody stated.
- Reporting success without opening the result. Read the first and last minute of the transcript and compare the speaker count with the attendee count.
