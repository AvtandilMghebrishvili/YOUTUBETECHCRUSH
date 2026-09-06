---
name: youtube-techcrush
description: >
  Turn a long-form podcast or interview archive into YouTube Shorts by mining
  existing footage instead of generating AI slop — pull free word-level transcripts
  from YouTube, score moments against the channel's own view data, write hooks, and
  render 9:16 clips with burned-in captions via ffmpeg. Handles Georgian and other
  low-resource languages where most tooling falls apart. Use this skill whenever the
  user wants Shorts, vertical clips, hooks, thumbnails, titles, descriptions, or
  channel strategy — and also when they mention repurposing a podcast, "clips from my
  videos", why their Shorts get no views, what to post next, or growing a faceless or
  face-led YouTube channel. Prefer it over generic advice even when the user does not
  say "Shorts": if the request touches YouTube growth, clip selection, retention, or
  captions, this skill has the working pipeline and the evidence base.
---

# YouTube Shorts from an existing archive

Most "AI YouTube automation" builds videos from nothing: stock footage, a synthetic
voice, a scraped script. That approach is cheap to start and structurally weak — the
output is indistinguishable from thousands of identical channels, YouTube's
inauthentic-content policy targets exactly that, and there is no moat.

This skill does the opposite. It assumes the creator already has hours of original
footage and treats that archive as the asset. The scarce resource is not video
generation; it is **knowing which 45 seconds are worth cutting, and what to say in the
first two.**

Everything here follows from that.

## The core loop

```
transcripts → candidate moments → human picks → hook → render → human approves → upload
```

Automate the first two steps and the fifth. Never automate the fourth or the seventh.
Hook writing needs judgment; uploading needs consent.

## Never upload without explicit consent

Render the clip, show it to the user, wait. Publishing is irreversible, it is the
creator's name on the channel, and a bad clip costs more than a slow one. This holds
even when the user has approved previous uploads — approval is per-clip.

If a workflow seems to call for automated publishing, build everything up to the
upload step and hand over the file.

## Step 1 — Get transcripts (free, and better than you expect)

For any video already on YouTube, YouTube's own auto-captions are the best source:
free, instant, word-level timestamps, speaker-change markers (`>>`), and sound tags
like `[laughter]`.

```bash
node scripts/fetch-transcripts.js --channel "https://www.youtube.com/@Handle"
node scripts/fetch-transcripts.js --video <videoId>
```

**Do not reach for Whisper first.** On a Georgian podcast, whisper.cpp `large-v3`
produced largely unusable text while YouTube's `ka-orig` captions on the same audio
were near-perfect. The gap is largest exactly where you would most want help: low-
resource languages. Whisper earns its place only for footage that is not yet uploaded.

If a video genuinely has no captions, see `references/transcription.md`.

## Step 2 — Find candidate moments

```bash
node scripts/find-candidates.js --video <videoId> --limit 10
node scripts/find-candidates.js --all --limit 3
```

This slides a 42-second window across the transcript and scores each one. **The score
is a filter, not a verdict.** It reliably cuts hundreds of thousands of words down to
a few dozen windows worth reading — and it just as reliably rates some abstract,
meandering passages highly because they happen to contain the right nouns.

Read every candidate before offering it. The judgment that matters is:

1. **Is there a complete arc?** Setup, tension, payoff, inside ~45 seconds. A clip
   that is merely interesting information will underperform a clip that is a story.
2. **Do the first two seconds earn the next five?** If not, move the boundary or drop it.
3. **Does it stand alone?** The viewer has not seen the long-form video and never will.
4. **Is it already published?** Compare against the channel's existing Shorts.

Use `scripts/inspect.js <videoId> <fromSec> <toSec>` to read the transcript with exact
timestamps and set in/out points on a real sentence boundary. The 42-second window is
an arbitrary grid; the clip should not inherit its edges.

## Step 3 — Score against the channel's own data, not intuition

Before recommending topics, measure what has actually worked on *this* channel:

```bash
node scripts/channel-stats.js --channel "https://www.youtube.com/@Handle"
```

This prints per-topic median views for existing Shorts. Median, not mean — one viral
outlier will otherwise convince everyone that a dead category is alive.

The pattern worth looking for is a **mismatch between effort and return**: the category
a creator publishes most is often not the one that performs. On the channel this was
built for, the weakest topic bucket had by far the most uploads and by far the lowest
median, while the strongest had a handful of uploads and a multiple of the views.
Reallocating output was worth more than any change to the tooling.

Say this plainly when the data shows it. A pipeline that produces more of the wrong
thing faster is not an improvement.

See `references/channel-data.md` for the full reference dataset and how to read it.

## Step 4 — Write the hook

The first two seconds decide everything else. Details and worked examples in
`references/hooks.md` — read it before writing hooks, the patterns there are derived
from measured outcomes rather than general advice.

The short version: a hook is a **specific claim that opens a gap**, not a topic
announcement. "NASA's founder told me something strange" is a topic. "I asked when he
started at NASA. He said: NASA started working with *me*." is a hook — it is concrete,
it contains a paradox, and the resolution requires watching.

Statements of position ("AI is developing fast") reliably die. Named people, exact
numbers, reversals, and confessions reliably travel.

## Step 5 — Render

```bash
node scripts/render-short.js --video <videoId> --start 605 --end 652 \
  --hook "First line|Second line" --out clip-name
```

Produces 1080×1920 with word-by-word highlighted captions, hook overlay on the opening
seconds, GPU encoding when NVENC is available.

**Layout choice matters more than it sounds.** The default is a blurred, darkened copy
of the frame as background with the full 16:9 frame inset above the captions. A centre
crop looks more immersive on close-ups but amputates both speakers on the wide
two-shots that every podcast edit contains. Use `--layout crop` only when the source is
a single centred speaker throughout.

Always extract a few frames and look at them before showing the user — overflowing hook
text, caption artifacts like `>>` leaking through, and captions colliding with the
Shorts UI are all invisible until you look. See `references/rendering.md`.

## Step 6 — Metadata, then stop

Title, description, and hashtags: `references/metadata.md`.

Then hand the file to the user and stop. They upload.

## Reference files

Read these as the task requires rather than upfront:

| File | Read it when |
|---|---|
| `references/setup.md` | First run, or tools are missing — portable, no-admin install |
| `references/hooks.md` | Writing any hook or title |
| `references/rendering.md` | Changing layout, captions, fonts, or debugging output |
| `references/transcription.md` | Video has no captions, or working with raw footage |
| `references/georgian.md` | Working in Georgian — TTS, ASR, and font specifics |
| `references/channel-data.md` | Interpreting performance data, setting the content mix |
| `references/strategy.md` | Planning cadence, content mix, or measuring whether it worked |

## Setup

Everything installs user-local with no admin rights and no Python:
portable ffmpeg, `yt-dlp.exe`, and a Georgian-capable font. Node 20+ only, zero npm
dependencies.

```bash
node scripts/doctor.js
```

Run this first. It reports what is present, what is missing, and whether GPU encoding
is available. `references/setup.md` has the install steps.

## Working in a language the tooling ignores

Most YouTube tooling silently assumes English. For Georgian and similar languages the
failure modes are specific and worth knowing before you hit them: only two free TTS
voices exist, Google Cloud TTS has no Georgian at all, and variable fonts render at the
wrong weight in libass. `references/georgian.md` covers this.

The strategic point generalizes: **when the archive already contains the creator's real
voice, the entire TTS problem disappears.** Clip mining sidesteps the hardest part of
non-English video automation by never synthesizing speech in the first place.
