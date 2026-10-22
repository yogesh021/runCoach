# runcoach

A running-coach backend, built as a learning project in Java and Spring Boot. runcoach turns a runner's answers about fitness, goals and schedule into a personalised training plan, then adapts that plan from what they actually run.

Status: planning complete, build not started. The first code lands in `services/identity-service` (see the pathway below).

## Documents

| Document | What it is | Where to read it |
|---|---|---|
| Backend plan | Decision log, architecture, roadmap, phase 1 sessions | [`docs/plan.html`](docs/plan.html), or the live page on claude.ai |
| Pathway | Interactive step-by-step learning and implementation tracker | [`docs/pathway.html`](docs/pathway.html), or the live page on claude.ai where progress is saved |
| Decisions in brief | Markdown summary of the decisions, for reading on GitHub | [`docs/DECISIONS.md`](docs/DECISIONS.md) |
| App design | Phone prototype sources (Design canvas format) | [`docs/design/`](docs/design/), rendered on the runcoach app design canvas on claude.ai |

The HTML documents render best through GitHub Pages or opened locally. The pathway's progress store only works on the claude.ai copy; the file here keeps progress in the browser only.

## Architecture in one paragraph

Four Spring Boot services behind a Spring Cloud Gateway: identity (accounts, hand-built JWT and refresh tokens), training (the whole domain: profile, goal, plan, workouts, results, revisions, rules), coach (the LLM integration: prompts, provider adapter, limits, cache, log), and notification (email, later push). One MySQL server with a schema per service. RabbitMQ through Spring Cloud Stream for events, with the outbox pattern and idempotent consumers. The LLM proposes plans; Java validates every one against versioned training rules before anything is saved.

## Principles

- The client expresses intent; the server decides.
- LLM output is untrusted input.
- Whoever decides also explains.
- Nothing changes silently, and nothing is deleted.
- Never block logging.
- Measured versus estimated, always labelled.

## Planned layout

```
services/
  identity-service/
  training-service/
  coach-service/
  notification-service/
  gateway/
infra/
  docker-compose.yml
docs/
```

## Author

Yogesh Tripathi. Mentoring, reviews and the mobile app by Claude.
