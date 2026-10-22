# runcoach decisions, in brief

Agreed 18 to 23 October 2026. The full reasoning is in `plan.html`; this is the version that reads on GitHub.

## Version 1

A stranger can install the app, create an account, answer the onboarding questions, get a plan that respects their schedule and fitness, see today's workout, log runs in seconds, give feedback, watch the plan adapt with a stated reason, manage or pause the plan, see their progress, and delete their account. The server survives a provider outage, a double-submitted log, and a bad LLM proposal without corrupting anyone's plan.

Out: GPS tracking in the app, guided audio, race recommendations, social, social sign-in, Keycloak, Kubernetes, a second LLM provider. Behind a flag: Health Connect and HealthKit import of runs.

## Architecture

- Microservices from the start, by Yogesh's decision, knowing the cost: identity, training, coach, notification, behind Spring Cloud Gateway.
- Training keeps the whole domain; splitting it would create distributed-consistency problems for no gain.
- Coach is deliberately dumb: prompt, call, return. No domain tables.
- One MySQL server, a schema per service, no cross-schema queries.
- RabbitMQ via Spring Cloud Stream, outbox on every publisher, idempotent consumers, dead-letter queue. Kafka later as an exercise.
- Resilience4j on every cross-service and provider call. OpenTelemetry and Grafana for observability.
- Config by environment variables and Compose service names; config server and Eureka are optional add-ons.
- Seven containers; about 4 GB of RAM.

## Accounts

- Email and password, hand-built JWT access tokens (about 15 min) and database-stored refresh tokens (about 30 days, rotated, revocable). Keycloak in version 2.
- Every user-owned row carries the user ID; no endpoint takes one from the client.
- Email verification before plan generation. Reset by single-use expiring link.
- Deletion: 14-day deactivation window, then a purge saga across services.

## Onboarding, profile, goal

- One request at the end; the app holds a draft. Server re-validates everything.
- Profile and goal are separate, versioned entities. Metres and seconds, integers.
- Injury information is structured fields only.
- Feasibility is its own endpoint and is re-run inside generation. The verdict carries weeks available, minimum and typical, inputs used, allowed actions and wording. The app renders; it does not compose.
- Rules table is versioned; every plan records its version.

## Plan generation

- The LLM (Anthropic, one provider for now, behind a port) generates the whole plan. Java validates. No template fallback. One repair round, then honest "try again later" with answers kept.
- Background job with a status endpoint; one in-flight job per user.
- Accepted plans cached by a hash of inputs plus rules and prompt versions.
- Limits: 5 generations per user per day, shown in the app; 5 adaptations per week; global daily token budget with alert and hard stop.

## Scheduling

- Scheduled workouts are rows with a local calendar date; the profile stores the time zone. Event timestamps are UTC.
- Missed is marked lazily when the user opens the app; scheduler in version 2.
- Statuses: scheduled, completed, partial, missed, skipped, moved.
- Reschedule within the week only, from server-provided allowed days; no plan version bump. Cross-week moves go through the revision engine with confirmation and must never raise next week's load as a side effect.
- Optimistic locking per workout and per plan.

## Logging and feedback

- One request, prefilled; Done works without touching anything.
- Results are set-by-overwrite per workout (PUT). Free runs carry an app-generated UUID. Events publish only on real change.
- Edits any time, deletes allowed. Untouched defaults are saved but flagged.
- Health import proposes and the user confirms; manual logging is the base path. Completion threshold about 90 percent.

## Adaptive coaching

- Java decides whether to adapt and in what scope; the LLM proposes inside that scope.
- Eight reason codes: PAIN_REPORTED, EFFORT_TOO_HIGH, RUNNER_EASED_SESSIONS, MISSED_SESSIONS, EXTRA_VOLUME, READY_FOR_MORE, CROSS_WEEK_MOVE, PROFILE_CHANGED. Pain outranks everything.
- Easing applies automatically with an undo; increases and restructures ask. Pain never leads to an increase.
- Runner self-adjustments (reduce intensity, reduce distance, convert to easy, move) are deterministic Java. "Not feeling well" skips and routes to the pause or pain path.
- Every change is a revision row; undo is a compensating revision; apply uses compare-and-apply against the plan version read at the start.

## Progress and records

- Computed on read. Measured versus estimated, labelled.
- Records within about 2 percent of standard distances, labelled with their source.

## Managing the plan

- One active plan per user; states active, paused, ended, completed; everything kept.
- Restart is a new plan. Goal or schedule changes regenerate the weeks ahead only, with confirmation.
- Resume after pause: parked. Placeholder is to re-run feasibility with the weeks left.

## Cross-cutting

- API versioning by path, additive only; app build number in a header.
- Coach sentences are server-generated from reason codes and templates.
- Hosting: nothing until the checkpoint; then Oracle Cloud's free ARM tier in Mumbai, or AWS EC2 with a Cloudflare tunnel on an evening schedule.

## Timeline

Checkpoint, an interview-ready backend, around month 7. Backend version 1 around month 10. App store around month 13 to 15.
