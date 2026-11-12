# Phase 3: Speaker separation

Goal: `diarization.json` inside `<out>`: who spoke when, with speakers numbered from 1.

The engine is [pyannote.audio](https://github.com/pyannote/pyannote-audio) with the `pyannote/speaker-diarization-community-1` pipeline. It answers "which voice is this" and knows nothing about names; names come in Phase 4.

Skip this phase when the user declined the model access in `SETUP.md`. Phase 4 then produces a transcript without speaker labels.

Run this phase only after transcription has finished. Never run both at once.

## Speaker count

The pipeline counts speakers by itself, and it counts better when told what to expect.

| You know | Pass |
|---|---|
| The exact number of people who spoke | `--num-speakers N` |
| Roughly | `--min-speakers A --max-speakers B` |
| Nothing | nothing |

Attendee lists overstate the count when someone stayed silent, so prefer a range unless the user is sure.

## Reference script

Save as `<out>/work/diarize.py` and run with `<python>`, as a background run for long recordings.

```
<python> <out>/work/diarize.py <recording> <out> --min-speakers 3 --max-speakers 6
```

```python
"""Who spoke when. Usage: diarize.py AUDIO OUT_DIR [options]"""
import os
os.environ.setdefault("PYANNOTE_METRICS_ENABLED", "0")  # no usage statistics
os.environ.setdefault("HF_HUB_DISABLE_TELEMETRY", "1")

import argparse, json, sys, time, warnings

ap = argparse.ArgumentParser()
ap.add_argument("audio")
ap.add_argument("out")
ap.add_argument("--num-speakers", type=int, default=None)
ap.add_argument("--min-speakers", type=int, default=None)
ap.add_argument("--max-speakers", type=int, default=None)
ap.add_argument("--device", default="auto", help="auto, cuda, cpu or mps")
ap.add_argument("--threads", type=int, default=4)
ap.add_argument("--start", type=float, default=None)
ap.add_argument("--end", type=float, default=None)
args = ap.parse_args()

for s in (sys.stdout, sys.stderr):
    try:
        s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass

try:  # low priority
    if os.name == "nt":
        import ctypes
        ctypes.windll.kernel32.SetPriorityClass(ctypes.windll.kernel32.GetCurrentProcess(), 0x40)
    else:
        os.nice(15)
except Exception:
    pass

warnings.filterwarnings("ignore")
import torch
torch.set_num_threads(args.threads)
from faster_whisper.audio import decode_audio
from pyannote.audio import Pipeline

MODEL = "pyannote/speaker-diarization-community-1"
try:
    pipeline = Pipeline.from_pretrained(MODEL)
    if pipeline is None:
        raise RuntimeError("access denied")
except Exception as e:
    print("could not load", MODEL, "-", repr(e)[:300], flush=True)
    print("The user must accept the model terms and store a Hugging Face token (see SETUP.md).", flush=True)
    sys.exit(3)

device = args.device
if device == "auto":
    device = "cuda" if torch.cuda.is_available() else "cpu"
pipeline.to(torch.device(device))
print(f"model={MODEL} device={device}", flush=True)

# Decode in memory, so the pipeline's own audio backend is not needed.
audio = decode_audio(args.audio, sampling_rate=16000)
offset = float(args.start or 0)
if args.start or args.end:
    audio = audio[int(offset * 16000): int(args.end * 16000) if args.end else len(audio)]
wave = torch.from_numpy(audio).unsqueeze(0)

options = {k: v for k, v in (("num_speakers", args.num_speakers), ("min_speakers", args.min_speakers),
                             ("max_speakers", args.max_speakers)) if v}
t0 = time.time()
output = pipeline({"waveform": wave, "sample_rate": 16000}, **options)

full = getattr(output, "speaker_diarization", output)
exclusive = getattr(output, "exclusive_speaker_diarization", None)
annotation = exclusive if exclusive is not None else full   # no overlaps: simpler to align with words
number = {label: i + 1 for i, label in enumerate(sorted(full.labels()))}
turns = [{"start": round(seg.start + offset, 2), "end": round(seg.end + offset, 2), "speaker": number[label]}
         for seg, _, label in annotation.itertracks(yield_label=True)]

os.makedirs(args.out, exist_ok=True)
with open(os.path.join(args.out, "diarization.json"), "w", encoding="utf-8") as f:
    json.dump({"model": MODEL, "options": options, "turns": turns}, f)

talk = {}
for t in turns:
    talk[t["speaker"]] = talk.get(t["speaker"], 0) + t["end"] - t["start"]
for spk, sec in sorted(talk.items(), key=lambda x: -x[1]):
    print(f"Speaker {spk}: {sec / 60:.1f} min", flush=True)
print(f"done: {len(talk)} speakers, {len(turns)} turns, {int(time.time() - t0)}s", flush=True)
```

Exit code 3 means the model could not be loaded. Give the user the steps from `SETUP.md`, or continue without speaker labels if they prefer.

## `diarization.json`

```
{ "model": "...", "options": { "min_speakers": 3, "max_speakers": 6 },
  "turns": [ { "start": 81.2, "end": 93.4, "speaker": 2 }, ... ] }
```

## Check

- The script printed `done` and the device matches the decision from `SETUP.md`.
- The speaker count is plausible against the attendee list. One more or one fewer is common; twice as many is a failure.
- No single "speaker" holds almost all the talk time in a meeting that was a real discussion.

## When the result is poor

| Symptom | Meaning | Try |
|---|---|---|
| Far too many speakers, some with seconds of talk | One voice split into several | Run again with `--num-speakers` or a tighter `--max-speakers` |
| Too few speakers | Similar voices merged | Run again with `--num-speakers` or a higher `--min-speakers` |
| Labels flip inside one person's long turn | Poor audio: one distant microphone, echo, crosstalk | Accept it and mark uncertain passages in Phase 4. Do not invent certainty |
| Very slow | Running on CPU | Check the PyTorch build in `SETUP.md`. On CPU, a one-hour recording can take an hour |

Speaker separation is the least reliable step. Overlapping speech, short interjections, and the first words after a change of speaker are where it goes wrong. Carry that uncertainty forward instead of hiding it.
