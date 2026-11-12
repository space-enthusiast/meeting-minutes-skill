# Phase 4: Speakers and corrections

Goal: `transcript-speakers.txt`, the speaker-labelled and corrected transcript that the minutes are written from, together with `speakers.json` and `corrections.tsv`.

This phase has three steps: merge, name the speakers, correct recurring errors. The merge is run again after each of the other two.

## Step 1: Merge

Every word gets the speaker who was talking at that word's midpoint. Consecutive words of one speaker become one utterance. An interjection of one or two words inside someone else's turn is absorbed, because such fragments are mostly alignment noise.

Save as `<out>/work/merge.py` and run with `<python>` (it uses only the standard library, so any Python works):

```
<python> <out>/work/merge.py <out>
```

```python
"""Combine transcript.json and diarization.json into transcript-speakers.txt. Usage: merge.py OUT_DIR"""
import bisect, json, os, sys

out = sys.argv[1]
MIN_WORDS = 3        # shorter interjections inside another speaker's turn are absorbed
MAX_SECONDS = 60     # a longer monologue is split at the next segment boundary

def load(name):
    p = os.path.join(out, name)
    return json.load(open(p, encoding="utf-8")) if os.path.exists(p) else None

transcript = load("transcript.json")
diar = load("diarization.json")
turns = sorted(diar["turns"], key=lambda t: t["start"]) if diar else []
starts = [t["start"] for t in turns]
names = {str(k): v for k, v in (load("speakers.json") or {}).items()}

fixes = []
cpath = os.path.join(out, "corrections.tsv")
if os.path.exists(cpath):
    for line in open(cpath, encoding="utf-8"):
        line = line.rstrip("\r\n")
        if line and not line.startswith("#") and "\t" in line:
            wrong, right = line.split("\t", 1)
            if wrong:
                fixes.append((wrong, right))

def speaker_at(t):
    """Speaker at time t; outside every turn, the nearest turn within one second."""
    i = bisect.bisect_right(starts, t) - 1
    best, dist = None, 1.0
    for j in (i - 1, i, i + 1):
        if 0 <= j < len(turns):
            a, b = turns[j]["start"], turns[j]["end"]
            if a <= t <= b:
                return turns[j]["speaker"]
            d = min(abs(t - a), abs(t - b))
            if d < dist:
                best, dist = turns[j]["speaker"], d
    return best

words = []
for seg in transcript["segments"]:
    ws = seg["words"] or [{"start": seg["start"], "end": seg["end"], "word": " " + seg["text"]}]
    for k, w in enumerate(ws):
        spk = speaker_at((w["start"] + w["end"]) / 2) if turns else 0
        words.append({"t": w["start"], "w": w["word"], "spk": spk, "first": k == 0})

prev = None                      # words that fell outside every turn take a neighbour's speaker
for w in words:
    if w["spk"] is None:
        w["spk"] = prev
    prev = w["spk"]
nxt = None
for w in reversed(words):
    if w["spk"] is None:
        w["spk"] = nxt
    nxt = w["spk"]

runs = []
for i, w in enumerate(words):
    if runs and runs[-1][0] == w["spk"]:
        runs[-1][2] = i
    else:
        runs.append([w["spk"], i, i])
for k in range(1, len(runs) - 1):
    spk, a, b = runs[k]
    if b - a + 1 < MIN_WORDS and runs[k - 1][0] == runs[k + 1][0]:
        for i in range(a, b + 1):
            words[i]["spk"] = runs[k - 1][0]

utts = []
for w in words:
    if utts and utts[-1]["spk"] == w["spk"] and not (w["first"] and w["t"] - utts[-1]["start"] > MAX_SECONDS):
        utts[-1]["text"] += w["w"]
    else:
        utts.append({"spk": w["spk"], "start": w["t"], "text": w["w"]})
for u in utts:
    text = " ".join(u["text"].split())
    for wrong, right in fixes:
        text = text.replace(wrong, right)
    u["text"] = text
utts = [u for u in utts if u["text"]]

def ts(x):
    x = int(x)
    return f"{x // 3600:02d}:{x % 3600 // 60:02d}:{x % 60:02d}"

def label(spk):
    return names.get(str(spk), f"Speaker {spk}") if turns else ""

def line(u):
    who = label(u["spk"])
    return f"[{ts(u['start'])}] {who}: {u['text']}" if who else f"[{ts(u['start'])}] {u['text']}"

with open(os.path.join(out, "transcript-speakers.txt"), "w", encoding="utf-8", newline="\n") as f:
    f.write("\n".join(line(u) for u in utts) + "\n")

# Samples per speaker, to help put names on numbers.
stats = {}
for u in utts:
    s = stats.setdefault(u["spk"], [0, 0])
    s[0] += 1
    s[1] += len(u["text"])
sample = []
for spk, (n, chars) in sorted(stats.items(), key=lambda x: -x[1][1]):
    mine = [u for u in utts if u["spk"] == spk]
    picked = {id(u): u for u in mine[:3]}
    for u in sorted(mine, key=lambda u: -len(u["text"]))[:8]:
        picked[id(u)] = u
    sample.append(f"=== {label(spk) or 'Transcript'}: {n} utterances, {chars} characters ===")
    sample += [f"[{ts(u['start'])}] {u['text'][:400]}" for u in sorted(picked.values(), key=lambda u: u["start"])]
    sample.append("")
os.makedirs(os.path.join(out, "work"), exist_ok=True)
with open(os.path.join(out, "work", "speaker-samples.txt"), "w", encoding="utf-8", newline="\n") as f:
    f.write("\n".join(sample))

print(f"{len(utts)} utterances, {len(stats)} speakers, {len(fixes)} corrections")
for spk, (n, chars) in sorted(stats.items(), key=lambda x: -x[1][1]):
    print(f"  {label(spk) or 'Transcript'}: {n} utterances, {chars} characters")
```

Lines in `transcript-speakers.txt` look like this:

```
[00:14:57] Speaker 4: ...
[00:16:06] Speaker 2: ...
```

## Step 2: Put names on speakers

Read `work/speaker-samples.txt`, then read `transcript-speakers.txt` for the passages that decide the question. Work out who each number is from evidence in the text.

Evidence, strongest first:

1. **Self-identification**: "I'm ... from the sales team", "as the one who built this".
2. **Being addressed**: someone says a name or title and the next speaker answers.
3. **Role and knowledge**: who presents, who owns a number, who speaks for which team. Useful only together with an attendee list.
4. **Elimination**: one name and one number are left. This is the weakest evidence.

Rules:

- A name needs evidence of kind 1 or 2, or several consistent signs of kind 3. Otherwise mark it as a guess, in the meeting's language: `Alice Kim (?)`, `김영희(추정)`.
- With no usable evidence, keep `Speaker N`.
- If the user is available, show the samples and your reasoning for the uncertain ones and ask. One answer from someone who was in the room beats any inference.
- Two numbers that are clearly one person (the same monologue continues across them) get the same name. Two people merged into one number cannot be repaired here: go back to Phase 3 with a speaker count.
- The person who recorded the meeting is usually the one who opens or closes it, but check; do not assume.

Write `<out>/speakers.json`, keyed by speaker number:

```json
{ "1": "Alice Kim", "2": "Bob Lee (?)", "3": "Speaker 3" }
```

Run the merge again.

## Step 3: Correct recurring recognition errors

Read through `transcript-speakers.txt` and collect words the model keeps getting wrong: product names, team names, jargon, a person's name heard three different ways.

Write `<out>/corrections.tsv`, one pair per line, the wrong form, a tab, the right form:

```
sass model	SaaS model
jon smyth	John Smith
```

Rules:

- Add a pair only when the context leaves no doubt about the intended word. A plausible guess is not enough.
- Put longer phrases before shorter ones; replacements are applied in file order.
- Prefer phrases to single short words, which may also occur correctly elsewhere.
- Do not rewrite sentences, fix grammar, or remove filler. The transcript stays a record of what was said.
- `transcript.txt` and `transcript.json` are never edited. They remain the uncorrected record.

Run the merge again.

## Check

- `transcript-speakers.txt` exists and every line has a timestamp.
- Each name in `speakers.json` has evidence you can point to, or carries a mark of uncertainty.
- Spot-check three places where the speaker changes: does the text on each side fit the person named?
- The corrections did not damage correct text: search the transcript for each right-hand form and read a few hits.

## What stays uncertain

- The first few words after a change of speaker may be attached to the previous speaker.
- Short replies ("yes", "right") are often assigned to whoever was talking around them.
- Overlapping speech is assigned to one person.

Mention these limits when you report, and do not quote a short reply as proof that a particular person agreed to something.
