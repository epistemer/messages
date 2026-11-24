# Roadmap

The design is [brief.md](brief.md). Ownership: [ownership.md](ownership.md),
delegated by epistemer on 2026-11-24. First of the three new neozp packages
(neozp decision D5).

## M0: design

| ✓ | Item | Done when |
|---|---|---|
| ☑ | M0.1 `docs/brief.md`: record of needs, idea, decisions, model, open questions with defaults | written (2026-11-24) |
| ☑ | M0.2 A review of the brief by the other model family | its findings fixed or answered ([review 0001](reviews/0001-brief.md), 2026-11-24) |
| ☐ | M0.3 epistemer answers the brief's open questions (§11), or takes the defaults | answered |
| ☐ | M0.4 Spikes S1 (SMTP), S2 (the outbox), S3 (auth in Confab), each recorded as an ADR | ADRs written |
| ☐ | M0.5 `docs/interface.md` with a cross-model review, locked by epistemer | locked |

## M1 onwards: build, planned at the lock

Expected: the core (model, rendering, outbox, loop), `messages/smtp`,
`messages/dir`, `messagestest`, `messagesschema`, `authmessages`, the docs
and the `messages` skill, then v0.1.0. Confab then deletes its outbox, which
closes that part of neozp roadmap I1.
