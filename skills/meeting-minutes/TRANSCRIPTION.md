# Phase 2: Transcription

Goal: `transcript.json` with word-level timestamps and `transcript.txt`, both inside `<out>`.

The engine is [faster-whisper](https://github.com/SYSTRAN/faster-whisper), a fast implementation of OpenAI's Whisper. It reads common audio and video formats directly; ffmpeg is not needed.

## Choices to make

| Choice | Default | Change it when |
|---|---|---|
| Model | `large-v3` | The machine has no GPU or little time: `large-v3-turbo` is several times faster and a little less accurate. Smaller models lose too much on meetings |
| Language | detect | The user knows it. Stating it avoids a wrong guess on a quiet opening |
| Context prompt | none | The meeting is full of jargon. Give one or two sentences: the topic and a few technical terms |
| Precision | from `SETUP.md` | GPU memory runs out: `int8_float16`. CPU: `int8` |
| Range | whole file | A trial run: process the first two or three minutes before committing to the full length |

**Never put people's names in the context prompt or in a hotword list.** The model then writes those names into silent stretches and drops real speech around them. Names are corrected later, in Phase 4.

Run a short trial first on a new machine or an unusual recording. It costs a minute and catches a broken setup, a wrong language, or unusable audio before an hour-long run.

## Reference script

Save as `<out>/work/transcribe.py` and run with `<python>`. Start it as a background run for anything longer than a few minutes.

```
<python> <out>/work/transcribe.py <recording> <out> --language ko
```

```python
"""Speech to text with word timestamps. Usage: transcribe.py AUDIO OUT_DIR [options]"""
import argparse, json, os, sys, time

ap = argparse.ArgumentParser()
ap.add_argument("audio")
ap.add_argument("out")
ap.add_argument("--model", default="large-v3")
ap.add_argument("--language", default=None, help="for example ko or en; omit to detect")
ap.add_argument("--prompt", default=None, help="topic and key terms, one or two sentences. No people's names")
ap.add_argument("--compute-type", default="auto")
ap.add_argument("--threads", type=int, default=4)
ap.add_argument("--start", type=float, default=None, help="seconds; for a trial run")
ap.add_argument("--end", type=float, default=None)
args = ap.parse_args()

for s in (sys.stdout, sys.stderr):
    try:
        s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass

# Low priority: slower, but the computer stays usable.
try:
    if os.name == "nt":
        import ctypes
        ctypes.windll.kernel32.SetPriorityClass(ctypes.windll.kernel32.GetCurrentProcess(), 0x40)
    else:
        os.nice(15)
except Exception:
    pass

# Import torch first when it is installed: that loads the GPU libraries faster-whisper reuses.
device, vram = "cpu", 0.0
try:
    import torch
    if os.name == "nt":
        lib = os.path.join(os.path.dirname(torch.__file__), "lib")
        os.add_dll_directory(lib)
        os.environ["PATH"] = lib + os.pathsep + os.environ.get("PATH", "")
    if torch.cuda.is_available():
        device = "cuda"
        vram = torch.cuda.get_device_properties(0).total_memory / 1024**3
except Exception as e:
    print("torch not usable, running on CPU:", repr(e)[:200], flush=True)

from faster_whisper import WhisperModel
from faster_whisper.audio import decode_audio

compute = args.compute_type
if compute == "auto":
    compute = ("float16" if vram >= 5.5 else "int8_float16") if device == "cuda" else "int8"
try:
    model = WhisperModel(args.model, device=device, compute_type=compute, cpu_threads=args.threads)
except Exception as e:
    if device != "cuda":
        raise
    print("GPU not usable, falling back to CPU:", repr(e)[:200], flush=True)
    device, compute = "cpu", "int8"
    model = WhisperModel(args.model, device=device, compute_type=compute, cpu_threads=args.threads)
print(f"model={args.model} device={device} compute={compute}", flush=True)

audio = decode_audio(args.audio, sampling_rate=16000)
offset = float(args.start or 0)
if args.start or args.end:
    audio = audio[int(offset * 16000): int(args.end * 16000) if args.end else len(audio)]
print(f"audio {len(audio) / 16000 / 60:.1f} min", flush=True)

t0 = time.time()
segments, info = model.transcribe(
    audio,
    language=args.language,
    beam_size=5,
    vad_filter=True,                         # skip silence
    vad_parameters=dict(min_silence_duration_ms=500),
    condition_on_previous_text=False,        # stops one error from snowballing into repeated text
    word_timestamps=True,                    # needed to attach speakers to words
    initial_prompt=args.prompt,
)

def ts(x):
    x = int(x)
    return f"{x // 3600:02d}:{x % 3600 // 60:02d}:{x % 60:02d}"

os.makedirs(args.out, exist_ok=True)
result, lines, mark = [], [], -1
for s in segments:
    words = [{"start": round(w.start + offset, 2), "end": round(w.end + offset, 2), "word": w.word} for w in (s.words or [])]
    result.append({"start": round(s.start + offset, 2), "end": round(s.end + offset, 2), "text": s.text.strip(), "words": words})
    lines.append(f"[{ts(s.start + offset)}] {s.text.strip()}")
    if int(s.start) // 300 != mark:
        mark = int(s.start) // 300
        print(f"at {ts(s.start + offset)} elapsed {int(time.time() - t0)}s", flush=True)

with open(os.path.join(args.out, "transcript.json"), "w", encoding="utf-8") as f:
    json.dump({"audio": os.path.basename(args.audio), "model": args.model, "language": info.language,
               "duration": round(len(audio) / 16000, 1), "segments": result}, f, ensure_ascii=False)
with open(os.path.join(args.out, "transcript.txt"), "w", encoding="utf-8", newline="\n") as f:
    f.write("\n".join(lines) + "\n")
print(f"done: {len(result)} segments, language={info.language}, {int(time.time() - t0)}s", flush=True)
```

The first run downloads the model (about 3 GB for `large-v3`) into the Hugging Face cache. No token is needed for it. Set `HF_HUB_OFFLINE=1` on later runs to guarantee that nothing is fetched.

## `transcript.json`

```
{ "audio": "...", "model": "...", "language": "ko", "duration": 4680.0,
  "segments": [ { "start": 81.2, "end": 86.9, "text": "...",
                  "words": [ { "start": 81.2, "end": 81.6, "word": " ..." }, ... ] }, ... ] }
```

Times are seconds from the start of the recording.

## Check

- The script printed `done` and both files exist.
- The device line matches the decision from `SETUP.md`. `device=cpu` on a GPU machine means the setup is wrong; fix it before a long run.
- Read the first and the last minute of `transcript.txt`. The text is in the right language and reads like the meeting.
- Scan the whole file for hallucination: the same sentence repeated many times, a line of bare names, or stock phrases that belong to broadcasts or video captions. If you find them, remove the context prompt (or shorten it) and transcribe again.
- Compare coverage with the duration: long gaps between timestamps should match real silence, not missing speech.

## When the result is poor

| Symptom | Likely cause | Try |
|---|---|---|
| Repeated or invented sentences | Context prompt too long or naming people; noisy silence | Drop the prompt; keep `vad_filter` on |
| Wrong language, or mixed scripts | Detection guessed from a quiet opening | State `--language` |
| Many wrong technical terms | Unknown jargon | A short context prompt with the terms; fix the rest through `corrections.tsv` in Phase 4 |
| Out of GPU memory | Model too large for the card | `--compute-type int8_float16`, or `large-v3-turbo` |
| Far too slow | Running on CPU | `large-v3-turbo`, or run overnight |
