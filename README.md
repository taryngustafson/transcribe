# 🎙️ transcribe

Turn a recording into a text transcript without watching or listening to it.

Point it at a video or audio file — or a folder of them — and it writes a plain
`.txt` transcript plus a timestamped `.srt` next to the recording.

Works with anything `ffmpeg` can read: `.mp4`, `.mov`, `.m4v`, `.mkv`, `.avi`,
`.webm`, and audio-only files like `.m4a`, `.mp3`, `.wav`. The rule of thumb is
that if QuickTime or VLC plays it, this transcribes it.

> **Everything runs on your Mac.** The audio is never uploaded, no API key is
> needed, and it works offline. Your recordings are only ever read, never modified.

> **🍎 Apple Silicon only.** It uses `mlx-whisper`, Apple's Metal build of OpenAI's
> open-weights Whisper model. On an M1 it transcribes at roughly **10× realtime** —
> an 8-minute recording takes about 50 seconds.

## Requirements

| Tool | Why | Install |
|---|---|---|
| `ffmpeg` | reads the audio out of the recording | `brew install ffmpeg` |
| `uv` | fetches the transcriber on demand | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |

**Python:** none to install. It runs on the `python3` that ships with macOS
(tested on 3.9), using only the standard library.

The transcriber itself is **not** permanently installed. `uvx` fetches it into a
throwaway environment each run — the same idea as running an R package without
adding it to your project's `renv` library.

`transcribe` checks for all of this up front and tells you what's missing.

## Installation

```bash
git clone https://github.com/taryngustafson/transcribe.git ~/transcribe
mkdir -p ~/bin
ln -s ~/transcribe/transcribe ~/bin/transcribe
```

`~/bin` does not exist on a fresh macOS account, which is why `mkdir -p` is there —
without it, `ln` fails with a "No such file or directory" that appears to blame the
script.

Then make sure `~/bin` is on your `PATH`. Check with:

```bash
echo $PATH | tr ':' '\n' | grep "$HOME/bin"
```

If that prints nothing, add it:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Now verify:

```bash
transcribe --help
```

> ⚠️ **Keep the cloned folder.** `~/bin/transcribe` is a symlink pointing back
> into it. If you delete or move the folder, the command breaks.

## Usage

```bash
transcribe <path>... [options]
```

`<path>` can be a single media file, a folder of them, or several of either.

## Examples

```bash
# One video — the transcript lands next to it
transcribe ~/Desktop/tutorial.mp4

# Choose what the transcripts are called
transcribe lecture.mp4 --name "Lecture 3"

# A whole Zoom meeting folder
transcribe "~/Documents/Zoom/2026-01-15 10.30.00 Team Meeting"

# One file, transcripts collected somewhere else
transcribe talk.mp4 -o ~/Desktop/transcripts

# See what it would do, without doing it
transcribe ~/Documents/Zoom/2026-01-* --dry-run

# Just the transcript, skip the timestamped subtitles
transcribe interview.m4a -f txt

# A recording that isn't in English
transcribe entrevista.mp4 -l es
```

## Options

| Flag | Description |
|---|---|
| `-o`, `--output-dir DIR` | Write transcripts here (default: next to each recording) |
| `-N`, `--name NAME` | Name the output files (default: the recording's own name) |
| `-f`, `--format LIST` | Comma-separated: `txt`, `srt`, `vtt`, `tsv`, `json` (default: `txt,srt`) |
| `-l`, `--language LANG` | Spoken language, or `auto` to detect (default: `en`) |
| `-m`, `--model MODEL` | Whisper model (default: `mlx-community/whisper-large-v3-turbo`) |
| `-r`, `--recursive` | Search folders recursively |
| `-n`, `--dry-run` | List what would be transcribed, then stop |
| `--force` | Overwrite existing transcripts |
| `--keep-audio` | Also keep the extracted 16 kHz `.wav` |
| `-q`, `--quiet` | Suppress the live progress line |
| `-h`, `--help` | Show help |
| `-V`, `--version` | Show version |

## Zoom folders

Nothing here is Zoom-specific, but Zoom is a common source, so it knows two of
its conventions.

Zoom writes a pair of files per stream — `audio1234.m4a` and `video1234.mp4` —
holding the **same audio track**. `transcribe` notices the pair and transcribes
the `.m4a`, which is identical and skips an extraction step. It tells you when it
does this.

It also skips files with no audio track, and warns about
`double_click_to_convert_*.zoom` files, which Zoom hasn't converted yet — open
those in Zoom first.

## Output

By default the transcripts take the recording's own name. For a file called
`audio1234567890.m4a` you get:

- **`audio1234567890.txt`** — the transcript as plain text, ready to paste into an
  LLM or search
- **`audio1234567890.srt`** — the same words with timestamps, so you can jump
  straight to a moment in the video instead of scrubbing for it

Zoom's filenames are not exactly memorable, so `--name` lets you pick something better:

```bash
transcribe audio1234567890.m4a --name "Team standup"
# -> Team standup.txt
# -> Team standup.srt
```

A few details worth knowing about `--name`:

- It names the **files**, not the folder. Use `-o` for the folder.
- If you accidentally type an extension (`--name notes.txt`), it drops it rather
  than producing `notes.txt.txt`.
- Pointed at several recordings at once, the first takes the name and the rest get
  `(2)`, `(3)`, and so on. `--dry-run` shows you the names before anything is written.

Existing transcripts are left alone unless you pass `--force`.

## Progress

Long recordings print a live line so you can tell at a glance that it's working:

```
[1/2] audio1234567890.m4a  (12:00)
    extracting audio...
    7:30 / 12:00   63%   10.0x realtime   ETA 0:27
    done in 1:12 (10.0x realtime) — 1800 words, wrote txt, srt
```

When output is piped to a log file it prints one line every 30 seconds instead of
repainting.

## Limits worth knowing

- **A transcript only captures what was said.** If the meaning lives on screen —
  slides, a diagram, "as you can see here" — the text alone won't carry it.
- **No speaker labels.** Whisper transcribes speech but doesn't separate who is
  talking. For a multi-person meeting you get the words in order, unattributed.
- **The first run downloads the model** (~1.5 GB), cached afterwards under
  `~/.cache/huggingface`. Later runs start in seconds.
- **Apple Silicon only**, and it says so plainly rather than failing strangely.
  On an Intel Mac, Linux or Windows, [openai-whisper](https://github.com/openai/whisper)
  or [whisper.cpp](https://github.com/ggerganov/whisper.cpp) do the same job.

## License

MIT — see [LICENSE](LICENSE).
