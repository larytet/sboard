# sboard - Ambient AI Collaboration Board

A shared session space (whiteboard + chat + doc) where an AI agent watches conversations in real time and autonomously injects sourced responses - no explicit invocation needed.

## The problem

Teams in 2026 still paste AI-generated SQL, Python, and bash snippets back and forth in Slack, context lost, no provenance, no continuity. sboard makes the AI a silent participant in the room: it reads the conversation, surfaces relevant answers, and stays grounded in sources - automatically.

## Design principles

- **Air-gap friendly** - runs fully on-premises; cloud sync is optional
- **Markdown-first** - content stored as plain `.md` files, no database required
- **No binary blobs** - all content is human-readable text; attachments are referenced by path, not embedded
- **Filesystem as storage** - sessions are directories, messages are files, indexing is `find` + `grep`
- **Automatic archiving** - sessions older than a configurable threshold are compressed and moved to `archive/`; active working set stays small

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

1. **Watch** - tails `transcript.md` for new content (file-watch, no polling)
2. **Decide** - scores whether a message warrants an AI response (code snippet, question, factual claim)
3. **Inject** - appends a sourced response to `ai_responses.md`, tagged with timestamp and confidence
4. **Cite** - every response includes the source: web search result, internal doc path, or prior session reference
5. **Archive** - a nightly job compresses sessions older than N days with `zstd`, preserving the index

## Getting started

```bash
git clone git@github.com:larytet/sboard.git
cd sboard
# configure your local model endpoint in agent/config.toml
python agent/watcher.py sessions/my-session/
```

## Integrations

Each integration is a thin adapter that writes into the session transcript as if it were a participant. The agent sees the enriched context and can respond across sources.

### Slack

- **Ingest** - a Slack bot joins a channel and mirrors messages into `transcript.md` with author + timestamp
- **Eject** - AI responses can be optionally posted back to the thread as a bot reply
- **Trigger** - `@sboard` mention in Slack forces an immediate injection; otherwise the agent decides autonomously

```toml
[integrations.slack]
enabled = true
bot_token = "xoxb-..."       # or $SLACK_BOT_TOKEN
channels = ["#incidents", "#eng-standup"]
post_responses = false       # set true to send AI replies back to Slack
```

### Jira

- **Pull** - linked Jira tickets are fetched and prepended to `context.md` when a session starts (or on demand)
- **Watch** - ticket status changes (transition, comment, assignee) are streamed into the transcript so the AI can surface blockers mid-session
- **Write** - the agent can append a comment to a ticket with a summary of the session's conclusions (requires explicit `--write-back` flag)

```toml
[integrations.jira]
enabled = true
base_url = "https://your-org.atlassian.net"
api_token = "..."            # or $JIRA_API_TOKEN
projects = ["OPS", "INFRA"]
watch_transitions = true
write_back = false
```

### PagerDuty

- **Incident feed** - open incidents are injected as structured context at session start; useful for on-call handoffs
- **Timeline sync** - PagerDuty alert events (trigger, ack, resolve) are appended to the transcript in real time so the AI has full incident history
- **Postmortem seed** - after an incident resolves, the session transcript is formatted and attached to the PagerDuty incident as a postmortem draft

```toml
[integrations.pagerduty]
enabled = true
api_key = "..."              # or $PD_API_KEY
services = ["P1ABC23"]
postmortem_on_resolve = true
```

### Adding your own

Integrations implement a two-method interface:

```python
class Adapter:
    def stream(self) -> Iterator[TranscriptEntry]:
        """Yield new entries as they arrive from the external system."""

    def write_back(self, entry: TranscriptEntry) -> None:
        """Optional: push a response back to the external system."""
```

Drop a file in `agent/integrations/` and add a `[integrations.<name>]` block in `config.toml`.

## Roadmap

- [ ] Multi-user transcript merge (operational transform on markdown)
- [ ] Web UI (read-only view of the live session)
- [ ] Plugin API for custom injectors (SQL explainer, code reviewer, diagram generator)
- [ ] Vector index over archived sessions for long-range retrieval
- [ ] GitHub integration - PR diffs and CI failures as session context
- [ ] Confluence / Notion - pull relevant docs into context automatically

## License

MIT
