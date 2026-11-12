# meeting-minutes

A coding-agent skill that turns a meeting recording into minutes on your own computer. Whisper transcribes, pyannote separates the speakers, and your agent writes the minutes from the speaker-labelled transcript. The audio never leaves the machine.

회의 녹음을 내 컴퓨터 안에서 전사하고 화자를 나눈 뒤, 화자가 붙은 전사본으로 회의록을 쓰는 에이전트 스킬이다. 녹음 파일은 밖으로 나가지 않는다.

## What it does

The skill runs five phases:

1. **Setup** -- look at the machine, create one Python environment, decide GPU or CPU, and arrange the one-time access to the speaker model. Done once per computer.
2. **Transcription** -- faster-whisper `large-v3` writes `transcript.json` with word-level timestamps.
3. **Speaker separation** -- pyannote `speaker-diarization-community-1` writes `diarization.json`: who spoke when.
4. **Speakers and corrections** -- merge words and speakers, put names on speaker numbers from evidence in the text, and fix recurring recognition errors. The result is `transcript-speakers.txt`.
5. **Minutes** -- the agent reads the whole transcript and writes `minutes.md`: summary, discussion by topic and speaker, decisions, action items, open questions, and the transcript as an appendix.

The skill is a set of instructions, not a program. It contains no package to install and no fixed command line. The agent reads the files, looks at the machine it is on, and decides how to set things up. The speech steps come with tested reference scripts inside the Markdown, which the agent runs as they are or adapts.

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Operating modes, principles, setup values, outputs, workflow overview, and anti-patterns |
| `SETUP.md` | Phase 1: inspecting the machine, creating the environment, PyTorch build choice, Hugging Face access |
| `TRANSCRIPTION.md` | Phase 2: model and parameter choices, reference script, checks, and fixes for poor results |
| `DIARIZATION.md` | Phase 3: speaker count, reference script, checks, and what to do when speakers are split or merged |
| `SPEAKERS.md` | Phase 4: merge script, rules for naming speakers, correction list |
| `MINUTES.md` | Phase 5: writing rules, minutes structure, handing the transcript to a different model |

## Installation

Install the skill with the [Skills CLI](https://skills.sh):

```bash
npx skills add https://github.com/space-enthusiast/meeting-minutes-skill \
  --skill meeting-minutes
```

Use `--global` for a user-level installation. Run `npx skills --help` for agent-selection options.

Or copy the folder `skills/meeting-minutes` into your agent's skills directory by hand (for Claude Code: `~/.claude/skills/meeting-minutes`).

Nothing else is installed at this point. On first use, the agent sets up the speech libraries by following `SETUP.md`, and asks before installing anything.

## Usage

Give your agent a recording and ask for minutes:

```
make meeting minutes from ~/recordings/weekly.m4a
attendees: Alice, Bob, Carol. the meeting was in Korean.
```

```
이 녹음으로 회의록 만들어줘: C:\meetings\weekly.m4a
```

```
just transcribe this, with speakers: interview.mp3
```

The results land in a folder next to the recording, named after it with `-minutes` appended.

## Requirements

- A coding agent that can run shell commands and read and write files
- Windows, macOS, or Linux, with about 10 GB of free disk space for the libraries and models
- An NVIDIA GPU with 6 GB or more is recommended. Without one everything runs on the CPU: a one-hour recording then takes hours instead of 15 to 40 minutes
- For speaker separation: a free Hugging Face account, to accept the speaker model's terms once. Without it the skill still transcribes, without speaker labels

## Privacy

- The recording is processed locally. No part of the audio is uploaded.
- The transcript is text, and the agent that writes the minutes reads it. With a cloud model, the text goes to that provider. For a confidential meeting, use a local model for the writing phase; `MINUTES.md` describes the hand-over.
- Accepting the speaker model's terms shares your Hugging Face email, user name, and stated affiliation with its authors. The access token is used only to download the model.

## Design principles

- **The recording is the source of truth.** Notes from a meeting bot are a cross-check, not a basis.
- **Only what was said.** No owner, date, number, or decision that the transcript does not support.
- **Honest speaker labels.** A speaker number becomes a name only on evidence; a guess is marked as a guess.
- **One heavy job at a time, at low priority.** The user still has to work on the computer.
- **The agent decides.** Machines differ. The skill explains the choices and the checks instead of hiding them in a script.

## Limits

- Names, numbers, and jargon are where speech recognition fails most. Verify the ones that matter against the recording.
- Overlapping speech, short replies, and the first words after a change of speaker are often attributed to the wrong person.
- Tested on Windows with an NVIDIA GPU. macOS, Linux, and CPU-only machines follow the same instructions but have not been tested yet; reports are welcome.

## License

MIT -- see [LICENSE](LICENSE).

Built on [faster-whisper](https://github.com/SYSTRAN/faster-whisper) (MIT; Whisper models by OpenAI, MIT) and [pyannote.audio](https://github.com/pyannote/pyannote-audio) (MIT; the `speaker-diarization-community-1` model is CC-BY-4.0).
