# EdTech Impact Analytics API

**Canonical Event Contract v1.1**

------------------------------------------------------------------------

## Authentication

Include your API token in the Authorization header.

    Authorization: Bearer your-api-token

Tokens are issued during onboarding and scoped to your `product_id`.

------------------------------------------------------------------------

# Endpoint

    POST https://im.pact.workers.dev/v1
    Content-Type: application/json
    Authorization: Bearer your-api-token

This endpoint accepts event ingestion for both real-time and batch
events.

------------------------------------------------------------------------

# Request Format

Requests must contain an **array of events**, even if sending a single
event.

Maximum **1,000 events per request (5MB payload limit).**

------------------------------------------------------------------------

# Minimum Required Payload

``` json
[
  {
    "schema_version": "1.1",
    "product_id": "study-app",
    "school_id": "sch_test_000",
    "event_name": "session_started",
    "occurred_at": "2026-03-01T10:00:00Z",
    "actor_id": "usr_test_001"
  }
]
```

# Minimum Payload with Raw Event

Include your original event in the `raw` field alongside the required
envelope fields. Your entire original event is preserved untouched ---
EdTech Impact standardises and analyses your data without modifying it.

``` json
[
  {
    "schema_version": "1.1",
    "product_id": "study-app",
    "school_id": "sch_alkhor_001",
    "event_name": "entered_study_mode",
    "occurred_at": "2026-02-20T08:31:00Z",
    "actor_id": "usr_anon_abc123",
    "raw": {
      ...your original event eg:
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

------------------------------------------------------------------------

# Full Payload Example

Example event containing **all domains and fields** supported by the
contract.

``` json
[
  {
    "schema_version": "1.1",
    "product_id": "study-app",
    "school_id": "sch_alkhor_001",
    "eti_school_id": "eti_34821",
    "event_name": "question_answered",
    "event_source": "sdk",
    "event_kind": "interaction",
    "occurred_at": "2026-03-01T10:22:00Z",
    "sent_at": "2026-03-01T10:22:01Z",
    "event_sequence": 3,
    "actor_id": "usr_anon_abc123",

    "actor": {
      "role": "student",
      "session_id": "sess_019d1a2b0001",
      "country": "GB",
      "education_level": "lower-secondary",
      "grade_index": 7,
      "class_id": "7B",
      "cohort_id": "pupil_premium",
      "institution_id": "mat_102",
      "national": {
        "yearGroup": "year-7",
        "keyStage": "ks3"
      }
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
      "assigned_by": "tch_992",
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

    "raw": {
      ...your orginal event eg:
      "event_name": "question_answered",
      "occurred_at": "2026-03-01T10:22:00.000000+00:00",
      "payload": {
        "question_id": "q42",
        "response": "3/4",
        "correct": true
      }
    }
  }
]
```

------------------------------------------------------------------------

# Required Fields

  ------------------------------------------------------------------------
  Field             Type              Description
  ----------------- ----------------- ------------------------------------
  schema_version    string            Contract version of the payload.
                                      Current version: `"1.1"`.

  product_id        string            Your EdTech Impact application
                                      identifier.

  school_id         string            Your school identifier.

  event_name        string            Event name from the canonical
                                      taxonomy or your own event names.

  occurred_at       timestamp         ISO-8601 timestamp when the event
                                      occurred.

  actor_id          string            Must be a pseudonymous identifier.
                                      Must not contain or be derived from
                                      personally identifiable information
                                      (emails, names, UPNs, phone
                                      numbers).
  ------------------------------------------------------------------------

------------------------------------------------------------------------

# Optional Envelope Fields

  ------------------------------------------------------------------------
  Field             Type              Description
  ----------------- ----------------- ------------------------------------
  eti_school_id     string            EdTech Impact school identifier if
                                      known. Skips server-side resolution.

  event_source      string            Indicates how the event entered the
                                      ingestion system. Useful for
                                      debugging and pipeline attribution.

  event_kind        string            interaction, state_change, or system
                                      classification of the event.

  sent_at           timestamp         ISO-8601 timestamp indicating when
                                      the client system sent the event.

  event_sequence    integer           Ordering index within a session when
                                      timestamps lack precision.

  product_version   string            Version of your application at the
                                      time the event was sent.

  raw               object            Your original event payload,
                                      preserved untouched. EdTech Impact
                                      maps and enriches from this data
                                      server-side. No fidelity is lost.
  ------------------------------------------------------------------------

------------------------------------------------------------------------

# Actor Domain

Describes the user and educational profile.

  Field             Type      Description
  ----------------- --------- ----------------------------------------
  role              string    student, teacher, admin, system
  session_id        string    Session identifier
  country           string    ISO-3166 country code
  education_level   string    ISCED level
  grade_index       integer   Years of schooling
  class_id          string    Class identifier
  cohort_id         string    Intervention or research cohort
  institution_id    string    MAT, district, or authority identifier
  national          object    Country-specific educational metadata

------------------------------------------------------------------------

# Activity Domain

  ----------------------------------------------------------------------------
  Field                 Type              Description
  --------------------- ----------------- ------------------------------------
  id                    string            Your content identifier. Required
                                          for most analytics and strongly
                                          recommended.

  name                  string            Human readable name

  type                  string            lesson, quiz, question, video,
                                          assessment, chat

  subject               string            Curriculum subject (defined in enum
                                          registry)

  topic                 string            Free text topic

  difficulty            string            beginner, intermediate, advanced

  ai                    boolean           Indicates AI involvement

  attempt               integer           Attempt number

  parent_id             string            Parent activity identifier

  assigned_by           string            Teacher identifier

  curriculum_standard   string            Curriculum reference
  ----------------------------------------------------------------------------

------------------------------------------------------------------------

# Result Domain

  ------------------------------------------------------------------------
  Field             Type              Description
  ----------------- ----------------- ------------------------------------
  success           boolean           Indicates successful completion of a
                                      non-assessment action

  correct           boolean           Indicates whether the response was
                                      correct

  score             number            Decimal score 0.0--1.0

  response          string            Student response

  duration_ms       integer           Time spent

  progress          number            Progress decimal 0.0--1.0
  ------------------------------------------------------------------------

### Correct vs Success

Use `correct` for assessment events such as `question_answered`.

Use `success` for non-assessment outcomes such as:

-   activity_completed
-   lesson_completed
-   video_completed

Events should not normally include both fields.

------------------------------------------------------------------------

# Client Domain

  Field         Type     Description
  ------------- -------- ------------------------------
  platform      string   web, ios, android, desktop
  device_type   string   mobile, tablet, desktop
  user_agent    string   Browser or device user agent
  app_version   string   Application version

------------------------------------------------------------------------

# Context Domain

  -------------------------------------------------------------------------
  Field              Type              Description
  ------------------ ----------------- ------------------------------------
  course_id          string            Course identifier

  assignment_id      string            Assignment identifier

  learning_path_id   string            Learning pathway identifier

  page_url           string            URL where event occurred. Query
                                       parameters and fragments should be
                                       removed to prevent accidental
                                       transmission of personal data.

  page_title         string            Page title
  -------------------------------------------------------------------------

------------------------------------------------------------------------

# Attributes

Flexible metadata container.

Rules:

-   Keys must be **snake_case**
-   Keys must be **\< 64 characters**
-   Must not duplicate structured fields like subject, score, session_id

Example:

``` json
"attributes": {
  "mastery_level": "emerald",
  "feature": "study_mode"
}
```

------------------------------------------------------------------------


# Response Format

### Success

``` json
{
  "request_id": "req_91df2c",
  "accepted": 2,
  "rejected": 0,
  "errors": []
}
```

### Partial Failure

``` json
{
  "request_id": "req_91df2c",
  "accepted": 1,
  "rejected": 1,
  "errors": [
    {
      "event_id": "018f9b34-3b6f-7a60-9a3e-9d6a2b2a4e6c",
      "field": "school_id",
      "message": "required field missing"
    }
  ]
}
```

### Authentication Error

``` json
{
  "request_id": "req_91df2c",
  "error": "Invalid or expired API token"
}
```

------------------------------------------------------------------------

# Error Handling

  Status   Action
  -------- --------------------------------
  200      Success
  400      Validation error
  401      Authentication error
  429      Rate limited
  5xx      Retry with exponential backoff

------------------------------------------------------------------------

# Testing

Use the test school during development.

    school_id = sch_test_000

Events sent with this school ID are accepted but excluded from
analytics.
