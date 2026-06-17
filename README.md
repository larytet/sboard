# sboard — Ambient AI Collaboration Board

A shared session space (whiteboard + chat + doc) where an AI agent watches conversations in real time and autonomously injects sourced responses — no explicit invocation needed.

## The problem

Teams in 2026 still paste AI-generated SQL, Python, and bash snippets back and forth in Slack, context lost, no provenance, no continuity. sboard makes the AI a silent participant in the room: it reads the conversation, surfaces relevant answers, and stays grounded in sources — automatically.

## Design principles

- **Air-gap friendly** — runs fully on-premises; cloud sync is optional
- **Markdown-first** — content stored as plain `.md` files, no database required
- **No binary blobs** — all content is human-readable text; attachments are referenced by path, not embedded
- **Filesystem as storage** — sessions are directories, messages are files, indexing is `find` + `grep`
- **Automatic archiving** — sessions older than a configurable threshold are compressed and moved to `archive/`; active working set stays small

## Directory layout

```
sboard/
  sessions/
    2026-06-17-standup/
      00_transcript.md      # running conversation log
      01_ai_responses.md    # agent injections with source citations
      context.md            # optional seed context for the agent
  archive/
    2026-05-*.tar.zst       # older sessions, compressed
  agent/
    watcher.py              # real-time conversation monitor + injector
    config.toml             # model endpoint, archive policy, thresholds
```

## How the agent works

1. **Watch** — tails `transcript.md` for new content (file-watch, no polling)
2. **Decide** — scores whether a message warrants an AI response (code snippet, question, factual claim)
3. **Inject** — appends a sourced response to `ai_responses.md`, tagged with timestamp and confidence
4. **Cite** — every response includes the source: web search result, internal doc path, or prior session reference
5. **Archive** — a nightly job compresses sessions older than N days with `zstd`, preserving the index

## Getting started

```bash
git clone git@github.com:larytet/sboard.git
cd sboard
# configure your local model endpoint in agent/config.toml
python agent/watcher.py sessions/my-session/
```

## Roadmap

- [ ] Multi-user transcript merge (operational transform on markdown)
- [ ] Web UI (read-only view of the live session)
- [ ] Plugin API for custom injectors (SQL explainer, code reviewer, diagram generator)
- [ ] Vector index over archived sessions for long-range retrieval

## License

MIT
