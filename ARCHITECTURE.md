# Recap — Architecture

A short tour of how the system fits together. The goal of this doc is to make the design choices defensible without paging the entire source code.

---

## High-level

```
┌──────────────────────────────────────────────────────────────┐
│                       Browser                                │
│   Jinja2 templates + Alpine.js + Quill + Bootstrap utilities │
│   WebSocket client for live transcription                    │
└────────────────┬─────────────────────────────────────────────┘
                 │  HTTPS + WSS
┌────────────────▼─────────────────────────────────────────────┐
│                Gunicorn (Render web service)                 │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │  Flask app — 19 blueprint routes (auth, contacts,        │ │
│ │  notes, emails, calendar, search, imports, ...)          │ │
│ │  + flask-sock WebSocket route for live STT               │ │
│ └────┬─────────────────────────────────────┬───────────────┘ │
│      │                                     │                 │
│  ┌───▼────────────┐   ┌──────────────┐   ┌▼──────────────┐   │
│  │  services/     │   │  webapp/db   │   │  webapp/      │   │
│  │  (AI, calendar │   │  adapter:    │   │  storage.py:  │   │
│  │  transcription │   │  SQLite or   │   │  R2 (prod) /  │   │
│  │  embeddings,   │   │  Postgres    │   │  FS (dev)     │   │
│  │  style learn.) │   │              │   │               │   │
│  └─┬──────┬─────┬─┘   └──────┬───────┘   └───────┬───────┘   │
└────┼──────┼─────┼────────────┼───────────────────┼───────────┘
     │      │     │            │                   │
     ▼      ▼     ▼            ▼                   ▼
┌─────────┐ ┌─────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────┐
│Anthropic│ │ Voyage  │ │  Deepgram   │ │   OpenAI    │ │ Google   │
│Claude   │ │  AI     │ │  (Nova-3 +  │ │ (Whisper +  │ │ Calendar │
│Sonnet   │ │ (embeds)│ │  streaming) │ │  GPT-4o-tr) │ │ (OAuth)  │
│ 4.5     │ │         │ │             │ │             │ │          │
└─────────┘ └─────────┘ └─────────────┘ └─────────────┘ └──────────┘
```

---

## Why Flask + Jinja + Alpine instead of React/Next

**Constraint:** I wanted to ship fast and have one repo, one process, one deploy. The product is mostly information-dense forms and tables — not an interactive single-page experience.

**Choice:** Server-rendered Jinja templates with Alpine.js for the small handful of reactive components (the email drafter, the contact detail Quill editor, the calendar). Bootstrap 5 for utility classes, hand-rolled CSS for the design system.

**Result:** Zero build step. No `node_modules`. No hydration overhead. Pages load fast because they're plain HTML. The whole front end is ~7000 lines of templates + ~500 lines of CSS + a few hundred lines of Alpine sprinkles.

**Where this is wrong:** If the product were a real-time collaborative interface (Linear, Figma), this stack would be wrong. For a CRM with mostly form-driven UX, it's right.

---

## The DB adapter

`webapp/db.py` is ~340 lines of "make SQLite and Postgres behave the same". The contract:

- Same SQL strings work against both. The adapter handles `?` (SQLite) vs `%s` (Postgres) placeholder translation.
- `last_insert_rowid()` (SQLite) gets translated to `lastval()` (Postgres) at the cursor wrapper layer.
- `LIKE` becomes `ILIKE` on Postgres for case-insensitive search.
- `BLOB` storage for embeddings is `BYTEA` on Postgres.
- Date/timestamp columns are stored as `TIMESTAMP` on Postgres but ISO strings on SQLite.

Migrations are split: `migrate.py` for SQLite, `migrate_pg.py` for Postgres. Both are idempotent — they only add columns/tables that don't exist. Run at app boot.

**Why two migration files instead of one ORM?** I started with SQLite + bare Flask, then needed Postgres for Render. Adding SQLAlchemy mid-project would have been a 2-day refactor. Splitting the migration runner was a 1-hour adapter. The trade-off pays off as long as new schema changes stay small.

---

## AI service layering

The AI logic is in `webapp/services/ai_service.py` (~2000 lines, single file). Each function is stateless and takes:

- Whatever context it needs (contact row, notes, interactions, profile, etc.)
- API key + model name
- Optional learning context (style profile, recent approved drafts, ...)

It returns plain data. No globals, no class state. Routes call these as black boxes.

The most complex function is `generate_email_draft` — it builds a layered prompt:

```
SYSTEM PROMPT
├── About the sender
├── 9 numbered rules (banned phrases, CTA discipline, closing rules,
│   detection logic, structure & spacing, voice rules, no placeholders,
│   sign-off formatting, subject line format, output JSON shape)
└── 3 sub-rules for special cases (3a closing, 3b interview detection,
    4a voice, 4b no double-enthusiasm)

USER MESSAGE
├── Contact context (name, company, role, notes, interactions, summary)
│   ├── TODAY'S DATE: 2026-05-14 (Thursday)
│   └── MOST RECENT INTERACTION: 2026-05-14 - that is today...
├── PURPOSE: <user-provided>
├── BANNED PHRASES (before examples)
├── Global style profile  (compounded across all approvals)
├── Template style profile (compounded across this category)
├── Per-contact style profile (compounded for this person)
├── Recent approved emails (for style reference)
├── BANNED PHRASES (repeated, after examples)
└── FINAL REMINDER (last thing the model reads)
```

Then on the response side:

```
RAW MODEL JSON
├── _sanitize_output:    strip em/en dashes, regex-fix obvious typos
└── _force_signoff:      strip & re-append Best,\n{FirstName} block
```

Every generation logs `model + input_tokens + output_tokens + cost_usd + latency_ms + feature + metadata` to `ai_usage` so I can answer cost / latency questions with SQL.

---

## Style learning (the part that compounds)

When a user approves a draft (thumbs up), or edits the draft text and then sends it, I diff the AI output against the final text and extract a `StyleProfile`:

```python
class StyleProfile(TypedDict):
    avg_length_chars: int
    preferred_openers: list[str]
    preferred_closings: list[str]
    phrases_to_avoid: list[str]
    tone_descriptors: list[str]
    paragraph_count_distribution: dict[int, int]
    common_subject_patterns: list[str]
    # ...
```

The profile is computed at three scopes:
- **Global** — across every email the user has ever approved.
- **Per-category** — for "thank-you", "intro", "follow-up", etc.
- **Per-contact** — for emails specifically to this person.

Next time the user drafts, all three profiles get rendered into the prompt. Edits the user made to bury "circle back" or replace formal sign-offs with "Best," compound across approvals. So the AI's voice converges on the user's actual voice.

The diff/extraction step itself is also an LLM call (Claude reading the AI draft + the user's edited version) so the system can pick up subjective things like tone shifts, not just literal phrase replacements.

---

## The transcription pipeline

```
audio (recorded or uploaded)
     │
     ▼
┌────────────────────────────────────────┐
│ transcription_service.py routing layer │
│ - has Deepgram key + not 'summary' mode? → Deepgram path
│ - else                                  → OpenAI Whisper path
└────────────────────────────────────────┘
     │                                  │
     ▼                                  ▼
┌──────────────────────┐    ┌────────────────────────────┐
│ Deepgram Nova-3      │    │ OpenAI Whisper / GPT-4o-tr │
│ - keyterms = contact │    │ - prompt = contact names   │
│   names + companies  │    │   joined (biases output)   │
│   (live biasing)     │    │ - no diarization (single   │
│ - diarize=true       │    │   speaker stream)          │
│ - returns segments   │    │                            │
│   [speaker, text]    │    │                            │
└──────────┬───────────┘    └────────────┬───────────────┘
           │                             │
           ▼                             ▼
┌──────────────────────────────────────────────────────┐
│ Layer 3: ai_service.correct_proper_nouns             │
│ - Pass transcript + contact names/companies/schools  │
│   to Claude, ask it to snap misheard tokens.         │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│ Speaker rendering                                    │
│ - Deepgram path: render_conversation(segments,       │
│   you_name=user, them_name=contact)                  │
│ - Whisper path: ai_service.label_speakers (Claude    │
│   infers speaker turns from context)                 │
└──────────┬───────────────────────────────────────────┘
           │
           ▼
   Final speaker-labeled transcript saved on the contact
```

Live streaming uses Deepgram's WebSocket endpoint via `flask-sock`. The client opens a WS to my server, my server opens a WS to Deepgram, audio chunks flow client → my server → Deepgram, transcripts flow back the same path with low-latency partials.

---

## Calendar integration

`calendar_service.py` is the larger of the two non-AI services. Three notable design choices:

1. **All-calendars fetch.** Most "Google Calendar + your app" stacks fetch only `primary`. Real users have multiple calendars (work, school, shared, subscribed). The fetcher calls `calendarList().list(showHidden=True)` to discover every calendar the user can read, fetches events from each, paginates via `nextPageToken`, and de-dupes on `iCalUID` because Google returns the same event multiple times when it's on shared calendars.

2. **Explicit timezone.** Pass `timeZone=<user's primary cal tz>` to every `events.list()` call. Without this, Google returns event `dateTime` in whatever zone the *event* was created in — usually UTC for imported events. This fixed a 5-hour display bug where "Sam <> Robert" at 5pm Central showed as 10pm.

3. **Networking-only filter.** The month view shows ONLY networking events. Classification ORs three signals:
   - Has any attendee → networking.
   - Title matches a curated keyword list (call, coffee, intro, 1:1, interview, ...) → networking.
   - Title contains a token from a 400-name first-name set → networking. (Catches "Sam <> Robert" when neither signal 1 nor 2 fires.)
   - Names that double as common English words (May, June, Hope, Pat, Will) are deliberately excluded from the name set to keep false positives low.

The view caches each `(user, start, end)` fetch for 5 minutes in-process. Prev/next/today links prefetch their target URLs on hover AND immediately on page load, so navigation feels instant after the first click.

---

## Deploy

- **Code lives** on GitHub. Push to `main` triggers Render's auto-deploy.
- **Web service** = single Gunicorn process running the Flask app.
- **Postgres** = Render-managed database, connected via `DATABASE_URL`.
- **Object storage** = Cloudflare R2 (S3-compatible) for uploaded audio + documents.
- **DNS + edge** = Cloudflare. `recap.network` resolves via proxied CNAME to the Render service. SSL auto-provisions.
- **Secrets** = Render env vars: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `DEEPGRAM_API_KEY`, `VOYAGE_API_KEY`, `GOOGLE_OAUTH_CLIENT_SECRET`, `R2_*`, `FERNET_KEY`, etc. The Fernet key encrypts user OAuth refresh tokens at rest.

Local dev is symmetric:
- SQLite file instead of Postgres
- Local filesystem instead of R2
- Same Flask app, same code paths.

`DATABASE_URL` being absent is the only signal — when it's not set, the adapter falls back to SQLite. No code changes between dev and prod.

---

## What I'd do differently next time

A few honest takeaways from building this:

- **Pick Postgres from day one.** The SQLite-to-Postgres adapter cost me maybe 4 hours of subtle bugs (placeholder differences, `last_insert_rowid` vs `lastval`, `LIKE` vs `ILIKE`, BYTEA vs BLOB). Worth it because I already had data in SQLite, but a greenfield project should start on Postgres.
- **Type the prompts.** I have ~30 prompt-building functions. They're all `str.format` or f-string concatenation. A typed `PromptBuilder` class with a slot system would prevent the "I forgot to escape a curly brace" class of bug I keep hitting.
- **Snapshot test the prompts.** When I change a prompt to fix one bug, I often regress another. A `pytest-snapshot` setup would catch that.
- **More integration tests around AI.** I have unit tests for sanitizers and signoff rewriting. I don't have end-to-end "draft an email and check the output" tests with a recorded Claude response, because mocking LLM output well is annoying. Worth doing.
