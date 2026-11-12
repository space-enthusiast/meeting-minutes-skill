# Phase 5: Minutes

Goal: `<out>/minutes.md`: minutes written from `transcript-speakers.txt`, followed by that transcript as an appendix.

Any language model can do this phase. The rules below do not depend on a particular one.

## Before writing

Read `transcript-speakers.txt` from the first line to the last. If it does not fit in one read, read it in consecutive parts and take notes per part; do not sample, and do not skip the middle. Meetings change direction late: a decision in minute 50 can overturn a plan from minute 10.

While reading, note:

- topics, in the order they came up;
- who argued what, with the reason given;
- every number, with its timestamp;
- what was agreed, and by whom;
- tasks someone took on, and any date mentioned;
- what was raised and left open.

## Rules

1. **Only what the transcript says.** Do not add facts, numbers, names, decisions, or deadlines.
2. **Attribute.** Say who said what when the speaker label is reliable. Keep uncertainty marks from `speakers.json`. For a passage where the label is doubtful, attribute to a team or write "a participant".
3. **Owners and dates only when stated.** If nobody named an owner or a date, write that it was not stated. Do not assign work by inference.
4. **Numbers as spoken.** When two different figures were given for the same thing, report both with their timestamps and do not pick one.
5. **Decided is not the same as discussed.** A proposal that nobody accepted is not a decision. A decision someone objected to is recorded with the objection.
6. **Carry the reasoning.** For each decision give the reason, the conditions, and the alternatives that were rejected. A reader who was absent should understand why.
7. **Recognition errors.** Where the intended word is obvious from context, use it. Where it is not, quote what was heard and mark it as unclear.
8. **No transcript in the body.** Short quotations are fine when the exact wording matters.
9. **Small talk stays out**, except what belongs on the record; put that in a short closing section.
10. **Language.** Write in the language of the meeting. Translate the section headings below into it.

## Structure

```
# [YYYY.MM.DD] <meeting title> - Minutes

- **Date and time**: <as given by the user or the file; otherwise "not stated">
- **Attendees**: <names, with team or role when known>
- **Purpose**: <one or two sentences>
- **Source**: <for example: audio recording, 78 min, transcribed and speaker-separated locally>
- **Speaker labels**: <which names are confirmed, which are guesses, and on what evidence>

## 1. Summary
<Five to eight numbered points, one or two full sentences each. A reader of this section alone knows the outcome.>

## 2. Discussion
<One subsection per topic, in the order discussed. Bullets of the form
"- <Speaker>: <position, with the reason or condition>".>

## 3. Decisions
<Numbered. Only what the meeting agreed on.>

## 4. Action items
| Owner | Action | Due |
|---|---|---|
| <name, or "not stated"> | <what> | <date, or "not stated"> |

## 5. Open questions
<Bullets. Raised and not resolved.>

## 6. Other notes
<Optional: off-topic remarks worth keeping.>

## Appendix: Transcript
<The full content of transcript-speakers.txt, followed by one line on its limits:
automatic transcription; names, numbers and speaker changes may be wrong.>
```

Add a glossary after the header when the meeting relies on abbreviations or internal names a later reader may not know. Adapt the section list to the meeting: a status meeting may need no "Decisions"; a negotiation may need a section per party. Keep "Summary" first and the transcript last.

If the user or their team already has a minutes format, follow theirs and keep these rules.

## Writing it with a different model

When the minutes are to be written by another model (a local one for confidentiality, or the user's preferred one), hand over a single self-contained prompt:

1. the "Rules" and "Structure" sections of this file;
2. the meeting information supplied by the user (title, date, attendees, purpose);
3. the full `transcript-speakers.txt`.

Save it as `<out>/work/llm-prompt.md` so the user can paste it into any chat window or pass it to any command-line tool. Put the answer in `minutes.md` and append the transcript.

If the transcript exceeds that model's context, split it at speaker changes into consecutive parts. Ask for detailed notes per part (topics, who said what, numbers, decisions, tasks, open points, with timestamps), then ask for the minutes from the combined notes. Never drop a part to make it fit.

## Check

- Every decision and action item can be traced to a timestamp in the appendix. Check each one.
- Every number in the minutes appears in the transcript with the same value.
- No owner or due date appears that nobody stated.
- Names in the minutes match `speakers.json`, uncertainty marks included.
- The header says where the minutes came from and how reliable the speaker labels are.
- A cold reader test: read only the Summary. Does it state the outcome, or does it merely list topics? Rewrite it if it only lists topics.
