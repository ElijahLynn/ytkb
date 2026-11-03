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
   accurate). Otherwise it downloads YouTube's small Opus audio track and
   transcribes it. Recommended: [AssemblyAI](https://www.assemblyai.com)
   (`YTKB_STT=assemblyai`), the most accurate service tested: the whole file goes
   up in one request and its word timestamps give one line per sentence. Other
   providers (Groq, DeepInfra, Cloudflare Workers AI, ...) take 3-minute chunks,
   up to 8 at a time. Every request has a time limit and is retried if it stalls.
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

Pick a transcription service and add its key to `~/.env` (or your environment).
AssemblyAI is recommended (most accurate; $50 free credit). A 1Password `op://`
reference works for any key:

```bash
YTKB_STT=assemblyai
YTKB_ASSEMBLYAI_API_KEY=...        # https://www.assemblyai.com/dashboard/api-keys
# or, e.g. Groq's free tier:
# YTKB_STT=groq
# YTKB_GROQ_API_KEY=op://vault/item/credential   # https://console.groq.com/keys
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
ytkb history                                     # your watch history, filtered (also: ytkb add later / liked)
ytkb ask "how are people doing evals for coding agents?"   # one-shot answer
ytkb chat "how are people doing evals for coding agents?"  # conversation: follow-ups keep context
ytkb pick                                        # choose queued videos with ↑↓ / space / enter
ytkb search '"autonomy slider" OR leash'         # SQLite FTS5 syntax
ytkb show VIDEO_ID --at 1243                     # transcript around 20:43
ytkb show VIDEO_ID --summary
ytkb list | ytkb stats
```

### Your watch history

`ytkb history` (and `ytkb add later` / `ytkb add liked` for Watch Later and Liked
videos) needs your YouTube login. `ytkb` doesn't ask
for a password; it has yt-dlp reuse the login cookies of a browser where you're
signed in to YouTube:

```bash
echo 'YTKB_BROWSER=brave' >> ~/.env    # or firefox, chrome, chromium, ...
ytkb history            # -n 300 for just the latest
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
against the uploaders' captions; lower is better), 2026-11-02:

| Service / model | Word errors | Time per 10 min | Price / audio hour |
|---|---|---|---|
| **AssemblyAI** `universal-3-5-pro` (`YTKB_STT=assemblyai`) | **3.46%** | 14–23 s | $0.21 |
| DeepInfra `Qwen/Qwen3-ASR-1.7B` | 3.60% | 3–7 s when healthy | ~$0.03 |
| DeepInfra `mistralai/Voxtral-Mini-3B-2507` | 3.82% | 14–20 s | ~$0.06 |
| ElevenLabs `scribe_v2` (3 of the 6 talks) | 4.16% | 8–11 s | $0.22 |
| Speechmatics Enhanced | 4.71% | 20–47 s | $0.75 |
| DeepInfra / Groq `openai/whisper-large-v3` | 5.38% | ~12 s | $0.03–0.11 |

AssemblyAI makes about a third fewer errors than Whisper large-v3 and was reliable
throughout; a 78-minute talk takes ~45 s. DeepInfra's Qwen3 is nearly as accurate
and much cheaper, but went from ~5 s to over 60 s per request on 2026-11-02; it's
available with `YTKB_STT=deepinfra` and `YTKB_STT_MODEL=Qwen/Qwen3-ASR-1.7B`.

To re-transcribe saved videos with a different model:

```bash
ytkb add --redo "https://youtu.be/VIDEO_ID"   # keeps the old transcript until this succeeds
```

## Settings

Environment variables, or lines in `~/.env` (values may be `op://` references):

| Variable | Default |
| --- | --- |
| `YTKB_STT` | `assemblyai` (recommended), `groq` (default if unset), `deepinfra`, `together`, `fireworks`, or `cloudflare` (turbo only) |
| `YTKB_ASSEMBLYAI_API_KEY` | AssemblyAI key ([https://www.assemblyai.com/dashboard/api-keys](https://www.assemblyai.com/dashboard/api-keys); $50 free credit) |
| `YTKB_ASSEMBLYAI_MODEL` | `universal-3-5-pro` |
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

- AssemblyAI: $0.21 per audio hour (new accounts get $50 of credit, ~240 hours).
- DeepInfra: about $0.03 per audio hour for Whisper large-v3 or Qwen3-ASR.
- Groq Whisper: about $0.11 per audio hour (`large-v3`); the free tier allows
  about 2 hours of audio per hour per model, and `ytkb` waits out the limit.
- Claude: one summary per video plus two calls per `ask`, on your Claude plan.

## License

MIT
