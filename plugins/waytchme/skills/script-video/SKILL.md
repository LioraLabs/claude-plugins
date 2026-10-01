---
name: script-video
description: Write a WaytchMe walkthrough script for a codebase: named scenes with on-screen actions and voice-over lines, plus the vocabulary file, ready to record. Use when asked to script, plan or storyboard a video of a project.
---

# Scripting a WaytchMe video

A WaytchMe video is made in three passes, and the script drives all of them:

1. **Footage.** The user records the screen in one take, speaking only
   directives to Murphy (the editor in the `edit-take` skill), never narration.
2. **Voice-over.** The user reads each scene's lines into its own file.
3. **Assembly.** Murphy cuts the footage into scenes and speeds up or holds each one
   to fit its voice-over.

Because the picture is retimed to the voice, each scene's actions must
fit inside its lines when sped up to at most 1.5×. A scene whose actions
take far longer than its lines needs fewer actions or more lines.

The user edits the script after you write it. Your draft is a starting
point they own, so write lines they would say, not marketing copy.

## Steps

1. **Find the waytchme checkout.** It holds `waytchme.py`; ask if it is not
   `~/dev/waytchme`. Pick a short `SLUG` for the video. Done when you know
   both paths.
2. **Study the codebase.** Read the README and product docs, then the UI
   code: the screens, panels, controls and their exact labels. Find the
   one thing a newcomer can build from nothing that touches the core ideas.
   Done when you can name that thing, the ideas it teaches in order, and the
   label of every control you will tell the user to click.
3. **Outline the story.** One scene per idea, in teaching order, building
   that one thing from an empty start to a result worth showing. Open with
   why the project exists and close with what the viewer can do next. If
   the user did not give a length, aim for 5 to 8 minutes. Done when every
   scene has a slug and one sentence of purpose.
4. **Write the script** to `sessions/SLUG-script.md` in the waytchme checkout,
   in the *Script format* below. Budget about 150 spoken words per minute
   and time each scene from its lines. Done when every scene has a start
   state, numbered actions naming real controls, and its lines, and the
   footer totals the running time.
5. **Write the vocabulary** to `sessions/SLUG-words.txt`: every product name,
   jargon term, register and file name the lines use, one per line. It is
   the prompt `waytchme.py transcribe --vocabulary` gives whisper. Done when
   every unusual word in the lines appears in it.
6. **Hand over.** Give the user both paths and the scene list with times,
   and tell them to edit the script before recording. Stop there.

## Script format

```markdown
# <Video title>

slug: <SLUG> · about <N> min · for <audience>

## Before recording
- <what is open, logged in, reset or pre-loaded, so scene 1 starts clean>

## Recording the footage
`python3 waytchme.py record sessions/<SLUG>`. Speak only to Murphy:
- "Murphy, scene <slug>." at the start state of each scene, then a beat of silence, then its actions.
- "Murphy, again." to redo the scene you are in: put the screen back to its start state, then go again.
- "Murphy, stop." when the last scene is done.
Click each control you name (zooms follow clicks), and hold still a second on every result.

## Recording the voice-over
Per scene, after the footage: `ffmpeg -v error -f pulse -i <mic> -c:a pcm_s16le sessions/<SLUG>/vo/<slug>.wav`, `q` to stop.
Mic names: `python3 waytchme.py devices`.

## 1 · <slug> — <Title>  (~<seconds> s)
**Start:** <the screen state this scene begins from>
**Do:**
1. <one action on a named control>
2. ...
**Say:**
<the voice-over lines, as spoken>

## 2 · ...

---
Total: about <m:ss> of voice-over across <n> scenes.
```

## Writing the lines

- **Show, then name.** Each line describes what the action on screen just
  did or is about to do; a line with nothing on screen under it belongs in
  the intro or outro.
- **One idea per scene**, introduced in plain words before its jargon:
  "the palette, which the hardware calls CGRAM".
- **The product's own names**, spelled as the UI spells them, so the vocabulary,
  the transcript and the screen agree.
- **Spoken rhythm.** Short sentences, contractions, no lists read aloud.
