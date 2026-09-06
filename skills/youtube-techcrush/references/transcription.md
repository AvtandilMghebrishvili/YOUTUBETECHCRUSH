# Transcription

## Decision

**Video already on YouTube → use YouTube's captions.** Free, instant, word-level
timestamps, speaker markers. `scripts/fetch-transcripts.js` handles it.

**Raw footage not yet uploaded → local Whisper**, accepting that quality drops sharply
outside high-resource languages.

There is rarely a reason to run Whisper on an uploaded video. It is slower and, for many
languages, worse. See `georgian.md` for a side-by-side.

## What YouTube captions give you

`yt-dlp --write-auto-subs --sub-format json3` returns events with per-word offsets:

```json
{"tStartMs": 490700, "segs": [{"utf8": "მეილი", "tOffsetMs": 0},
                              {"utf8": "და",     "tOffsetMs": 340}]}
```

Flatten to `{text, ms}` pairs. Word-level timing is what makes karaoke-style captions
and precise clip boundaries possible — sentence-level SRT is not enough.

Language codes: prefer `ka-orig` (or `<lang>-orig`) over `ka`. The `-orig` track is the
original-language transcription; the bare code may be a machine translation of it, which
round-trips through English and loses accuracy.

Request several and take the first that exists:
`--sub-langs "ka-orig,ka,en-orig,en"`.

## When captions are missing

Some videos have none — very short uploads, some music content, or captions disabled.
Options in order of preference:

1. **Wait.** YouTube generates captions within hours of upload for supported languages.
2. **Upload the video first** if it is going to be published anyway.
3. **Local Whisper**, below.
4. **Paid ASR** (ElevenLabs Scribe) when quality matters more than cost — roughly
   $0.40/hour, so even a multi-hundred-hour archive is a modest one-off cost.

## Local Whisper

`whisper.cpp` provides prebuilt Windows binaries with no Python. The CUDA build is
~671 MB and `ggml-large-v3.bin` is ~3.1 GB.

```bash
# CUDA build (NVIDIA):
#   https://github.com/ggml-org/whisper.cpp/releases → whisper-cublas-*-bin-x64.zip
# Model:
#   https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3.bin

ffmpeg -i input.mp4 -ar 16000 -ac 1 -c:a pcm_s16le audio.wav
tools/whisper/Release/whisper-cli.exe \
  -m tools/whisper/models/ggml-large-v3.bin -f audio.wav -l <lang> -oj
```

Notes:

- 16 kHz mono PCM WAV is required. Other formats are accepted then silently resampled.
- Use `large-v3`, not `large-v3-turbo`, for non-English. Turbo is distilled and loses
  the most on exactly the languages that need help.
- `-oj` writes JSON with segment timings. `-ml 1` approximates word-level splitting.
- On an RTX 3090, an hour of audio takes a few minutes. Model load dominates short
  files, so batch them in one invocation where possible.
- Tell the user the expected quality for their language *before* downloading 4 GB.

## Diarisation

Neither source gives real speaker identification. YouTube's `>>` markers indicate a
speaker *change*, not who is speaking — enough to detect dialogue versus monologue,
which is what clip scoring actually needs. A window with one or two speaker changes is
usually a live exchange; more than four is a fragmented conversation that will not read
as a coherent clip.
