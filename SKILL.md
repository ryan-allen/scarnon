---
name: scarnon
description: "YouTube research tool. Search for videos on a topic, transcribe them locally, and synthesize insights across multiple sources."
allowed-tools: Bash, Read, Glob, Grep, WebSearch, WebFetch, AskUserQuestion, Task, TaskOutput
---

# /scarnon

YouTube research tool — search, transcribe, and synthesize.

## Tool Dependencies

This skill installs tools progressively as they're needed. Do NOT check for all tools upfront. Instead, check for each tool right before it's first needed, and offer to install it if missing.

Tool install commands (macOS Homebrew):
- `yt-dlp`: `brew install yt-dlp`
- `ffmpeg`: `brew install ffmpeg`
- `jq`: `brew install jq`
- `mlx_whisper`: `pip install mlx-whisper`

Pattern for checking a tool before use:
```bash
command -v TOOL_NAME >/dev/null 2>&1 && echo "OK" || echo "MISSING"
```

If missing, tell the user what's needed and why, and offer to install it. If they decline, stop gracefully.

## Input Routing

```
/scarnon                       -> Search Mode (guided)
/scarnon <topic text>          -> Search Mode with topic pre-filled
```

`$ARGUMENTS` determines behavior:
- Empty or missing -> **Search Mode** (ask for topic)
- Contains text (no `http`) -> **Search Mode** with topic pre-filled

## Search Mode

### Step 1: Topic

If `$ARGUMENTS` already contains topic text, use it and skip to Step 2.

Otherwise, ask:
- question: "What topic do you want to find YouTube videos about?"
- header: "Topic"
- options:
  - "AI safety" / "Example topic"
  - "Cryptocurrency regulation" / "Another example"
- multiSelect: false

The user will typically select "Other" and type their topic.

### Step 2: Check yt-dlp

```bash
command -v yt-dlp >/dev/null 2>&1 && echo "OK" || echo "MISSING"
```

If missing, offer to install: `brew install yt-dlp`

### Step 3: Search YouTube

Generate 3-4 search queries that approach the topic from different angles:
- **Core:** the topic as stated
- **Debate:** topic + "debate" or "discussion"
- **Expert:** topic + "interview" or "expert"
- **Critical:** "problems with" or "against" + topic

Run all searches **in parallel** in one bash call with rich metadata. Each yt-dlp search fetches ~2-3s per video, so parallelizing the 4 queries cuts wall time from ~4 min to ~30s.

Use `ytsearch10` (not `ytsearch20`) — 10 results per query x 4 queries = 40 results before dedup, which is plenty.

**Fields (tab-separated):**
1. `id` — video ID
2. `title` — video title
3. `channel` — channel name
4. `duration_string` — human-readable duration (e.g. "1:23:45")
5. `duration` — duration in seconds (for filtering)
6. `view_count` — total views (for ranking)
7. `upload_date` — YYYYMMDD (for recency)
8. `channel_follower_count` — subscriber count (for authority signal)
9. `tags` — Python list of tags (for clustering)
10. `description` — first 200 chars (for clustering)

```bash
TMPFILE="/tmp/scarnon_search.tsv"
PRINT_FMT="%(id)s	%(title)s	%(channel)s	%(duration_string)s	%(duration)s	%(view_count)s	%(upload_date)s	%(channel_follower_count)s	%(tags)s	%(description).200s"

yt-dlp "ytsearch10:CORE_QUERY" --print "$PRINT_FMT" 2>/dev/null > /tmp/scarnon_s1.tsv &
yt-dlp "ytsearch10:DEBATE_QUERY" --print "$PRINT_FMT" 2>/dev/null > /tmp/scarnon_s2.tsv &
yt-dlp "ytsearch10:EXPERT_QUERY" --print "$PRINT_FMT" 2>/dev/null > /tmp/scarnon_s3.tsv &
yt-dlp "ytsearch10:CRITICAL_QUERY" --print "$PRINT_FMT" 2>/dev/null > /tmp/scarnon_s4.tsv &
wait

cat /tmp/scarnon_s1.tsv /tmp/scarnon_s2.tsv /tmp/scarnon_s3.tsv /tmp/scarnon_s4.tsv > "$TMPFILE"

# Dedup by video ID (field 1), filter for 10min+ (field 5 >= 600s), sort by views descending (field 6)
sort -t"	" -k1,1 -u "$TMPFILE" | awk -F'\t' '$5+0 >= 600' | sort -t"	" -k6,6 -rn
```

**IMPORTANT yt-dlp notes:**
- Use `ytsearch10` — 10 results per query is the sweet spot (fast enough, good coverage)
- Use `ytsearch` NOT `ytsearchdate` — the latter returns empty results
- Do NOT use `--flat-playlist` — it strips per-video metadata
- Do NOT use `--match-filters` for duration — it doesn't work with search. Filter post-hoc via awk
- Do NOT use `--dump-json` — it returns ~700KB per video and is extremely slow. `--print` gets the same fields instantly.
- Tab characters in `--print`, `sort -t`, and `awk -F` must be literal tabs, not `\t` escapes
- Rich fields (`view_count`, `tags`, etc.) cost zero extra time — yt-dlp fetches full metadata per video regardless of which `--print` fields you request

### Step 4: Filter, Cluster & Present Results

#### 4a. Relevance Filtering

Before clustering, filter out irrelevant results. The multi-angle search casts a wide net and will catch tangential videos that share a word (e.g. "nuclear" matching nuclear war, or "fusion" matching chromosome fusion).

Review each result's title, tags, and description snippet. **Drop any video that is not actually about the user's topic.** Use your judgement — if a reasonable person searching for this topic would not want this result, exclude it.

Do not tell the user about dropped results. Just show a brief note like "Found 45 results, showing 34 relevant" so they know filtering happened.

#### 4b. Clustering

Group the remaining results into 3-5 clusters based on titles, tags, and description snippets. Look for natural groupings along dimensions like:
- Content type (e.g. "explainer", "debate/panel", "long-form interview", "news coverage")
- Angle/stance (e.g. "optimistic", "skeptical/critical", "balanced analysis")
- Subtopic (e.g. specific technologies, economic impact, policy, predictions)

Choose cluster dimensions that best fit the actual results — don't force categories that don't exist.

#### 4c. Cluster Selection

Use AskUserQuestion with multiSelect to let the user pick which clusters interest them. Each option label is the cluster name with video count, and the description lists the top 2-3 videos (by views) as a preview.

- question: "I found M relevant videos in N groups. Which areas interest you?"
- header: "Clusters"
- options: one per cluster (up to 4 — combine smallest clusters if more than 4), e.g.:
  - label: "Explainers (12 videos)"
  - description: "Top: Real Engineering (8.1M views), PBS Space Time (3.9M views), Undecided (2.5M views)"
- multiSelect: true

#### 4d. Present Selected Videos

After the user picks clusters, show the videos from those clusters in a table. Number all videos sequentially across selected clusters.

```
## {topic}
*Showing M videos from N selected clusters (filtered from X total results)*

### Cluster Name
Brief description of what unifies this group.

| # | Title | Channel | Duration | Views | Date |
|---|-------|---------|----------|-------|------|
| 1 | Title here | Channel Name (1.2M subs) | 1:23:45 | 2.3M | 2025-03 |
| 2 | Another one | Other Channel (500K subs) | 45:12 | 890K | 2024-11 |

### Another Cluster
...
```

Format large numbers for readability: 1,234,567 -> "1.2M", 234,567 -> "235K", 12,345 -> "12K".
Format dates as YYYY-MM from the YYYYMMDD upload_date field.

Within each cluster, sort by view_count descending (most popular first).

#### 4e. Smart Narrowing

After presenting the selected cluster(s), use AskUserQuestion to offer curated subsets based on the data. Generate options dynamically from the actual results — these should be genuinely useful cuts, not generic placeholders.

Good narrowing dimensions (pick the 3-4 most useful for this result set):
- **By popularity:** "Top N by views" — the biggest hits
- **By recency:** "Most recent (YEAR+)" — only recent content
- **Balanced pick:** "Best of both" — mix of popular + recent, with diversity across channels
- **By duration:** "Deep dives (30min+)" or "Quick watches (under 20min)"
- **All:** "All N videos" — process the full set

- question: "How do you want to narrow down these N videos?"
- header: "Selection"
- options: 3-4 curated subsets (dynamically generated from the data). Each label names the cut, each description lists the specific videos that would be included.
- multiSelect: false

After the user picks, show the final video list and confirm:

```
Ready to process N videos:

1. **Title** — Channel (duration)
2. **Title** — Channel (duration)
...
```

This is the final set. Proceed to Step 5.

### Fallback

If fewer than 4 results after filtering for 10min+, lower the threshold to 5min (300s) and re-filter the same temp file. If still fewer than 4, show what's available and note the limited results.

### Step 5: Download Subtitles

Download auto-generated subtitles for all selected videos. This is fast (~4s per video) and doesn't require ffmpeg or Whisper.

#### 5a. Setup cache directory

```bash
CACHE_DIR="/Users/ryan/.claude/skills/scarnon/.cache"
mkdir -p "$CACHE_DIR"
```

#### 5b. Download auto-subs in parallel

For each selected video, download auto-subs in a single bash call using background jobs:

```bash
CACHE_DIR="/Users/ryan/.claude/skills/scarnon/.cache"

for vid in VIDEO_ID_1 VIDEO_ID_2 VIDEO_ID_3; do
  (
    URL="https://www.youtube.com/watch?v=${vid}"
    HASH=$(echo -n "$URL" | md5 -q -s "$URL" 2>/dev/null || echo -n "$URL" | md5sum | awk '{print $1}')
    SRT_FILE="${CACHE_DIR}/${HASH}_audio.srt"
    META_FILE="${CACHE_DIR}/${HASH}_metadata.txt"

    # Skip if already cached
    if [ -f "$SRT_FILE" ] && [ -s "$SRT_FILE" ]; then
      echo "CACHED:${vid}"
      exit 0
    fi

    # Download auto-subs
    yt-dlp --write-auto-subs --sub-langs "en.*" --sub-format srt \
      --skip-download -o "${CACHE_DIR}/${HASH}_autosub" "$URL" 2>/dev/null

    # yt-dlp may write .en.srt or .en-orig.srt — find whatever was written
    FOUND=$(ls ${CACHE_DIR}/${HASH}_autosub*.srt 2>/dev/null | head -1)
    if [ -n "$FOUND" ] && [ -s "$FOUND" ]; then
      cp "$FOUND" "$SRT_FILE"
      echo "SUBS_OK:${vid}"
    else
      echo "NO_SUBS:${vid}"
    fi

    # Also save metadata if not cached
    if [ ! -f "$META_FILE" ]; then
      yt-dlp --print "URL: https://www.youtube.com/watch?v=%(id)s\nTitle: %(title)s\nChannel: %(channel)s\nDuration: %(duration_string)s\nDescription:\n%(description).500s" "$URL" 2>/dev/null > "$META_FILE"
    fi
  ) &
done
wait
```

#### 5c. Verify results

```bash
CACHE_DIR="/Users/ryan/.claude/skills/scarnon/.cache"
for vid in VIDEO_ID_1 VIDEO_ID_2 VIDEO_ID_3; do
  URL="https://www.youtube.com/watch?v=${vid}"
  HASH=$(echo -n "$URL" | md5 -q -s "$URL" 2>/dev/null || echo -n "$URL" | md5sum | awk '{print $1}')
  SRT="${CACHE_DIR}/${HASH}_audio.srt"
  if [ -f "$SRT" ] && [ -s "$SRT" ]; then
    SIZE=$(wc -c < "$SRT" | tr -d ' ')
    echo "OK:${vid}:${HASH}:${SIZE}bytes"
  else
    echo "MISSING:${vid}:${HASH}"
  fi
done
```

Parse the output. At least 2 videos must have subtitles for synthesis to proceed. If fewer than 2:
- Tell the user which videos had no subtitles
- Offer: "Whisper transcription is not yet supported in scarnon. Want to proceed with what's available, or try different videos?"

For videos with subtitles, report success and proceed to extraction:
```
Downloaded subtitles for N/M videos:
- Video Title — OK (36KB)
- Video Title — OK (52KB)
- Video Title — no subtitles available, skipping
```

### Step 6: Extract (Fan-out to Haiku)

**This is the key context-saving step.** SRT files can be 30-70KB each. Reading them all into main context would blow the window. Instead, fan out to parallel Haiku subagents — each reads one transcript and returns a ~2KB structured extraction.

For each video with a successful transcript, launch a Task subagent **in parallel** with:
- `subagent_type: "general-purpose"`
- `model: "haiku"`

Prompt for each subagent:

```
Read the metadata file at {CACHE_DIR}/{hash}_metadata.txt and the transcript file at {CACHE_DIR}/{hash}_audio.srt.

Extract a structured summary for cross-video research synthesis on the topic "{topic}". Be thorough but concise (~800 tokens).

Return EXACTLY this format:

## Video: {title}
**Channel:** {channel}
**URL:** {url}
**Duration:** {duration}

### Main Arguments & Themes (4-8 items)
For each major topic/argument:
- **Theme name**: 2-3 sentence description of the position/argument
- Key quote: "exact quote" — [MM:SS](https://youtube.com/watch?v=VIDEO_ID&t=XXXs)

To calculate timestamps: convert SRT timestamp HH:MM:SS to total seconds for the t= parameter.

### Notable Claims (3-5 items)
Specific factual claims, predictions, or assertions that could be verified:
- Claim with quote and timestamp

### People & Resources Mentioned
- Notable people, papers, tools, companies referenced

### Key Positions Summary
3-5 sentences capturing the speaker's overall stance and worldview as expressed in this video.

### Practical Advice / Action Items
Any concrete recommendations or actionable takeaways (if present). Write "None" if purely analytical.
```

**Launch ALL subagents in a single message** (parallel tool calls). Each reads its own transcript independently, keeping raw SRT out of the main context.

### Step 7: Lens Selection

After collecting all extractions from the subagents, briefly present a summary of what each video covers (1-2 lines each from the extractions). Then use AskUserQuestion to let the user choose analysis lenses.

Pick the 4 most relevant lenses for this result set from:
- **Shared Themes** — always offer when 2+ videos. Overlapping topics across videos.
- **Contradictions** — offer when positions clash. Opposing viewpoints highlighted.
- **Fact Check** — offer when verifiable claims found. Uses WebSearch to verify.
- **Unique Insights** — always offer when 2+ videos. What each video uniquely contributes.
- **Open Questions** — offer when unresolved debates exist. What remains unanswered.
- **Action Items** — offer when practical advice found. Concrete next steps.
- **Timeline** — offer when videos span different dates. How positions evolved.

- question: "Based on these N videos, which analysis lenses would you like applied?"
- header: "Lenses"
- options: the 4 most relevant lenses (with brief descriptions tailored to what's in the extractions)
- multiSelect: true

### Step 8: Synthesis

Produce the final research output using the extracted summaries (NOT the raw transcripts). The extractions contain all the quotes, timestamps, and structure needed.

```markdown
## Research: {topic}

### Sources
| # | Title | Channel | Duration |
|---|-------|---------|----------|
| 1 | [Title](url) | Channel | HH:MM:SS |
| 2 | [Title](url) | Channel | HH:MM:SS |
...
```

Then one section per selected lens:

**If Shared Themes selected:**
```markdown
### Shared Themes
For each theme that appears across 2+ videos:
#### Theme Name
- How Video 1 addresses it (with timestamped quote)
- How Video 2 addresses it (with timestamped quote)
- Where they agree or diverge
```

**If Contradictions selected:**
```markdown
### Contradictions & Disagreements
For each disagreement:
#### Topic
- **Video 1's position**: quote with timestamp
- **Video 2's position**: quote with timestamp
- Analysis of the disagreement
```

**If Fact Check selected:**
```markdown
### Fact Check
For each verifiable claim, use WebSearch to check it:
- **Claim**: "quote" — Video Title [timestamp]
- **Verdict**: Supported / Disputed / Unverified
- **Source**: link to verification source
```

**If Unique Insights selected:**
```markdown
### Unique Insights
For each video, 2-3 points not covered by any other video:
#### Video Title
- Insight with timestamped quote
```

**If Open Questions selected:**
```markdown
### Open Questions
Unresolved debates or questions that the videos collectively leave unanswered:
1. Question — context from videos
```

**If Action Items selected:**
```markdown
### Action Items
Concrete recommendations aggregated across videos:
1. Action item — from Video Title [timestamp]
```

**If Timeline selected:**
```markdown
### Timeline
How perspectives on this topic have evolved across the videos (ordered by date):
- **Date — Video Title**: position/stance at that time
```

End with:

```markdown
### Key Takeaways
3-5 bullet points synthesizing the most important insights across all videos.

### What Next?

> I have all transcripts cached and can help you:
> - **Deep dive** into any theme above with full quotes from all videos
> - **Extract all quotes** on a specific sub-topic
> - **Web search** to follow up on claims or find newer information
> - **Research more** — search for additional videos on this topic
> Just ask!
```

## Error Handling

- yt-dlp not installed -> offer to install
- Search returns no results -> suggest broadening the topic or trying different terms
- yt-dlp WARNING lines in stderr -> ignore (yt-dlp is chatty)
- Do NOT retry the same search more than once
- Auto-subs not available -> skip that video, tell the user
- Fewer than 2 transcripts -> offer to try different videos or proceed with what's available
- Subagent extraction fails -> read that transcript directly as fallback (note: costs more context)
