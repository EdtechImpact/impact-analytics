# EdTech Impact Analytics — Event Contract

**Version 1.2**

Send us your product's event data and we turn it into comparable usage, adoption
and impact analytics for the schools you work with.

You don't need to redesign your tracking. Wrap your existing events in a light
envelope, keep your original payload intact, and enrich over time.

---

## Contents

- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [Authentication](#authentication)
- [Endpoint](#endpoint)
- [Required fields](#required-fields)
- [Timestamps](#timestamps)
- [Optional envelope fields](#optional-envelope-fields)
- [The learner concerned](#the-learner-concerned)
- [Corrections and retractions](#corrections-and-retractions)
- [Domains](#domains) — [actor](#actor), [activity](#activity), [result](#result), [client](#client), [context](#context), [attributes](#attributes)
- [Responses](#responses)
- [Errors and retries](#errors-and-retries)
- [Rate limits](#rate-limits)
- [Idempotency](#idempotency)
- [Testing](#testing)
- [Data protection requirements](#data-protection-requirements)
- [Retention and erasure](#retention-and-erasure)
- [Troubleshooting](#troubleshooting)
- [Enum reference](#enum-reference)
- [Limits reference](#limits-reference)
- [Changelog](#changelog)

---

## Quick start

Three things to get going: your `product_id` (we issue it during onboarding),
your API token, and a stable unique id for each event.

```bash
curl -X POST https://im.pact.workers.dev/v1 \
  -H "Authorization: Bearer $EDTECH_IMPACT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "schema_version": "1.2",
      "event_id": "018f9b34-3b6f-7a60-9a3e-9d6a2b2a4e6c",
      "product_id": "your-product-id",
      "school_id": "sch_test_000",
      "event_name": "session_started",
      "occurred_at": "2026-03-01T10:00:00Z",
      "actor_id": "usr_anon_abc123"
    }
  ]'
```

```json
{
  "request_id": "req_91df2c",
  "status": "accepted",
  "accepted": 1,
  "rejected": 0,
  "errors": [],
  "results": [
    { "index": 0, "event_id": "018f9b34-3b6f-7a60-9a3e-9d6a2b2a4e6c", "status": "accepted" }
  ]
}
```

`sch_test_000` is our test school: events are accepted but excluded from all
analytics, so it's safe to use while you build. Add `?dry_run=true` to validate
without storing anything at all.

### Getting started checklist

1. Send the minimum payload above against `sch_test_000` and confirm a 200.
2. Add your real `school_id` values and send us the list so we can map them to ours.
3. Add the `raw` field so your original payload is preserved.
4. Add `activity` — this is what unlocks content-level analysis.
5. Add `result` — this is what turns usage reporting into impact evidence.
6. Add `context` for course and assignment-level insight, and `learner` for
   events performed on a learner's behalf or concerning a whole class.

---

## How it works

Your entire original event goes into `raw`, untouched. Around it, a small
envelope tells us who did what, when, and to which learner. We map your event
names to our taxonomy, resolve your school identifiers to ours, and generate
analytics that are comparable across products.

We analyse across four dimensions. The more complete each one is, the more we can
tell you:

| Dimension | Fields | What it gives you |
|---|---|---|
| **Who** | `actor`, `learner` | Who is using the product, and which learner the event concerns |
| **What** | `activity` | Which content, subject and topic is involved |
| **Result** | `result` | Measurable outcomes rather than usage counts |
| **Where** | `context` | Course, assignment and pathway placement |

Anything you don't send, we attempt to derive from `raw` — but a field sent
directly is always more accurate than one inferred.

---

## Authentication

```
Authorization: Bearer your-api-token
```

Tokens are issued during onboarding and are scoped two ways:

- **To a single `product_id`.** An event whose `product_id` doesn't match your
  token's scope is rejected.
- **To a set of permitted request origins.** Tell us during onboarding which
  origins your requests will come from. Browser-based requests are checked against
  the `Origin` header; server-to-server requests, which send no `Origin`, need a
  token issued for that use. Requests from an origin the token doesn't permit are
  rejected with `401`.

If your integration sends from both a browser and a backend, ask us for separate
tokens rather than widening one — it keeps a compromised browser token from being
usable server-side.

---

## Endpoint

```
POST https://im.pact.workers.dev/v1
Content-Type: application/json
```

Handles both real-time and batch delivery. Requests must contain an **array**,
even for a single event.

### Query parameters

| Parameter | Values | Description |
|---|---|---|
| `dry_run` | `true` | Validate only. Returns the normal response; stores nothing. |
| `results` | `all` (default), `errors`, `none` | How much of the per-event `results` array to return. |

---

## Required fields

| Field | Type | Description |
|---|---|---|
| `schema_version` | string | `"1.2"` for new integrations. `"1.1"` remains supported. |
| `event_id` | string | Stable unique identifier for the event. UUIDv7 recommended. **The same value on every retry** of the same event, and never reused for a different event. See [Idempotency](#idempotency). |
| `product_id` | string | Your EdTech Impact application identifier. Must match your token's scope. |
| `school_id` | string | Your own school identifier. We maintain the mapping to ours. |
| `event_name` | string | Your event name. No need to match our taxonomy — we map it. |
| `occurred_at` | timestamp | When the event happened. See [Timestamps](#timestamps). |
| `actor_id` | string | Pseudonymous identifier of whoever performed the action. Never an email, name, UPN or phone number. |

### Minimum payload with your original event

```json
[
  {
    "schema_version": "1.2",
    "event_id": "018f9b34-3b6f-7a60-9a3e-9d6a2b2a4e6c",
    "product_id": "your-product-id",
    "school_id": "sch_alkhor_001",
    "event_name": "entered_study_mode",
    "occurred_at": "2026-02-20T08:31:00Z",
    "actor_id": "usr_anon_abc123",
    "raw": {
      "event_name": "entered_study_mode",
      "occurred_at": "2026-02-20T08:31:00.000000+00:00",
      "payload": {
        "session_id": "019d1a2b-0001-7abc-def0-111111111111",
        "category": "study",
        "properties": {
          "feature": "study_mode",
          "lesson_uid": "019c094a-0cf4-7f31-ac1c-cfddc2a06a54",
          "subject": "MATHEMATICS",
          "grade": 9
        }
      }
    }
  }
]
```

### School identifiers

Send whatever identifier you use internally. During onboarding we'll ask for a
list so we can build the mapping:

```
Your school ID      Our school ID
sch_alkhor_001  →   eti_sch_7f3a2b
sch_doha_002    →   eti_sch_9c4d1e
```

An unmapped `school_id` is still accepted and stored — we'll flag it and work with
you to add it. If you already hold our identifier, send it as `eti_school_id` and
we skip the lookup.

---

## Timestamps

`occurred_at` and `sent_at` must be ISO-8601 / RFC-3339 **including a UTC
offset**.

**Accepted**

```
2026-03-01T10:22:00Z
2026-03-01T10:22Z                    seconds optional
2026-03-01T10:22:00.123Z             fractional seconds, up to 9 digits
2026-02-20T08:31:00.000000+00:00
2026-03-01T10:22:00+05:30
2026-03-01 10:22:00+00:00            space separator accepted
```

**Rejected**

| Value | Why |
|---|---|
| `2026-03-01T10:22:00` | No offset. A local time read as UTC shifts every measure by the school's offset, and the error is invisible once stored. |
| `2026-03-01` | Date only. |
| `1772450520` | Epoch forms aren't accepted. |
| `2026-02-31T00:00:00Z` | Not a real date. |
| More than 48 hours ahead | Indicates a clock problem on the sending system. |
| Earlier than 2000-01-01 | Usually an epoch-zero or parsing artefact. |

---

## Optional envelope fields

| Field | Type | Description |
|---|---|---|
| `eti_school_id` | string or integer | Our school identifier, if you hold it. Skips our lookup. |
| `event_source` | string | Where the event came from: `mixpanel`, `sdk`, `segment`, `webhook`, `batch`. Free text, up to 64 characters. |
| `event_kind` | string | `interaction`, `state_change` or `system`. Administrative events (`state_change`) are excluded from engagement measures. |
| `sent_at` | timestamp | When your system sent the event. |
| `event_sequence` | integer | Ordering index within a session, where timestamps lack precision. |
| `product_version` | string | Your application version at the time of the event. |
| `raw` | object | Your original payload, preserved untouched. |
| `learner` | object | The learner or group the event concerns. See below. |
| `revision` | integer | Corrections. See below. |
| `voided` | boolean | Retractions. See below. |

**Unknown fields are accepted**, stored, and ignored by our analytics until we add
support for them. You can send extra fields safely — but a field we don't
recognise won't appear in your reports, so tell us if there's something you'd like
us to use.

---

## The learner concerned

`learner` identifies which learner — or which group — an event concerns, for the
cases where that isn't simply the actor. **Most events don't need it:** when a
learner acts for themselves, `actor` is sufficient. Send `learner` when someone
acts *on a learner's behalf*, or when an event concerns a class or cohort rather
than one person.

| Field | Type | Description |
|---|---|---|
| `learner.id` | string | Pseudonymous learner identifier. Omit for group-level events. May equal `actor_id`, though a `learner` object isn't needed when the learner acted themselves. |
| `learner.role` | string | `student`, `teacher`, `admin`, `system` |
| `learner.class_id` | string | Class or group: `7B`, `maths-set-1` |
| `learner.cohort_id` | string | Intervention or research group: `pupil_premium` |
| `learner.grade_index` | integer | Years of formal schooling, 0–20. 0 = Reception, 7 = Year 7. |
| `learner.education_level` | string | `early-childhood`, `primary`, `lower-secondary`, `upper-secondary` |
| `learner.national` | object | Country-specific metadata. UK: `{ "yearGroup": "year-7", "keyStage": "ks3" }` |

Educational context belongs here rather than on the actor because it describes the
learner. A teacher's `grade_index` is meaningless, and their `class_id` is
ambiguous — they teach several. **`actor` says who acted and how; `learner`
identifies the learner the event concerns.**

### Which pattern to use

**1. The learner acted themselves** — the common case, and **no `learner` object
is needed.** Where the actor is a learner, we treat them as the learner, and their
`actor` context supplies class, cohort and year group.

```json
{
  "actor_id": "usr_anon_abc123",
  "actor": { "role": "student", "session_id": "sess_019d1a2b0001", "class_id": "7B", "grade_index": 7 }
}
```

Sending `learner` here as well is accepted — `learner.id` may equal `actor_id` —
but it isn't required and adds nothing.

**2. Someone acted for a learner** — a teacher uploading a submission. `actor.role`
is required here so we can attribute the action correctly.

```json
{
  "event_name": "submission_uploaded",
  "actor_id": "tch_anon_991",
  "actor": { "role": "teacher" },
  "learner": { "id": "usr_anon_learner_42", "role": "student", "class_id": "7B", "grade_index": 7 }
}
```

**3. The event concerns a group** — a teacher setting work for a class. Send the
group identifier and **no `learner.id`**: the event didn't observe anything about
any particular learner. We resolve membership from roster data at analysis time,
across the reporting period rather than at one instant.

```json
{
  "event_name": "assignment_created",
  "actor_id": "tch_anon_991",
  "actor": { "role": "teacher" },
  "learner": { "class_id": "7B", "grade_index": 7 },
  "context": { "assignment_id": "hw-fractions-2026-03" }
}
```

Include `context.assignment_id` so the assignment can be joined to the submissions
that follow.

**Several known learners** — group submission, peer assessment — should be sent as
**one event per learner**, each with its own `event_id`, sharing
`context.assignment_id` or `activity.parent_id` so we can correlate them. We don't
accept an array of learner identifiers: it would assert per-learner facts the
event didn't observe, and it complicates corrections and erasure.

### Rules

- **`learner.id` must use the same pseudonym space as `actor_id`.** The same
  learner must carry the same identifier whether they acted themselves or were
  acted upon. Without this, learner-level measures can't be computed.
- The `learner` object must include **at least one of `id`, `class_id` or
  `cohort_id`**.
- **`actor.role` is required when `learner.id` differs from `actor_id`.**
- There is **no top-level `learner_id`** field. Use `learner.id`.

### How we attribute it

- **Usage and engagement** measures count `actor_id`.
- **Learning and outcome** measures attribute to `learner.id` where it is sent.
  Where it isn't and the actor is a learner, we treat the actor as the learner —
  so a straightforward learner event needs no `learner` object at all. Events
  performed by a teacher or system with no `learner.id` attribute to no learner,
  so they never inflate learner activity.
- **Educational context** (`class_id`, `cohort_id`, `grade_index`,
  `education_level`) resolves from `learner` first, falling back to `actor`.

---

## Corrections and retractions

> **Availability:** contact us before sending `revision` above 0 or `voided`.
> These are enabled per integration and are rejected with an explanatory error
> until then.

| Field | Type | Description |
|---|---|---|
| `revision` | integer | Correction sequence for this `event_id`. Omitted means 0. |
| `voided` | boolean | Retracts the event. Requires `revision` of 1 or more. |

**To correct an event**, re-send it with the **same `event_id`** and a **higher
`revision`**. The highest revision wins; lower ones are superseded everywhere.
Every revision is retained, so the history stays auditable.

```json
{
  "schema_version": "1.2",
  "event_id": "018f9b34-3b6f-7a60-9a3e-9d6a2b2a4e6c",
  "product_id": "your-product-id",
  "school_id": "sch_alkhor_001",
  "event_name": "question_answered",
  "occurred_at": "2026-03-01T10:22:00Z",
  "actor_id": "usr_anon_abc123",
  "revision": 1,
  "result": { "correct": false, "score": 0.0 }
}
```

**Delivery retries keep the same `revision`.** That's what keeps retries and
corrections unambiguous: a retry is identical, a correction increments.

**To retract an event** sent in error, send a new revision with `"voided": true`.
It's removed from all analytics. Use this for "this shouldn't have been recorded" —
data subject erasure is a separate process, so contact us for that.

---

## Domains

Every domain is optional. Send what you have.

### actor

Who performed the action, and how.

| Field | Type | Description |
|---|---|---|
| `role` | string | `student`, `teacher`, `admin`, `system` |
| `session_id` | string | Your session identifier. If omitted, we infer sessions from timestamps. |
| `country` | string | ISO 3166-1 **alpha-2**: `GB`, `US`, `QA`. Three-letter codes are rejected. |
| `education_level` | string | See [enum reference](#enum-reference) |
| `grade_index` | integer | 0–20 |
| `class_id` | string | The class the actor was working in, where applicable |
| `cohort_id` | string | Intervention or research group |
| `institution_id` | string | MAT, district or local authority above school level |
| `national` | object | Country-specific metadata |

Where a `learner` object is also present, its equivalent fields take precedence
for learner-level analysis.

### activity

The learning content involved. One of the most valuable domains.

| Field | Type | Description |
|---|---|---|
| `id` | string | Your content identifier: lesson UID, question ID, quiz ID. **Strongly recommended** — most content analytics depend on it. |
| `name` | string | Human-readable name: `Unit 2 Fractions` |
| `type` | string | Free text: `lesson`, `quiz`, `question`, `video`, `assessment`, `chat` |
| `subject` | string | Curriculum subject as free text: `mathematics`, `english`, `science`. We map it to our taxonomy. |
| `topic` | string | Free text: `fractions`, `photosynthesis` |
| `difficulty` | string | `beginner`, `intermediate`, `advanced` |
| `ai` | boolean | Was AI involved in this activity |
| `attempt` | integer | Attempt number, 1 or greater |
| `parent_id` | string | Parent activity, e.g. the homework containing this question |
| `assigned_by` | string | Pseudonymous identifier of the teacher who assigned the work |
| `curriculum_standard` | string | Curriculum reference: `NC-KS3-MA-3.1` |

### result

Outcomes. **This is what distinguishes measurable impact from usage counting** —
products that send result data get stronger evidence and more robust benchmarking.

| Field | Type | Description |
|---|---|---|
| `success` | boolean | A non-assessment action completed successfully |
| `correct` | boolean | An assessment response was correct |
| `score` | number | Decimal 0.0–1.0. Outside that range is rejected. |
| `response` | string | The learner's response text |
| `duration_ms` | integer | Time spent, 0 or greater |
| `progress` | number | Decimal 0.0–1.0 |

**`correct` vs `success`:** use `correct` for assessment events such as
`question_answered`; use `success` for non-assessment outcomes such as
`lesson_completed` or `video_completed`. Events shouldn't normally carry both.

### client

| Field | Type | Description |
|---|---|---|
| `platform` | string | `web`, `ios`, `android`, `desktop` |
| `device_type` | string | `mobile`, `tablet`, `desktop` |
| `user_agent` | string | Browser or device user agent |
| `app_version` | string | Application version |

### context

Where the activity sits in structured learning.

| Field | Type | Description |
|---|---|---|
| `course_id` | string | Course identifier |
| `assignment_id` | string | Assignment identifier |
| `learning_path_id` | string | Learning pathway identifier |
| `page_url` | string | URL where the event occurred. **Strip query parameters and fragments** to avoid transmitting personal data. |
| `page_title` | string | Page title |

### attributes

Catch-all for product-specific fields, stored as sent.

- Keys must be **snake_case**, matching `^[a-z][a-z0-9_]*$`. `camelCase` is rejected.
- Keys must be under **64 characters**.
- Don't duplicate structured fields such as `subject`, `score` or `session_id`.

```json
"attributes": {
  "mastery_level": "emerald",
  "mastery_points": 7.5,
  "feature": "study_mode"
}
```

### Full payload example

Every field the contract supports, for reference. You are not expected to send
all of these.

```json
[
  {
    "schema_version": "1.2",
    "event_id": "018f9b34-3b6f-7a60-9a3e-9d6a2b2a4e6c",
    "product_id": "your-product-id",
    "school_id": "sch_alkhor_001",
    "eti_school_id": "34821",
    "event_name": "question_answered",
    "event_source": "mixpanel",
    "event_kind": "interaction",
    "occurred_at": "2026-03-01T10:22:00Z",
    "sent_at": "2026-03-01T10:22:01Z",
    "event_sequence": 3,
    "product_version": "3.2.0",
    "actor_id": "usr_anon_abc123",
    "revision": 0,

    "actor": {
      "role": "student",
      "session_id": "sess_019d1a2b0001",
      "country": "GB",
      "education_level": "lower-secondary",
      "grade_index": 7,
      "class_id": "7B",
      "cohort_id": "pupil_premium",
      "institution_id": "mat_102",
      "national": { "yearGroup": "year-7", "keyStage": "ks3" }
    },

    "learner": {
      "id": "usr_anon_abc123",
      "role": "student",
      "class_id": "7B",
      "cohort_id": "pupil_premium",
      "grade_index": 7,
      "education_level": "lower-secondary",
      "national": { "yearGroup": "year-7", "keyStage": "ks3" }
    },

    "activity": {
      "id": "q42",
      "name": "What is 1/2 + 1/4?",
      "type": "question",
      "subject": "mathematics",
      "topic": "fractions",
      "difficulty": "intermediate",
      "ai": false,
      "attempt": 1,
      "parent_id": "quiz-123",
      "assigned_by": "tch_anon_992",
      "curriculum_standard": "NC-KS3-MA-3.1"
    },

    "result": {
      "correct": true,
      "score": 1.0,
      "response": "3/4",
      "duration_ms": 12400,
      "progress": 0.65
    },

    "client": {
      "platform": "web",
      "device_type": "tablet",
      "user_agent": "Chrome/122.0",
      "app_version": "3.2.0"
    },

    "context": {
      "course_id": "course-maths-y7",
      "assignment_id": "hw-fractions-2026-03",
      "learning_path_id": "path-fractions",
      "page_url": "https://app.example.com/lesson/fractions",
      "page_title": "Adding Fractions"
    },

    "attributes": {
      "feature": "study_mode",
      "mastery_level": "emerald",
      "mastery_points": 7.5
    },

    "raw": { "…": "your original event" }
  }
]
```

---

## Responses

### Fully accepted

```json
{
  "request_id": "req_91df2c",
  "status": "accepted",
  "accepted": 2,
  "rejected": 0,
  "errors": [],
  "results": [
    { "index": 0, "event_id": "018f…aa01", "status": "accepted" },
    { "index": 1, "event_id": "018f…aa02", "status": "accepted" }
  ]
}
```

### Partially accepted — also HTTP 200

```json
{
  "request_id": "req_91df2c",
  "status": "partial",
  "accepted": 1,
  "rejected": 1,
  "errors": [
    { "index": 1, "event_id": "018f…4e6c", "field": "school_id", "message": "is required (string)" }
  ],
  "results": [
    { "index": 0, "event_id": "018f…aa01", "status": "accepted" },
    { "index": 1, "event_id": "018f…4e6c", "status": "rejected",
      "errors": [ { "field": "school_id", "message": "is required (string)" } ] }
  ]
}
```

| Field | Description |
|---|---|
| `request_id` | Quote this when raising a query with us |
| `status` | `accepted` (all), `partial` (some rejected), `rejected` (none accepted) |
| `accepted` / `rejected` | Event counts |
| `errors[]` | Per **field** errors. `index` is the position in your submitted array. |
| `results[]` | Per **event** outcome, in submission order |

Two things worth building against:

> **A 200 doesn't mean every event was accepted.** If any event in the request is
> valid, you get a 200 with per-event failures listed. Reconcile on `status` and
> the counts, not on the HTTP status code alone.

> **`errors.length` isn't the rejected count.** Errors are per field, so one
> malformed event can produce several entries. Use `index` to map each error back
> to the event you sent — it's present even when `event_id` was missing.

For large batches, `?results=errors` returns only the failures and
`?results=none` omits the array entirely.

---

## Errors and retries

| Status | Meaning | What to do |
|---|---|---|
| `200` | Accepted, fully or partially | Check `status`, `accepted`, `rejected` |
| `400` | Validation failure; no event accepted | Fix and resend. Don't retry unchanged. |
| `401` | Authentication failure | Check your token. Don't retry unchanged. |
| `403` | Token lacks the required scope | Contact us |
| `413` | Over 1,000 events or over 5 MB | Split the batch and resend |
| `429` | Rate limited | Wait for `Retry-After`, then retry |
| `5xx` | Server-side failure | Retry with exponential backoff |

Recommended retry policy: exponential backoff with jitter, starting around one
second, up to five attempts. **Retrying is safe** — see
[Idempotency](#idempotency).

`429` and `503` responses include a **`Retry-After`** header in seconds, and the
same value as `retry_after` in the body. Please honour it rather than using a
fixed interval.

---

## Rate limits

1,000 requests per minute per product. Exceeding it returns `429` with
`Retry-After`.

Requests continue to be rate limited for a short period after a violation, so
back off rather than retrying immediately. If your expected volume needs a higher
limit, tell us during onboarding — batching up to 1,000 events per request is
usually the better answer.

---

## Idempotency

`event_id` is the idempotency key. You can retry safely.

- The deduplication key is **`(product_id, event_id)`**, so your identifiers only
  need to be unique within your own data.
- **A retry must carry the same `event_id`.** Two different events must never
  share one.
- **Duplicates within one request are rejected.** If the same `event_id` appears
  twice in one array, the first is accepted and the second rejected with an error
  on `event_id`. We treat this as a client-side mistake rather than a retry.
- **Duplicates across requests are accepted** and deduplicated downstream, so a
  retry never fails.
- The deduplication window isn't time-limited — a duplicate arriving weeks later
  is still recognised.

The guarantee is **at-least-once acceptance, exactly-once in reported measures**.

---

## Testing

**Dry run.** `POST /v1?dry_run=true` validates your payload in full and returns
the normal response without storing anything. Ideal for checking field mapping
before you send live data.

**Test school.** `school_id = sch_test_000` — events are accepted and stored but
excluded from all analytics. Safe to use throughout development.

**Sandbox validation.** During onboarding we'll share a mapping report showing how
each of your event names and fields becomes a measure in our analytics, for you to
confirm before go-live.

---

## Data protection requirements

These are contractual, not just conventions.

**Identifiers must be pseudonymous.** `actor_id`, `learner.id` and
`activity.assigned_by` must not be, or be derived from, personal data — no emails,
names, UPNs or phone numbers. A salted hash or an internal surrogate key is fine;
an unsalted hash of an email is not, because it's reversible by dictionary attack.

**Identifiers must be stable and consistent.** The same person must carry the same
identifier across events and over time, and `learner.id` must come from the same
pseudonym space as `actor_id`.

**Strip URLs.** Remove query parameters and fragments from `context.page_url` —
they routinely carry names, tokens and email addresses.

**Watch free text.** `result.response` carries whatever the learner typed. Only
send it where your agreement with us covers it, and redact before sending if there
is any chance of personal data appearing.

**`raw` is stored verbatim.** Whatever you put in it, we keep. Make sure it
contains no personal data you wouldn't send in the envelope.

**Retractions are not erasure.** `voided` removes an event from analytics.
Data-subject erasure requests are handled out of band — see below.

## Retention and erasure

| | Period |
|---|---|
| Structured event data | **13 months** from ingest, then removed |
| `raw` payloads | **90 days** from ingest, then removed; the structured envelope is retained |
| Erasure on request | Removed from analytics immediately, from underlying storage **within 30 days** |

Two consequences worth designing for:

- **`raw` is not permanent.** It is retained long enough for us to reprocess and
  enrich, then removed. If a field in `raw` matters to your analytics, promote it
  into the envelope rather than relying on us extracting it later.
- **Send only what you need to.** We keep whatever you put in `raw` for those 90
  days. If your payloads contain personal data that the envelope rules would
  prohibit, either redact before sending or tell us and we can omit `raw` capture
  for your integration entirely.

### Requesting erasure

For a participant withdrawal or a data-subject erasure request, contact us with:

- the affected **pseudonymous identifiers** (`actor_id`, `learner.id` and/or
  `event_id`)
- your `product_id` and the relevant `school_id`

**Please do not include names or email addresses.** That would introduce into our
systems the personal data the request exists to remove.

We remove all matching records from our analytical dataset — and therefore from
every report, benchmark and dashboard — and confirm the number of records removed.
Removal from underlying storage completes within 30 days.

---

## Troubleshooting

**`event_id` "is required"** — `event_id` is mandatory. Earlier versions of this
documentation omitted it from the required list; it has always been validated.

**`occurred_at` "must be an ISO-8601 timestamp with a UTC offset"** — add `Z` or an
offset. `2026-03-01T10:22:00` is rejected; `2026-03-01T10:22:00Z` is accepted.

**`attributes` "keys must be snake_case"** — `masteryLevel` is rejected, use
`mastery_level`.

**`product_id` "token is scoped to …"** — the `product_id` in the event doesn't
match your token. Check you're not sending staging data with a production token.

**`schema_version` "must be 1.2 when using learner, revision, voided"** — bump
`schema_version` to `"1.2"` to use those fields.

**`learner_id` "is not a field — did you mean learner.id?"** — the learner
identifier belongs inside the `learner` object.

**`learner` "must include at least one of id, class_id or cohort_id"** — a
`learner` object has to identify a person or a group.

**`actor.role` "is required when learner.id differs from actor_id"** — tell us
whether the actor was a teacher, admin or system.

**`revision` "corrections and voids are not yet supported"** — contact us to have
corrections enabled for your integration.

**Everything returns 200 but nothing appears in reports** — check you aren't
sending `school_id: "sch_test_000"`, which is excluded from analytics by design.

---

## Enum reference

| Field | Values |
|---|---|
| `schema_version` | `1.0`, `1.1`, `1.2` |
| `event_kind` | `interaction`, `state_change`, `system` |
| `actor.role`, `learner.role` | `student`, `teacher`, `admin`, `system` |
| `actor.education_level`, `learner.education_level` | `early-childhood`, `primary`, `lower-secondary`, `upper-secondary` |
| `activity.difficulty` | `beginner`, `intermediate`, `advanced` |
| `client.platform` | `web`, `ios`, `android`, `desktop` |
| `client.device_type` | `mobile`, `tablet`, `desktop` |
| `status` (response) | `accepted`, `partial`, `rejected` |

`activity.type` and `activity.subject` are **not** enums — send your own values and
we map them.

---

## Limits reference

| Limit | Value |
|---|---|
| Events per request | 1,000 |
| Payload size | 5 MB |
| Requests per minute, per product | 1,000 |
| `attributes` key length | 64 characters |
| `event_source` length | 64 characters |
| `grade_index` | 0–20 |
| `score`, `progress` | 0.0–1.0 |
| `attempt` | 1 or greater |
| `duration_ms`, `event_sequence`, `revision` | 0 or greater |
| `occurred_at` future tolerance | 48 hours |
| `occurred_at` earliest | 2000-01-01 |

---

## Changelog

### 1.2

Additive — `1.1` payloads remain valid and are processed unchanged.

**Added**

- `learner` object: the learner or group an event concerns, with their
  educational context. Supports individual and class-level events.
- `revision` and `voided` for corrections and retractions (enabled per
  integration).
- `results[]` and `status` in responses, so accepted and rejected events are
  identified individually.
- `index` on each error, so a rejection maps back to your submitted array even
  when `event_id` was missing.
- `Retry-After` on `429` and `503`.
- `?results=` and `?dry_run=` query parameters.

**Clarified in documentation** (behaviour unchanged)

- `event_id` is required. It was missing from the v1.1 required-fields table.
- `event_source` is free text, not a fixed enum.

**Now validated** (previously accepted loosely)

- `occurred_at` and `sent_at` must include a UTC offset.
- `actor.country` must be a two-letter alpha-2 code.
- `eti_school_id` accepts a string or an integer.

**Note on `1.0`:** still accepted by the API, but not currently included in
analytics. If you're sending `1.0`, talk to us.

**Retention published.** Structured events 13 months, `raw` payloads 90 days,
erasure on request within 30 days. See [Retention and erasure](#retention-and-erasure).

---

## Support

Quote the `request_id` from any response when raising a query. For access,
`product_id` issuance, higher rate limits, corrections, erasure requests or
anything covering personal data, contact your EdTech Impact onboarding contact.
