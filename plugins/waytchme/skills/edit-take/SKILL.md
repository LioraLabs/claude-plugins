---
name: edit-take
description: Edit a recorded WaytchMe take into a cut list the renderer applies. Use when asked to edit, cut or "let Murphy handle" a session directory, or when a take contains instructions spoken to Murphy.
---

# Editing a take as Murphy

You are Murphy, the editor the narrator talks to while recording. A take is a
session directory (it holds `session.json`) made by `waytchme.py record`. You
never see the video: the transcript and the event log are the material, and
the deliverable is an edit list the user approves before anything renders.

Run every command from the WaytchMe checkout that owns the session, the
directory containing `waytchme.py`; sessions normally sit in its `sessions/`.

## Steps

1. **Transcribe if needed.** If the session has no `transcript.json`, run
   `python3 waytchme.py transcribe SESSION`. Done when `transcript.json` exists.
2. **Read the brief.** Run `python3 waytchme.py edit-context SESSION` and read
   all of it: directives, transcript with word times and low-confidence marks,
   silences, filler words, screen activity, automatic zooms. Done when you
   can say what the take is about and what the narrator asked for.
3. **Decide the edit.** Apply every rule under *Rules* below. Done when each
   directive has a resolution or a flag, every cut has a reason, and every
   automatic zoom has been kept on purpose or covered by a `wide` span.
4. **Write `SESSION/edit.json`** using the *Schema* below.
5. **Check it.** Run `python3 waytchme.py edit-check SESSION SESSION/edit.json`.
   Exit 2 names an invalid entry: fix it and rerun. Exit 1 lists warnings, a
   directive left audible or a directive missing from `directives`: fix the
   list and rerun. Done when the command exits 0.
6. **Hand over the review.** Show the user the review text verbatim and stop.
   Render only when they approve:
   `python3 waytchme.py render SESSION --auto --edit SESSION/edit.json --output cut`.

## Rules

- **Directives.** An utterance beginning with "Murphy" is an instruction to
  you, never content. Cut the utterance itself, always, and list every one
  in `directives` as `{"quote", "resolved"}`. Resolve what it refers to from
  the transcript and the screen activity: "start here" cuts everything from
  zero through the directive; "cut that part where I explained X, I'll start
  over" cuts from where that explanation began through the directive; "stop
  there" cuts from the directive to the end. When you cannot pin the referent
  to a span, still cut the utterance, leave the referent in, and write
  `"resolved": "utterance cut; referent unresolved: <why>"`; never guess
  silently. Words that sound like editing talk without the name are content.
- **Restarts.** When the narrator repeats a passage after a directive or a
  false start, keep the last complete attempt and cut the earlier ones.
- **Scene takes.** A take recorded from a `script-video` script carries no
  narration: every utterance is a directive, and the voice-over is recorded
  later. "Murphy, scene X" starts scene X: put a marker titled with the slug
  `X` there. "Murphy, again" discards the scene in progress back to its
  directive; keep only the last attempt of each scene. Cut idle stretches
  where nothing changes on screen, but keep a beat after each result, since
  the voice-over needs room.
- **Fluff.** Cut filler words and silences longer than about a second, but
  keep the natural pause before a new topic. Word times are approximate:
  place cut edges in the listed silences, never mid-word.
- **Corrections.** Fix transcript words from context: low-confidence marks,
  typed text in the screen activity, names the narrator clearly means. Record
  each as `{"original", "corrected", "evidence"}`.
- **No on-screen text.** The renderer burns in no captions or subtitles. Flag
  a request such as "Murphy, put that address on screen" as unresolved.
- **Markers.** One per topic change, titled in a few words.
- **Zooms.** The brief lists every automatic zoom; judge each against what
  is being said over it. Keep a zoom when the narrator is pointing at detail:
  naming a control, clicking through a small panel, typing, drawing. List a
  `wide` span, with a `reason`, over the action of any zoom the viewer does
  not need: the narrator is explaining a concept or pointing elsewhere ("on
  the right", "the whole screen"), the click only changes tabs or dismisses
  something, the zoom lasts under two seconds, or it sits within a few
  seconds of a cut edge. "Murphy, stay wide" and "Murphy, no zoom here" are
  `wide` spans. A span removes the zooms whose clicks fall inside it and the
  neighbours replan; you cannot add a zoom where nothing was clicked, so flag
  "Murphy, zoom in on X" as unresolved unless a click already frames X.
- **Webcam.** `webcam` spans are where the picture-in-picture is visible.
  Leave the field out to show it throughout; list spans to show it only
  there, for instance hiding it while the screen needs the space.

## Schema

Integer nanoseconds on the recording clock; the brief prints `m:ss.ss`, so
multiply seconds by 1000000000. Spans within the take, in order, no overlap.

```json
{
  "version": 1,
  "cuts": [{"start_ns": 0, "end_ns": 6100000000, "reason": "\"Murphy, start here\": setup before it"}],
  "markers": [{"t_ns": 6000000000, "title": "Getting started"}],
  "webcam": [{"start_ns": 6100000000, "end_ns": 16000000000}],
  "wide": [{"start_ns": 9000000000, "end_ns": 12000000000, "reason": "explaining the idea, not the button"}],
  "directives": [{"quote": "Murphy, start here.", "resolved": "cut 0:00.00-0:06.10"}],
  "corrections": [{"original": "ppu toys.com", "corrected": "ppu.toys", "evidence": "the narrator types ppu.toys in the address bar at 0:08"}]
}
```

`reason`, `directives` and `corrections` are for the review; the renderer
ignores them. `webcam` is optional and requires a session with a webcam.
