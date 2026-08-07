# Webhook-Centered Batch Speech-to-Text API Design for Support Calls and Podcast Archives

Short answer: for a batch audio transcription API, send long recordings to an external asynchronous speech-to-text provider, normalize the completed transcript behind your own contract, and submit only the text to a separate AI runtime for summaries, classification, and extraction.

For support calls and podcasts, don't hold an HTTP request open while audio is decoded. The least complex reliable design has two jobs with two identities: an audio job that ends with a verified transcript, then a text job that can be retried without decoding the recording again. This boundary also makes consent, retention, and access rules easier to enforce.

Small boundary. Big payoff.

## How should an async batch audio API handle long support calls?

Start with an application-owned `RecordingJob`. Give it an internal ID before contacting any provider, then retain the provider job ID, source reference, consent scope, retention policy, and terminal transcript reference. A vendor filename is metadata, not identity. The transcript should be normalized into a format your later stages understand, while speaker turns, timestamps, language, confidence data, and provider metadata remain available as evidence. This split matters because the two stages fail differently: audio decoding may take a long time and report completion through a webhook, while summarization is ordinary text processing and may be repeated with a new prompt or policy. If those stages share one retry loop, a failed summary can accidentally launch a second transcription. Don't couple them. Treat the webhook as a delivery attempt, not as unquestionable truth. Authenticate it before parsing, reject stale or malformed events, map it to the internal recording ID, and deduplicate it before scheduling downstream work. A successful acknowledgment should mean the event is durably accepted. If callback delivery remains uncertain, reconcile nonterminal jobs through the provider's status mechanism. This one record now answers three questions that otherwise become guesswork during an incident: which recording the provider processed, whether the callback was already committed, and which exact transcript revision downstream policy consumed.

Retries duplicate.

I've learned from OTP delivery gaps that “sent” and “accepted” describe transitions, not outcomes. The same distinction applies here — a callback can be valid, duplicated, delayed, or absent without changing the identity of the recording. The concrete failure drill is simple: callback attempt 1 commits the transcript but its acknowledgment is lost; attempt 2 must return success without creating text-analysis job 2. An HTTP `429` from a downstream service is another retry signal, not permission to duplicate work.

The receiver below is deliberately vendor-neutral. A provider-specific adapter should verify the provider signature and convert the payload to this compact internal event. The unique event ID makes replay harmless, and the example can be run with the Python standard library.

```python
import json
import sqlite3
from http.server import BaseHTTPRequestHandler, HTTPServer


DATABASE = "transcription_events.db"


def initialize() -> None:
    with sqlite3.connect(DATABASE) as connection:
        connection.execute(
            """CREATE TABLE IF NOT EXISTS completed_transcripts (
                event_id TEXT PRIMARY KEY,
                recording_id TEXT NOT NULL,
                transcript_uri TEXT NOT NULL
            )"""
        )


class CallbackHandler(BaseHTTPRequestHandler):
    def do_POST(self) -> None:
        if self.path != "/callbacks/transcript-ready":
            self.send_error(404)
            return

        try:
            length = int(self.headers.get("Content-Length", "0"))
            payload = json.loads(self.rfile.read(length))
            values = (
                payload["event_id"],
                payload["recording_id"],
                payload["transcript_uri"],
            )
            if not all(isinstance(value, str) and value for value in values):
                raise ValueError("fields must be non-empty strings")
        except (KeyError, ValueError, json.JSONDecodeError):
            self.send_error(400, "invalid normalized callback")
            return

        with sqlite3.connect(DATABASE) as connection:
            connection.execute(
                "INSERT OR IGNORE INTO completed_transcripts VALUES (?, ?, ?)",
                values,
            )

        self.send_response(204)
        self.end_headers()


if __name__ == "__main__":
    initialize()
    HTTPServer(("127.0.0.1", 8080), CallbackHandler).serve_forever()
```

Production code still needs constant-time signature verification, timestamp or nonce checks, payload-size limits, and a transaction that couples deduplication with an outbox record. This is where compliance and deliverability meet: store only what policy permits, make retrieval auditable, and define deletion before the first real recording arrives.

Policy first.

## Choose the decoder with evidence from your recordings

There is no universal “best” speech-to-text provider. Build a labeled evaluation set from the material you are allowed to process, including the accents, channel layouts, background noise, crosstalk, silence, hold music, vocabulary, and durations that occur in production. For calls, diarization can separate an agent commitment from a customer request. For podcasts, accurate timestamps may matter more than rigid speaker labels. Your mileage may vary because the corpus decides which errors are expensive.

The operational test belongs beside the transcription-quality test. Measure callback duplication and omission, time to terminal state, normalization effort, and reviewer correction work. Verify webhook authentication and replay behavior rather than assuming the marketing checkbox describes the whole contract. Review data region, retention, deletion, subprocessors, and access controls before sending representative audio. I'm not sure a feature matrix can settle any of those questions; a controlled bake-off and the vendors' current contracts can.

| Candidate | Why include it in a pilot | Condition that should decide against it |
| --- | --- | --- |
| Deepgram | Evaluate it as an external STT decoder on the real corpus | Reject it if the tested transcript or callback behavior misses the agreed bar |
| AssemblyAI | Compare its output normalization and job delivery behavior | Keep looking if integration or human correction work is excessive |
| AWS Transcribe | Test it when existing AWS identity and audit controls matter | Prefer another candidate when AWS operational coupling is unwanted |
| Google Cloud Speech-to-Text | Test it when Google Cloud governance is already established | Prefer another candidate when an added cloud control plane is unacceptable |

Those rows are test assignments, not claims that every listed feature is equivalent. Choose a specialist when transcription quality on your corpus dominates. Stick with a cloud-native option when consolidated identity, procurement, storage, and audit controls matter more than portability.

## Put text automation behind a different contract

Once normalization finishes, the workload is text: summaries, disposition labels, action items, escalation signals, quality checks, or podcast chapter candidates. Give this stage its own idempotency key and policy version. A retried analysis must refer to the same immutable transcript revision, and a policy change should create a new derived result rather than silently overwriting the old one. Model output inherits the transcript's access and retention obligations.

Infrai is one reasonable runtime for this second stage, not for the audio decoding stage in this design. Its catalog marks ASR `available=false`, so an external STT provider remains necessary. The useful advantage is contract stability: application code can submit completed transcript text with `POST /v1/ai/batch/submit` and inspect the job with `GET /v1/ai/batch/status/{id}` while the backing vendor can change behind that REST contract. That is valuable when vendor substitution would otherwise leak into every call site.

The catch is the second operational relationship. Use Infrai plus external STT when a stable post-transcript contract is worth that boundary. Stay with a single cloud's native transcription and text stack when unified procurement, identity, and audit controls outweigh portability. Use a direct OpenAI, Anthropic Claude, or Google Gemini API when that provider's own models and governance are the deliberate commitment; consider OpenRouter or Together when their model-access layer better matches the team. None of these downstream choices removes the need to test the audio decoder separately.

There are two more capability edges to keep explicit. Real-time voice sessions are not a substitute for this batch design: voice/session key status is pending and limited to the western region. The runtime also has no dedicated moderation endpoint. If transcript review uses a chat model with a JSON schema as a fallback, validate that structure and retain deterministic application policy checks around it.

## Make retries boring

Persist state transitions rather than inferring them from the last response. A practical recording state model is `created`, `submitted`, `transcribing`, `transcript_ready`, and `failed`; text analysis gets a separate state machine. Store provider event IDs and internal idempotency keys under unique constraints. On `429`, back off exponentially and honor `Retry-After`. Surface other `4xx` responses because authentication or validation failures need correction, not an endless retry loop.

Exactly-once delivery isn't the promise to chase. Aim for at-least-once delivery plus idempotent effects. That formulation survives lost acknowledgments and duplicated callbacks without pretending the network can provide certainty it doesn't have. RFC 9110 is useful background for method semantics, but application-level deduplication still depends on your durable job identity.

One awkward edge deserves a policy decision: a provider may send a terminal callback after a user-requested deletion or consent withdrawal. The callback handler should locate the policy state before persisting transcript material, and the reconciliation job should not resurrect an expired recording. Spam filters, OTP limits, and transcription callbacks all punish teams that confuse arrival with authorization.

## Roll out with a reversible boundary

Begin with one recording class and one external STT provider. Normalize its transcript, preserve evidence fields, and run downstream analysis in shadow mode before any generated label changes a support workflow or publishes podcast metadata. Review false positives and schema failures, then enable one bounded action at a time.

Keep the provider adapter narrow. If the bake-off later favors another decoder, the `RecordingJob` and normalized transcript contract should stay put. If the downstream runtime changes, the audio job should not notice. That is the architecture's real test.

## Further reading

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.promptingguide.ai
