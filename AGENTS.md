# AGENTS.md

How to use SILframe from the command line, without opening the window.
Written for AI agents and scripts. The README covers the window.

## What this is

A video player for Windows with simple editing. `silframe-cli.exe` does the non-playback part
from the command line: it reads a video's properties, saves one frame as a PNG, and exports a
clip (trim, join ranges, crop, rotate, zoom, change speed). It uses the same code as the app.

## Setup

- Requires: Windows 10 version 2004 or later, or Windows 11, x64. SILframe 1.1.0 or later,
  installed with its setup program.
- The command is installed next to the app:

  ```
  %LOCALAPPDATA%\Programs\SILframe\silframe-cli.exe
  ```

  In PowerShell: `& "$env:LOCALAPPDATA\Programs\SILframe\silframe-cli.exe" info talk.mp4`.
  If the person chose "Add to PATH" during setup, `silframe-cli` alone works in terminals opened
  after that.
- From source: see [docs/BUILDING.md](docs/BUILDING.md). After `tools\bootstrap-dependencies.ps1`,
  `dotnet run --project SILframe.Cli -- info talk.mp4` runs the same command.
- Working directory: any. Paths can be relative or absolute.
- FFmpeg and ffprobe are bundled. Nothing else to install.

The examples below write `silframe-cli`. Use the full path when it is not on `PATH`.

Check that it works before anything else:

```bash
silframe-cli --version
```

Success: exit code 0 and `{"ok": true, "version": "1.1.0"}` (or a later version).

## Common tasks

Times are seconds (`12.5`), `m:ss` (`1:02.5`), or `h:mm:ss`, from the start of the video.

### Read a video's properties

```bash
silframe-cli info talk.mp4
```

- Output: `media.durationSeconds`, `media.displayWidth` / `displayHeight`, `media.frameRate`,
  `media.videoCodec`, `media.audioCodec`, `media.hasAudio`, `media.isHdr`, and more.
- `displayWidth` and `displayHeight` are the picture as it is shown, with rotation applied.
  `--crop` uses these pixels.
- Check: exit code 0 and `"ok": true`.

### Save one frame as a PNG

```bash
silframe-cli frame talk.mp4 --time 1:23.5 --out slide.png
```

- Output: the PNG at `--out`. Without `--out`, it goes next to the video, named after the video
  and the time. Read the path from `output` in the result.
- The frame saved is the first one at or after `--time`.
- `--crop X,Y,W,H`, `--rotate 90|180|270`, and `--zoom F[,FX,FY]` change the picture.
  The crop is taken from the picture as `info` reports it, and `--rotate` then turns the result.
  With `--rotate 90` or `270`, a `W` by `H` crop is saved as `H` by `W` pixels.
  There are no percentages: for the top-left quarter of a 1920x1080 video, pass `--crop 0,0,960,540`.
  All four crop numbers must be even.
- Check: exit code 0, `"ok": true`, and `width` / `height` in the result, which are read back
  from the written PNG. (`info` reads videos only, not PNG files.)

### Export a clip

```bash
silframe-cli export talk.mp4 --start 0:10 --end 0:40 --out clip.mp4
silframe-cli export talk.mp4 --segment 0:10-0:20 --segment 1:00-1:15 --out highlights.mp4
silframe-cli export talk.mp4 --start 5:00 --end 9:30 --copy --out part.mkv
```

- Output: one file at `--out`. The extension selects MP4, MKV, or MOV.
  Re-encoded output is H.264 video with AAC audio.
- `--segment` can be repeated; the ranges are joined into one file. Give them from the earliest
  to the latest, without overlap. They can't be reordered.
- `--crop`, `--rotate`, `--zoom`, and `--speed 0.25`–`4` apply to the whole export, in the same
  way as for `frame`.
- `--copy` cuts without re-encoding: fast and lossless, but the cut points move to nearby
  keyframes, so the result can be a little longer than asked. It can't be combined with
  `--crop`, `--rotate`, `--zoom`, `--speed`, or more than one `--segment`, because those change
  the picture or the timing and so need re-encoding.
- Check: exit code 0, `"ok": true`, and `media` in the result, which is read back from the written
  file. Compare `media.durationSeconds` with `requestedDurationSeconds`, which is the length of
  the ranges in the source (with `--speed 2`, the file is half of it), and `media.displayWidth` / `displayHeight` with the size you expect.
  A re-encoded file can report a few hundredths of a second more than requested, because the
  audio track ends slightly after the last video frame.
  With `--copy`, `boundariesAreApproximate` is `true` and the duration can differ by a few seconds.
- Progress such as `encoding 42%` goes to stderr while it runs.

## Running without prompts

The command never asks for input and never opens a window.

- stdout carries exactly one JSON object. Progress goes to stderr; `--quiet` hides it.
  The output is always JSON, so there is no `--json` option to add (it is accepted and ignored).
- An existing `--out` file is not replaced: the command fails with `invalid_request`.
  `--overwrite` replaces it. The input file is never replaced.
- Exit codes: `0` success, `1` the request could not be carried out, `2` the command line is wrong.
  In PowerShell, read `$LASTEXITCODE` right after the command.

Errors look like this, on stdout:

```json
{
  "ok": false,
  "error": {
    "code": "invalid_request",
    "message": "--time 150 is past the end of the video (120 seconds)."
  }
}
```

`code` is one of `usage`, `invalid_request`, `failed`, `io_error`, `canceled`. `message` says what
was wrong and, where there is a way around it, what to change. For `failed`, `detail` holds the
end of the FFmpeg output.

## Side effects

- Writes: the `--out` file (or, for `frame` without `--out`, a PNG next to the video).
- Temporary files: an export writes a temporary file next to `--out` and renames it when done.
  After a failure or Ctrl+C, the partly written file is removed and `--out` is not created.
- Overwrites or deletes: nothing, unless `--overwrite` is given.
- Network: none. Only local files can be opened.
- Settings: reads and writes no settings. The same command gives the same result.
- Safe to run again: yes, with a new `--out` name or with `--overwrite`.

## Limits

Local files only. Only the first audio track is exported. Encoding is done in software.
HDR videos can't be re-encoded (`media.isHdr` is `true`); export them with `--copy`.

## More help

- `silframe-cli --help` lists every option with examples.
- [docs/CLI.md](docs/CLI.md) (English) and [docs/CLI.ja.md](docs/CLI.ja.md) (Japanese) describe
  each option and the result fields.
- The window: [README.md](README.md) (English), [README.ja.md](README.ja.md) (Japanese).

## agent-friendly check record

The criteria for the agent-friendly mark are in [tomhda/agent-friendly-mark](https://github.com/tomhda/agent-friendly-mark).

2026-10-11 — Claude Opus 5.5 and Codex gpt-6.1-sol (high), each in a new conversation with only
this repository and an installed SILframe 1.1.0. Tasks: read a video's properties, save a cropped
frame, join two ranges into one file, and cut a range without re-encoding while rotating it
(a planted failure: `--copy` can't rotate). Both finished all four without help, without guessing
a command that doesn't exist, and recovered from the failure by re-encoding. Neither read the
source to learn the usage. Their reports led to: `width` and `height` in the `frame` result, and
the notes above on which frame `--time` picks, even crop numbers, the order of `--segment`, and
the reported duration.
