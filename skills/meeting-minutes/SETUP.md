# Phase 1: Setup

Goal: one Python environment with the speech models' libraries, and two decisions: GPU or CPU, and speaker separation or not.

Do this once per computer. On later runs, go straight to "Check".

## 1. Look at the machine

Find out, with the shell:

- operating system and CPU architecture;
- whether an NVIDIA GPU is present and how much memory it has (`nvidia-smi --query-gpu=name,memory.total --format=csv,noheader`; a missing command means no usable NVIDIA GPU);
- free disk space where the environment and model cache will live. About 10 GB is needed;
- which Python tooling exists: `uv --version`, `python --version` or `python3 --version`.

Decide from what you find:

| Machine | Speech-to-text runs on | Speaker separation runs on | Expect for a one-hour recording |
|---|---|---|---|
| NVIDIA GPU, 6 GB or more | GPU, `float16` | GPU | 15 to 40 minutes in total |
| NVIDIA GPU, 4 to 6 GB | GPU, `int8_float16` | GPU | somewhat slower |
| Apple Silicon | CPU, `int8` | CPU (`mps` may work; fall back to CPU on any error) | one to a few hours |
| No GPU | CPU, `int8` | CPU | several hours. Offer `large-v3-turbo`, or processing overnight |

Tell the user which row applies before starting a long run.

## 2. Create the environment

Any tool that creates an isolated Python 3.10 to 3.12 environment works. [uv](https://docs.astral.sh/uv/) is the simplest because it also downloads Python. If neither uv nor a suitable Python is installed, show the user the official uv install command for their system and run it only after they agree.

Keep the environment outside the skill directory and outside the recording's folder, for example `~/.cache/meeting-minutes/venv`. Below, `<env>` is that folder and `<python>` is its interpreter (`<env>/bin/python`, or `<env>\Scripts\python.exe` on Windows).

Install in this order. PyTorch goes first so that the right build is in place before the other packages ask for it.

**a. Create it**

```
uv venv <env> --python 3.12
```

Without uv: `python -m venv <env>`, then use `<python> -m pip install` wherever the next steps say `uv pip install --python <python>`.

**b. PyTorch**

- Windows or Linux on x86-64 (with or without an NVIDIA GPU): take the CUDA build. It bundles the GPU libraries that the speech-to-text engine reuses, so the user does not install CUDA separately, and it still runs on CPU.

  ```
  uv pip install --python <python> torch torchaudio --index-url https://download.pytorch.org/whl/cu126
  ```

  If `nvidia-smi` shows a driver too old for this build, pick the matching index from https://pytorch.org/get-started/locally/.

- macOS, or Linux on ARM: the default build.

  ```
  uv pip install --python <python> torch torchaudio
  ```

**c. Speech libraries**

```
uv pip install --python <python> faster-whisper "av>=12,<14" pyannote.audio
```

`av` is pinned because newer releases removed an argument that faster-whisper passes. If the versions you get no longer need the pin, drop it.

The first install downloads several GB. Run it as a background run when you can.

## 3. Speaker separation needs a one-time model access

The speaker model (`pyannote/speaker-diarization-community-1`) is free, but its authors require each user to accept its terms on Hugging Face. Explain this and let the user choose:

- **Set it up** (a few minutes, once), or
- **Skip speaker separation**. The transcript then has no speaker labels, and the minutes cannot say who said what.

If the user sets it up, they do these steps themselves in a browser:

1. Sign in at https://huggingface.co (a free account).
2. Open https://huggingface.co/pyannote/speaker-diarization-community-1, fill in the short form, and press "Agree and access repository". This shares the account's email, user name, and the affiliation they type with the model's authors, who may send occasional mail.
3. Create a token at https://huggingface.co/settings/tokens. A fine-grained token with only "Read access to contents of gated repos you can access" is enough.
4. Store the token on this computer: set the environment variable `HF_TOKEN`, or write the token as the only content of `~/.cache/huggingface/token`.

The token only authorizes downloading the model files. After the first download the model runs offline. Do not read the token back, print it, or ask for it in the conversation; check only that it exists.

## Check

Run with `<python>`:

```python
import os, platform
print(platform.system(), platform.machine(), platform.python_version())
import torch
print("torch", torch.__version__, "cuda", torch.cuda.is_available())
if torch.cuda.is_available():
    p = torch.cuda.get_device_properties(0)
    print("gpu", p.name, round(p.total_memory / 1024**3, 1), "GB")
import faster_whisper, pyannote.audio
print("faster-whisper", faster_whisper.__version__, "pyannote.audio", pyannote.audio.__version__)
from huggingface_hub import get_token
print("hugging face token:", "found" if (os.environ.get("HF_TOKEN") or get_token()) else "not found")
```

- Every import succeeds.
- `cuda True` on a machine with an NVIDIA GPU. If it prints `False` there, the CPU build of PyTorch was installed: reinstall step b with the CUDA index.
- A warning that mentions `torchcodec` is harmless. The reference scripts decode audio another way.
- `hugging face token: not found` is fine only if the user chose to skip speaker separation.

Record the outcome (GPU or CPU, speaker separation on or off) so that later phases and later runs do not repeat the investigation.
