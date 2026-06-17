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

## End-to-end: incident response

**1. Alert fires**

PagerDuty triggers. sboard creates a new session directory automatically:
`sessions/2026-06-17-14:32-incident-P1ABC23/`. It pre-populates `context.md`
with the incident title, severity, affected services, and any runbooks linked
to that service.

**2. Slack channel opens**

The on-call creates `#incident-2026-06-17-api-down` (or PagerDuty creates it).
The sboard bot joins. Every message, code snippet, and log paste from this
point streams into `transcript.md`.

**3. Huddle starts**

Five people join the voice call. Whisper (on-prem) transcribes audio and
appends to the same `transcript.md`, speaker-labeled. Voice and Slack are now
one unified timeline.

**4. Agent watches in real time**

As content arrives the agent scans continuously:

- Stack trace pasted - agent finds the file and line in the codebase, notes the
  last commit that touched it and the PR description
- "Check the database latency" said out loud - flagged as an open question in
  `open_items.md`
- SQL or bash snippet pasted - agent explains it and notes which tables or
  services it touches
- "This started around 14:15" - agent scans PagerDuty timeline and recent Slack
  history for anything that changed near that time

All annotations are appended to `ai_responses.md` with timestamps, visible to
anyone watching the session.

**4a. Log correlation**

The agent actively fetches logs from configured backends and cross-references
them against what participants are saying:

- Someone says "payment service is throwing 500s" - agent queries Loki or
  Elastic for `service=payment level=error` in the last 30 minutes, pulls the
  top error patterns, and injects a summary with log line counts and first/last
  occurrence times
- An error string appears in Slack - agent searches all log backends for that
  exact string, reports which pods/hosts it appeared on and how frequently
- Someone pastes a request ID or trace ID - agent fetches the full trace from
  the log store and reconstructs the request path across services
- Log volume spike detected independently - agent surfaces it proactively
  without being asked, timestamped against the incident timeline
- Postgres slow query log shows a query taking 30s - agent links it back to
  the service and endpoint mentioned 2 minutes earlier in the Slack thread

The cross-reference works in both directions: Slack comment triggers a log
fetch, and an anomaly in the logs triggers a Slack annotation. The merged
view in `ai_responses.md` shows the conversation and the evidence side by side.

**5. Participants interact via Slack as normal**

Paste a log snippet, a config value, a URL - the agent cross-references it.
No slash commands. No new tool to open. No change to how the team works.

**6. Resolution**

On-call resolves the PagerDuty incident. sboard detects this and generates
`postmortem_draft.md`:

- Full timeline reconstructed from the merged transcript
- Root cause candidates ranked by frequency of mention and agent corroboration
- Open items that were raised but never answered
- Every code snippet annotated with its source
- Action items extracted from phrases like "we should", "someone needs to"

The draft is attached to the PagerDuty incident automatically.

**7. Archive**

Session is compressed after N days. The index stays searchable - next time the
same service pages, sboard surfaces: *"similar incident 6 weeks ago, root cause
was connection pool exhaustion, resolved by restarting the worker pool"*.

---

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

### Kubernetes

- **Pod log tail** - streams stdout/stderr from pods matching a label selector into the transcript
- **Event watch** - k8s events (CrashLoopBackOff, OOMKilled, failed scheduling) are injected as structured entries
- **Describe on demand** - when a pod name appears in conversation, agent runs `kubectl describe pod` and appends the output

```toml
[integrations.kubernetes]
enabled = true
kubeconfig = "~/.kube/config"   # or in-cluster service account
namespaces = ["production", "payments"]
label_selector = "app in (api, worker)"
tail_lines = 200
```

### Loki

- **LogQL query** - when a service name, error string, or time window is mentioned, agent constructs a LogQL query and fetches matching log lines
- **Spike detection** - polls log volume for configured labels; injects an alert if error rate exceeds threshold
- **Trace correlation** - extracts trace IDs from log lines and links them to spans if Tempo is also configured

```toml
[integrations.loki]
enabled = true
url = "http://loki:3100"
default_labels = {env = "prod"}
lookback_minutes = 30
error_spike_threshold = 2.0     # ratio vs baseline to trigger proactive alert
```

### Elasticsearch / Kibana

- **Full-text search** - agent searches the configured index pattern for error strings, request IDs, or trace IDs extracted from the conversation
- **Aggregation queries** - "how many 500s in the last hour?" triggers an aggregation query; result is injected as a table
- **Kibana link generation** - every log fetch appends a Kibana deep-link so participants can open the full context in one click

```toml
[integrations.elasticsearch]
enabled = true
url = "https://elastic:9200"
api_key = "..."                 # or $ES_API_KEY
index_pattern = "logs-*"
kibana_url = "https://kibana:5601"
```

### Postgres

- **Slow query log** - tails `pg_stat_statements` for queries exceeding a threshold; slow queries are injected with the query text, execution plan summary, and the calling service if identifiable
- **Lock wait detection** - monitors `pg_locks` and `pg_stat_activity`; deadlocks and long waits are surfaced immediately
- **Schema lookup** - when a table name appears in conversation, agent fetches column definitions and recent migration history

```toml
[integrations.postgres]
enabled = true
dsn = "postgresql://user:pass@host:5432/db"  # or $POSTGRES_DSN
slow_query_threshold_ms = 1000
watch_locks = true
```

### GitHub / GitLab

- **Blame and history** - when a file path or function name appears in conversation, agent runs `git blame` and surfaces the last author, commit message, and PR/MR link
- **PR/MR context** - recent merges to the affected service are injected at session start; agent flags any merge in the 24h before the incident
- **CI pipeline status** - pipeline results for the affected service are streamed into the transcript as they complete
- **Write: unit test generation** - agent can generate a failing unit test that reproduces the bug described in the conversation and open a draft PR/MR with it
- **Write: CI trigger** - agent can push a branch and trigger a pipeline run directly from the session; result is appended to the transcript when complete

```toml
[integrations.github]
enabled = true
token = "ghp_..."               # or $GITHUB_TOKEN
repos = ["org/api", "org/worker"]
watch_ci = true
write_back = false              # set true to allow test generation and CI triggers

[integrations.gitlab]
enabled = false
url = "https://gitlab.your-org.com"
token = "glpat-..."             # or $GITLAB_TOKEN
projects = ["org/api", "org/worker"]
watch_pipelines = true
write_back = false
```

**Unit test + CI flow**

When the agent identifies a reproducible failure path from the logs and conversation:

1. Agent generates a minimal unit test targeting the failing code path
2. Appends the test to `ai_responses.md` for participant review
3. If `write_back = true` and a participant approves (reacts with `+1` in Slack), agent opens a draft PR/MR with the test
4. CI pipeline triggers automatically; results stream back into the session transcript
5. Pass/fail is noted in `postmortem_draft.md` as a validation step

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

## Related projects

Plain-text and local-file knowledge systems that share the same storage philosophy as sboard:

- [DokuWiki](https://www.dokuwiki.org) - the classic (2004). PHP wiki on flat `.txt` files, no database, still actively maintained. The reference point for filesystem-native team wikis.
- [Obsidian](https://obsidian.md) - local vault of `.md` files, rich linking, no cloud required. Proprietary but free for personal use.
- [Logseq](https://logseq.com) - open source outliner on local markdown files, knowledge graph, daily notes.
- [Foam](https://foambubble.github.io/foam/) - VS Code extension that adds bidirectional links to any existing markdown folder.
- [TiddlyWiki](https://tiddlywiki.com) - entire wiki in a single HTML file, runs in browser, no server.
- [MkDocs](https://www.mkdocs.org) - renders a folder of markdown into a browsable static site, one YAML config, no database.
- [Markopolis](https://pypi.org/project/markopolis/) - open source self-hostable web app, point it at a directory of `.md` files, serves with full-text search.
- [WikidPad](https://wikidpad.sourceforge.net) - desktop wiki notebook (2003), all data local, cross-linked plain text files.
- [PmWiki](https://www.pmwiki.org) - flat-file PHP wiki with a [documented rationale](https://www.pmwiki.org/wiki/PmWiki/FlatFileAdvantages) for avoiding databases.
- [Google Open Knowledge Format](https://flowtivity.ai/blog/google-open-knowledge-format/) - open spec (June 2026) for storing organizational knowledge as markdown files with YAML frontmatter, designed for AI agent access.

## License

MIT
