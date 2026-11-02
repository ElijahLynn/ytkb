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
   accurate). Otherwise it downloads YouTube's small Opus audio track, splits
   it into 20-minute chunks without re-encoding, and transcribes them with
   Whisper on [Groq](https://console.groq.com) (`whisper-large-v3`, falling
   back to `whisper-large-v3-turbo` when rate-limited). The video's title and
   channel are passed as a prompt so names get spelled right.
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

## Settings

Environment variables, or lines in `~/.env` (values may be `op://` references):

| Variable | Default |
| --- | --- |
| `YTKB_STT` | `groq`, or `cloudflare` for Workers AI `whisper-large-v3-turbo` (~$0.03/audio hour) |
| `YTKB_GROQ_API_KEY` | falls back to `GROQ_API_KEY` |
| `YTKB_CF_ACCOUNT_ID` / `YTKB_CF_API_TOKEN` | Cloudflare account id + API token with Workers AI permission |
| `YTKB_FREE` | unset; `1` = use YouTube auto-captions instead of Whisper (free, less accurate) |
| `YTKB_BROWSER` | none; set to use your YouTube login for history / Watch Later |
| `YTKB_TOPICS` | AI-assisted engineering, software practice, business/startups |
| `YTKB_DB` | `~/.local/share/ytkb/ytkb.db` |
| `YTKB_GROQ_MODEL` / `YTKB_GROQ_FALLBACK` | `whisper-large-v3` / `whisper-large-v3-turbo` |
| `YTKB_CLAUDE_MODEL` | `sonnet` |

## Costs

- Groq Whisper: about $0.11 per audio hour (`large-v3`); the free tier allows
  about 2 hours of audio per hour per model, and `ytkb` waits out the limit.
- Claude: one summary per video plus two calls per `ask`, on your Claude plan.

## License

MIT
