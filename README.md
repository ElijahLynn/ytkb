# ytkb — YouTube knowledge base

Turn the YouTube talks you watch into a local, searchable knowledge base, then
ask questions and get answers with links that jump to the exact moment
something was said.

```
$ ytkb add "https://www.youtube.com/watch?v=CeOXx-XTYek"
+ Extreme Harness Engineering … — Latent Space (1:17:53)
  downloaded 40 MB in 4s; splitting
  transcribing part 1/4 with Groq … 4/4
  summarizing with Claude
  done: 599 segments

$ ytkb ask "what is the OpenAI guy's name?"
The guest is Ryan Lopopolo, who works on Frontier product exploration
at OpenAI (Extreme Harness Engineering @ 2:17 → https://youtu.be/CeOXx-XTYek?t=137) …
```

Single Python file, stdlib only. Everything lives in one SQLite database
(`~/.local/share/ytkb/ytkb.db`) with full-text search over timestamped
transcript lines.

## How it works

1. **Transcript.** If the uploader wrote captions, `ytkb` uses them (free,
   accurate). Otherwise it downloads YouTube's small Opus audio track, splits it
   into 3-minute chunks without re-encoding, and transcribes up to 8 chunks at a
   time. Each request has a 15 s limit and is retried if it stalls, so one stuck
   request can't hold up a whole video. Providers: Groq, DeepInfra, Cloudflare
   Workers AI and others (`YTKB_STT`). No prompt is sent: with the title as a
   prompt, Whisper large-v3 sometimes echoed it in place of ~30 s of speech.
2. **Summary.** Claude (via the [Claude Code](https://claude.com/claude-code)
   CLI, so it runs on your Claude subscription) writes a summary and a list of
   the concrete techniques and claims.
3. **Answers.** `ytkb ask` has Claude plan full-text searches, gathers the
   matching passages, and answers with citations like
   `[Title @ 20:43](https://youtu.be/ID?t=1243)`. Claude runs with all tools
   disabled, since transcripts are untrusted input.

Nothing runs a model locally: yt-dlp downloads, ffmpeg splits, SQLite stores.

## Install

Requires `python3`, [`yt-dlp`](https://github.com/yt-dlp/yt-dlp), `ffmpeg`,
`curl`, and the [`claude`](https://claude.com/claude-code) CLI (logged in).

```bash
git clone https://github.com/ElijahLynn/ytkb ~/projects/ytkb
ln -s ~/projects/ytkb/ytkb ~/.local/bin/ytkb
```

Add a Groq API key ([https://console.groq.com/keys](https://console.groq.com/keys))
to `~/.env` or your environment. A 1Password reference works too:

```bash
YTKB_GROQ_API_KEY=gsk_...
# or
YTKB_GROQ_API_KEY=op://vault/item/credential
```

### Claude Code skill (optional)

Ask questions inside any Claude Code session; the session itself does the
searching and answering, calling `ytkb search` / `ytkb show`:

```bash
ln -s ~/projects/ytkb/skills/ytkb ~/.claude/skills/ytkb
```

## Usage

```bash
ytkb add "https://youtu.be/VIDEO_ID"            # ingest (no args: clipboard)
ytkb add "https://www.youtube.com/@Channel" -n 10       # latest 10 videos
ytkb add "https://www.youtube.com/playlist?list=..."    # a whole playlist
ytkb add "https://www.youtube.com/feed/history"         # recent history, filtered
ytkb ask "how are people doing evals for coding agents?"   # one-shot answer
ytkb chat "how are people doing evals for coding agents?"  # conversation: follow-ups keep context
ytkb pick                                        # choose queued videos with ↑↓ / space / enter
ytkb search '"autonomy slider" OR leash'         # SQLite FTS5 syntax
ytkb show VIDEO_ID --at 1243                     # transcript around 20:43
ytkb show VIDEO_ID --summary
ytkb list | ytkb stats
```

### Your watch history

Pasting [https://www.youtube.com/feed/history](https://www.youtube.com/feed/history)
(or Watch Later / Liked videos) needs your YouTube login. `ytkb` doesn't ask
for a password; it has yt-dlp reuse the login cookies of a browser where you're
signed in to YouTube:

```bash
echo 'YTKB_BROWSER=brave' >> ~/.env    # or firefox, chrome, chromium, ...
ytkb add "https://www.youtube.com/feed/history"   # -n 300 for just the latest
```

History is mostly off-topic, so it's queued and Claude pre-checks the videos
it thinks you want (by title). A picker opens: ↑↓ move, space toggles, `a`
all/none, enter imports. Only checked videos are transcribed.

**The filter learns from you.** Every time you confirm the picker, your choices
are saved, especially your corrections: videos you checked that Claude missed
and ones it picked that you unchecked. Future filtering shows Claude those
titles as examples alongside `YTKB_TOPICS`, and follows the examples when they
disagree. Videos you `ytkb add` directly count as wanted. `ytkb taste` shows
what it has learned.

For your **entire** history (the feed only reaches back so far):

1. Export it from [Google Takeout](https://takeout.google.com): deselect all,
   pick **YouTube and YouTube Music**, include only **history**, format **JSON**.
2. Queue it, let Claude keep the relevant videos (by title), and ingest:

```bash
ytkb import-takeout watch-history.json
ytkb triage                 # topics: YTKB_TOPICS
ytkb process -n 20          # newest first; re-run to continue
```

### Choosing a speech model

Accuracy on 6 captioned technical talks (~9,800 words, 10 minutes each, scored
against the uploaders' captions; lower is better):

| Model (DeepInfra) | Word errors | Notes |
|---|---|---|
| `Qwen/Qwen3-ASR-1.7B` | **3.6%** | Most accurate. Its word timestamps start a line at each sentence |
| `mistralai/Voxtral-Mini-3B-2507` | 3.8% | No timestamps of its own |
| `openai/whisper-large-v3` | 5.4% | Default |

Qwen3 makes about a third fewer errors than Whisper large-v3, but DeepInfra's Qwen3
went from ~5 s to over 60 s per request on 2026-11-02, so it's opt-in for now:

```bash
echo 'YTKB_STT_MODEL=Qwen/Qwen3-ASR-1.7B' >> ~/.env
ytkb add --redo "https://youtu.be/VIDEO_ID"   # re-transcribe a saved video
```

`--redo` keeps the old transcript until the new one has fully succeeded.

## Settings

Environment variables, or lines in `~/.env` (values may be `op://` references):

| Variable | Default |
| --- | --- |
| `YTKB_STT` | `groq` (default), `deepinfra`, `together`, `fireworks`, or `cloudflare` (turbo only) |
| `YTKB_STT_MODEL` | override the provider's default model (e.g. `Qwen/Qwen3-ASR-1.7B` on DeepInfra) |
| `YTKB_STT_TIMEOUT` | `15` seconds per transcription request before it's retried |
| `YTKB_TIMING_MODEL` | `openai/whisper-large-v3-turbo`: word timings for text models that don't return their own |
| `YTKB_DEEPINFRA_API_KEY` / `YTKB_TOGETHER_API_KEY` / `YTKB_FIREWORKS_API_KEY` | key for that provider |
| `YTKB_GROQ_API_KEY` | falls back to `GROQ_API_KEY` |
| `YTKB_CF_ACCOUNT_ID` / `YTKB_CF_API_TOKEN` | Cloudflare account id + API token with Workers AI permission |
| `YTKB_FREE` | unset; `1` = use YouTube auto-captions instead of Whisper (free, less accurate) |
| `YTKB_BROWSER` | none; set to use your YouTube login for history / Watch Later |
| `YTKB_TOPICS` | AI-assisted engineering, software practice, business/startups |
| `YTKB_DB` | `~/.local/share/ytkb/ytkb.db` |
| `YTKB_GROQ_MODEL` / `YTKB_GROQ_FALLBACK` | `whisper-large-v3` / `whisper-large-v3-turbo` |
| `YTKB_CLAUDE_MODEL` | `sonnet` |

## Costs

- DeepInfra: about $0.03 per audio hour for Whisper large-v3 or Qwen3-ASR.
- Groq Whisper: about $0.11 per audio hour (`large-v3`); the free tier allows
  about 2 hours of audio per hour per model, and `ytkb` waits out the limit.
- Claude: one summary per video plus two calls per `ask`, on your Claude plan.

## License

MIT
