# scarnon

**"Scarnon?"** — Australian for *"what's going on?"*

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that rips through YouTube so you don't have to. Chuck it a topic, it'll find the videos, grab the transcripts, and give you the good oil — all without leaving your terminal.

## What's it do

You say `/scarnon nuclear fusion breakthroughs` and it:

1. **Searches YouTube** from multiple angles (4 parallel queries, ~30 seconds flat)
2. **Filters out the rubbish** — no one asked for Shroud of Turin when they searched for fusion
3. **Clusters the results** into sensible groups (explainers, debates, interviews, etc.)
4. **Lets you pick** which clusters and videos you actually care about
5. **Downloads subtitles** in parallel (auto-captions, no Whisper needed for ~80% of videos)
6. **Fans out to Haiku subagents** — each one reads a transcript and sends back a tight summary, keeping the main context lean as
7. **Synthesises across videos** with lenses you choose: shared themes, contradictions, fact checks, open questions, whatever
8. **Gives you timestamped quotes** so you can jump straight to the good bits

The whole thing runs in about 2 minutes for 3 videos. Not bad for a bloke in a terminal.

## Installation

Chuck it in your skills directory:

```bash
git clone <this-repo> ~/.claude/skills/scarnon
```

Or just paste this repo URL into Claude Code and say *"oi install this skill for me yeah"* — it'll figure it out. She's not stupid.

That's it. She'll show up as `/scarnon` next time you fire up Claude Code.

### Dependencies

Scarnon installs tools **as it needs them**, not all up front. First time you run a search it'll check for `yt-dlp` and offer to install it. You won't get asked about `ffmpeg` or `mlx_whisper` until you actually need them (which for auto-captions, you don't).

The only hard requirement for search + subtitle mode:

```bash
brew install yt-dlp
```

## Usage

```
/scarnon                              # guided — asks you what topic
/scarnon future of jobs AI            # pre-filled topic, straight to search
```

Then just follow the prompts. Pick your clusters, narrow your selection, choose your analysis lenses. She's pretty self-explanatory once you're in it.

## How it works under the hood

### Search (the fast bit)

Four `yt-dlp` searches run in parallel with `&` and `wait`. Each grabs 10 results with rich metadata (views, subs, date, tags, description) in a single `--print` pass. No `--dump-json` — that returns 700KB per video and is slow as. The whole search takes ~30 seconds wall time.

### Subtitles (the free bit)

~80% of YouTube videos have auto-generated captions. Scarnon grabs those via `--write-auto-subs` — takes about 4 seconds per video, no audio download, no Whisper, no GPU. Parallel download for the lot.

### Extraction (the clever bit)

SRT files are 30-70KB each. Reading three of them into your context window is a bad time. Instead, scarnon fans out to parallel Haiku subagents — each reads one transcript, extracts themes/quotes/claims into ~2KB of structured markdown, and sends it back. The main context never sees the raw subtitles. Fan-out, fan-in. Sweet as.

### Synthesis (the useful bit)

With compact extractions from each video, the main model cross-references them through whichever lenses you picked: shared themes, contradictions, unique insights, open questions, fact checks, timelines, action items. All with timestamped YouTube links so you can verify anything that sounds suss.

## What it doesn't do (yet)

- **Direct URL mode** — `/scarnon https://youtube.com/...` for a single video
- **Whisper fallback** — for the ~20% of videos without auto-captions
- **Library mode** — browse your cached transcripts
- **Compare mode** — side-by-side analysis of cached videos
- **Non-YouTube** — podcasts, MP3s, etc.

She'll get there. Baby steps.

## Why "scarnon"?

Because every good tool deserves a name that makes people go "what?" before they try it. Also because `/transcribe` was taken and this one's better anyway.

If you're not Australian: "scarnon" is short for "what's going on" — as in, *"scarnon with nuclear fusion?"* or *"scarnon in the Middle East?"*. It's what you say when you want the lowdown. Which is exactly what this does.

## Licence

Do what you want with it mate. Just don't be a drongo about it.
