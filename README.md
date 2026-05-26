# Anteroom Corpus Schema

This repository documents the data model and architectural discipline
behind the Anteroom regulatory corpus.

The Anteroom application code lives in a private repository. This public
repo exists because the *shape* of the corpus, the integrity rules
around it, and the change-tracking model are the parts most worth
making transparent. Reviewers, contributors, and serious readers can
understand how Anteroom thinks about regulatory text without needing
access to the application itself.

## What this repo contains

- The corpus data model (table definitions, key fields, relationships)
- The reviewer attribution model
- The stable provision ID system
- The change-tracking and version history model
- The signal-to-corpus mapping model

## What this repo does not contain

- The corpus content itself (provisions, summaries, syntheses)
- The classifier rules that determine which provisions apply to a launch
- The synthesis prompts used at analysis time
- The regulatory signals pipeline
- Anything operational that would expose Anteroom subscribers or paid IP

## Core principles

Anteroom treats regulatory text as a *living corpus*, not a one-time
memo. Three principles run through the schema:

1. **Source integrity.** Every provision points back to a primary
   source URL. No secondary commentary masquerades as primary text.

2. **Verification discipline.** Every provision carries a named
   reviewer and a last-verified date. Cites without those fields do
   not ship.

3. **Change awareness.** Every provision has a stable identifier
   that survives renumbering, restructuring, and amendment. When
   the text changes, the identifier does not. Subscribers can be
   notified about the same provision over time even when the
   underlying document is reorganized.

## Tables (simplified)

### `corpus_provision`

The canonical record of a single regulatory obligation.

| Field | Type | Description |
|---|---|---|
| `stable_provision_id` | string | Permanent identifier that survives source-document restructuring |
| `corpus_version` | integer | Monotonically increasing version counter for the corpus |
| `source_url` | string | Canonical link to the primary source |
| `source_jurisdiction` | string | EU, US-CA, US-CO, US-NY, US-IL, UK, etc. |
| `source_regime` | string | e.g. EU AI Act, NIST AI RMF, GDPR, CCPA, BIPA, ADA |
| `source_citation` | string | The citation form a lawyer would actually use |
| `provision_text` | text | The current canonical text of the obligation |
| `plain_summary` | text | Plain-English summary, reviewer-written |
| `reviewer_name` | string | Named reviewer who verified the provision |
| `last_verified_at` | timestamp | When the reviewer last confirmed accuracy |
| `applies_to_lenses` | string[] | Which use-case lenses this provision can be triggered by |

### `corpus_provision_history`

Append-only record of how a provision has changed over time. Drives the
diff that subscribers see when a provision moves.

| Field | Type | Description |
|---|---|---|
| `stable_provision_id` | string | Same identifier as above |
| `from_version` | integer | Corpus version before the change |
| `to_version` | integer | Corpus version after the change |
| `from_text` | text | Provision text before |
| `to_text` | text | Provision text after |
| `change_summary` | text | Plain-language description of what changed and what to do about it |
| `changed_at` | timestamp | When the change was recorded |

### `signal_event`

A single regulatory or enforcement signal pulled from a primary-source
feed (regulator RSS, regulator HTML, court filing, agency release).
Each signal is classified and mapped to one or more provisions.

| Field | Type | Description |
|---|---|---|
| `signal_id` | string | Unique identifier for the event |
| `source` | string | Originating feed (EDPB, FTC, NIST, FDA, ICO, CNIL, etc.) |
| `source_url` | string | Link to the primary item |
| `title` | string | Title of the source item |
| `summary` | text | One-paragraph summary |
| `classified_at` | timestamp | When the classifier processed the signal |
| `mapped_provision_ids` | string[] | Which corpus provisions this signal touches |

### `subscription`

A user's saved analysis being watched for drift. Notifications are
computed against the corpus version saved here.

| Field | Type | Description |
|---|---|---|
| `subscription_id` | string | Unique identifier |
| `email` | string | Subscriber email |
| `saved_at` | timestamp | When the analysis was first saved |
| `saved_corpus_version` | integer | Corpus version at save time |
| `watched_provision_ids` | string[] | Which provisions this subscription monitors |

### `notification_event`

Append-only record of which subscribers were notified about which
provision changes. Prevents double-sending and lets the diff cursor
advance correctly when a subscription has already been notified at a
later version than its original save.

| Field | Type | Description |
|---|---|---|
| `subscription_id` | string | Foreign key to subscription |
| `stable_provision_id` | string | Foreign key to provision |
| `to_version` | integer | Corpus version notified about |
| `sent_at` | timestamp | When the email went out |

A `UNIQUE (subscription_id, stable_provision_id, to_version)` constraint
guarantees each subscriber sees each change to each provision exactly
once.

## License

This documentation is published under MIT. The Anteroom application
itself is private.

## Contact

[anteroom.so](https://anteroom.so) — hello@anteroom.so
