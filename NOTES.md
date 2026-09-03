# transcribe — project notes

Working notes for the tool itself. User-facing documentation lives in `README.md`.

## What it is

A local CLI that turns any audio or video file into a `.txt` transcript plus a
timestamped `.srt`. Runs `mlx-whisper` on Apple Silicon through `uvx`, so nothing is
installed permanently, nothing is uploaded, and no API key is involved.

## Status

Working. v1.2.0. Verified by hand on real recordings, not only by an automated
harness — including a first run performed by someone following the README rather
than being handed the commands.

## Design decisions worth remembering

- **`uvx`, not a permanent install.** The transcriber is fetched into a throwaway
  environment each run. Keeps the machine clean; costs a few seconds of startup.
- **Prefer Zoom's `.m4a` over its `.mp4`.** Zoom writes `audioNNN.m4a` and
  `videoNNN.mp4` holding the same audio track. Transcribing the `.m4a` is identical
  and skips an extraction.
- **The decode rate is measured from the first decoded segment**, not from process
  launch. Model loading takes ~15 s and would otherwise poison the ETA — an early
  version confidently predicted 1:25:09 for a 50-second job.
- **Whisper's output is force-flushed** (`PYTHONUNBUFFERED=1` in the child). Without
  it, Python block-buffers into a pipe and the progress line shows nothing until the
  run is already over.
- **Sources are never written to.** Every path is read-only; output goes elsewhere.
- **Stdlib only**, so it runs on the `python3` that ships with macOS. No virtualenv,
  no `pip install`, nothing for a new user to set up beyond ffmpeg and uv.

## Verified

- Reproduced two known-good transcripts word for word, which is what confirmed
  correctness rather than just "it ran".
- `--name` edge cases: embedded path rejected, typed extension dropped, empty name
  rejected, multiple recordings numbered `(2)`, `(3)`.
- Filenames with emoji, apostrophes, spaces, and 200 characters. Symlinked inputs.
- Skip-if-already-transcribed, and `--force` to override it.
- Progress display under a real terminal and piped to a log.
- Exit codes: 0 on success, 1 on failure.

## On transcription accuracy

An earlier note here claimed Whisper was mangling proper nouns in a test recording.
That was wrong — the words in question were spoken exactly as transcribed. Check the
source audio before concluding the model is at fault.

## Pre-publish audit

Ran the tool the way a stranger would, on a machine stripped of its tools. Seven
issues found, all fixed in v1.2.0:

1. **`--dry-run` needed `ffprobe` but skipped the prerequisite check.** Without ffmpeg
   installed, the first command a new user ran said "no audio track, skipping" — blaming
   their file for a missing dependency. The worst of the batch, and invisible from inside
   a working environment.
2. **A read-only output folder produced a raw Python traceback.** Now one sentence.
3. **An explicitly named file was rejected on its extension**, so `transcribe clip.mka`
   failed even though ffmpeg reads it fine. Named files are now honoured whatever the
   extension; folder scans still filter, to avoid probing everything.
4. **"No audio track" was reported for files ffprobe simply could not read.** Two
   different problems wearing the same message.
5. **A bad `--language` dumped 30 lines of usage plus all 200 language names.** Now the
   single relevant line.
6. Word count read its own output using the locale encoding rather than UTF-8.
7. Dead parameter in the progress emitter.

Also added: an Apple Silicon check that explains itself, a LICENSE, and a README note
that no Python install is needed.

The lesson worth keeping: every one of those was invisible from inside a working
directory with the right commands already in hand. They only appeared by running from
outside one.

## Possible future features

None of these are started, and none are needed for what the tool does today.

- **Speaker labels (diarization).** Whisper transcribes speech but doesn't separate
  who is talking, so a multi-person meeting comes out unattributed. Adding it means a
  second model and a change to the output format.
- **`--hint` / custom vocabulary.** Whisper accepts an initial prompt that biases it
  toward supplied terms — useful for jargon, gene names, or unusual spellings. Worth
  doing only if a real misspelling shows up.
- **Word-level timestamps.** `mlx_whisper` supports `--word-timestamps`, which would
  make the `.srt` finer-grained. No use case yet.
