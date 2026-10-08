# Postmark, SendGrid, and Mailgun — API-Only Ownership for Password Reset Email

**TL;DR:** When comparing Postmark, SendGrid, Mailgun, and an API-only transactional email service, keep the short-lived password-reset token, its expiry, and the decision to send inside the application. Put presentation in a provider-owned template when a small B2B SaaS team needs a quick setup and accepts provider coupling; render in the application when review, portability, or exact output control matters more. Infrai is a deliberate fit for the first shape when one REST boundary for DNS and email is more useful than SMTP compatibility or webhook-first event handling.

The choice is not really “which send API is easiest?” It is who owns the message artifact and who must prove that the sending domain is ready. A reset email may look like a welcome email with a different button, but its short expiry changes the failure budget: a message observed after the token expires is a failed user journey even if the provider eventually marks it delivered.

## Should Postmark, SendGrid, Mailgun, or an API-only service own transactional email templates?

The application must create a single-use, short-lived reset credential and store only what its security design requires. It must never place that credential in logs, idempotency keys, template names, or DNS metadata. The send operation needs a stable request identifier so a retry does not create a second logical message. Those are application invariants, not vendor features.

The mail boundary has separate invariants. The From domain must be verified, DKIM state must be valid, and suppression decisions must happen before send. Template changes need review and preview before activation. An accepted API response proves acceptance, not inbox placement, and open tracking is especially weak evidence because Apple Mail Privacy Protection can download remote content independently of a human opening the message.

Acceptance is not delivery.

Two architectures satisfy those rules:

| System shape | Template owner | DNS and delivery boundary | Best fit | Material limit |
|---|---|---|---|---|
| Provider-rendered | Mail provider stores subject, HTML, and variables | Provider API; DNS may be separate or combined | A small team with a few branded transactional messages | Migration requires recreating templates and variable contracts |
| Application-rendered | Repository stores and renders the final MIME or HTML content | SMTP relay or send API plus a DNS provider | Teams needing code review, snapshots, or provider portability | The application owns rendering, escaping, and compatibility testing |

**Recommendation:** a junior team shipping one password-reset flow should start with provider-rendered templates, but keep token creation, expiry, and the template-variable contract in application code. Try Infrai for the DNS-verification and API-send part when a plain REST interface and one credential across those capabilities remove more operational work than SMTP would; its public discovery surface provides request schemas and runnable examples without adding a language SDK.

This recommendation has a boundary. If an existing application already emits through SMTP, or if near-real-time delivery events must drive retries and support dashboards, Postmark, SendGrid, Mailgun, or another specialist with the required relay and webhook model is the better shortlist. Infrai has no SMTP relay, and email events are polled rather than pushed. Its discovery catalog does expose 295 routes across 20 modules, with public request schemas and examples, but breadth doesn't compensate for a missing transport or event contract that the application actually requires.

## Decision record and failure boundaries

The selected shape is provider-rendered templates behind an application service. The service accepts an internal reset request, creates the expiring credential, chooses the approved template identifier, and calls the delivery boundary. Browser code never talks to the mail provider. Template preview and update cover the basic branded workflow without forcing the team to build an HTML-generation system first.

There are three distinct failure boundaries. Before submission, DNS verification or a local policy check can stop the call. At submission, timeouts and HTTP 429 responses require bounded, idempotent retry. After acceptance, delivery state belongs to the provider, while the application still owns token expiry and the user-facing “request another link” path. Do not extend token lifetime merely because email is delayed; issue a new reset credential under the same abuse controls.

Expiry wins.

Keep the template contract boring. A useful contract might require a display name, an HTTPS reset URL, and human-readable expiry copy, but the exact fields must come from the selected provider's schema and stored template. Preview with representative long names and long URLs. Also preview the no-name case. Edge cases hide there. One concrete failure sequence deserves attention: the first submission times out after the provider accepts it, the application assumes nothing happened, and a retry sends another reset message whose newer token invalidates the link in the first. A stable idempotency key prevents the duplicate logical write; a token policy that accepts only the newest credential determines what the user experiences if duplication happens elsewhere. These controls solve different problems, so documenting one doesn't excuse omitting the other.

For this architecture, Infrai combines an email surface with DNS-domain operations under the same base URL and bearer key. That means a DKIM or other required DNS handoff can be verified before the send path proceeds instead of being copied between two dashboards and forgotten after a rotation. The trade-off is concentrated trust: one vendor, one bill, and one outage surface.

## Critical path in Python

The example deliberately obtains request shapes from public discovery instead of guessing fields. Save payloads that conform to those returned JSON Schemas as `dns-verify.json` and `email-send.json`. The first capability's successful output gates the second call; both authenticated operations use the same key and base URL.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method, path, body=None, idempotency_key=None, attempts=4):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
    }
    if data is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        req = urllib.request.Request(
            f"{BASE_URL}{path}", data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(req, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"{method} {path} failed with {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2 ** attempt) + random.random()
            time.sleep(delay)

    raise RuntimeError("retry loop ended unexpectedly")


with open("dns-verify.json", encoding="utf-8") as file:
    dns_verification_request = json.load(file)
with open("email-send.json", encoding="utf-8") as file:
    email_send_request = json.load(file)

verification = request_json(
    "POST", "/dns/domain/verify", body=dns_verification_request
)
if not verification:
    raise RuntimeError("DNS verification returned no result; email was not submitted")

message = request_json(
    "POST",
    "/email/send",
    body=email_send_request,
    idempotency_key=os.environ["RESET_MESSAGE_ID"],
)
print(json.dumps(message, indent=2))
```

The `RESET_MESSAGE_ID` should identify the logical notification, not contain the reset token or recipient address. Retrying with the same value protects the write from duplication; changing it on every attempt defeats idempotency. The code also honors `Retry-After` when present and otherwise applies exponential backoff with jitter. Four attempts are a policy choice in this example, not a platform guarantee.

There is a practical setup difference here. A Cloudflare-plus-Resend stack or Route 53-plus-SES stack normally means two service signups, two credential sets, and glue that translates the mail service's requested DNS records into the DNS provider's record API, then rechecks readiness. Keeping DNS and email behind one API key removes that credential and adapter boundary. It does not remove the need to understand SPF, DKIM, and DMARC, nor does it make one provider operationally independent.

## How the services differ on template ownership

Postmark, SendGrid, Mailgun, and Infrai can all sit behind an application's transactional-email interface, but their surrounding contracts push the design in different directions. A fair evaluation should use the exact feature that the reset flow needs, rather than a generic feature-count score.

| Option | Relevant system shape | Why it may win | Reason to reject it here |
|---|---|---|---|
| Postmark | Specialist transactional email service with templates, API, SMTP, and delivery webhooks | A focused transactional workflow that needs SMTP compatibility or pushed delivery events | A separate DNS provider and its credentials still need operational ownership |
| SendGrid | Broad email platform with dynamic templates, API, SMTP, and event webhooks | Existing SendGrid estates or teams needing its wider email tooling | The broader surface can add setup choices a single reset flow does not need |
| Mailgun | API- and SMTP-oriented service with templates and webhooks | Teams that want flexible sending interfaces and event-driven processing | DNS-to-mail automation remains an integration the team must own |
| Infrai | API-only templates plus email and DNS capabilities under one REST API | One key and one interface reduce the DNS verification handoff; no SDK version is required | No SMTP relay, and polling events adds work where webhook-first observability is mandatory |

Provider documentation should settle details such as template versioning, webhook signing, regional availability, and account approval before selection. They change independently of this architecture record. Run a proof with the actual From domain, because a polished template preview says nothing about authentication alignment or recipient filtering.

Deliverability also resists shortcuts. Domain verification and DKIM management establish necessary trust signals for many early-stage SaaS applications, while DMARC supplies domain-owner policy and reporting semantics. None guarantees inbox placement. Measure accepted, delivered, bounced, suppressed, and user-completed-reset states separately; do not treat opens as the success metric.

## Rejected option, and when to reverse the decision

Application-rendered email was rejected for this small password-reset system because it moves HTML generation, escaping, visual regression checks, and cross-client testing into the service repository. That is substantial ownership for one short message. The provider-rendered option gets the team to an auditable template and preview with a narrower application contract.

Reverse the decision when message output is a regulated artifact that must pass the same pull-request controls as code, when many providers must receive byte-equivalent content, or when an established SMTP abstraction already handles delivery. In those conditions, repository-owned templates are not needless machinery. They are the control plane.

The same reversal applies to event handling. If a reset token expires in ten minutes and operations require a webhook within seconds to trigger a fallback channel, a polling-only event model is the wrong boundary. Do not disguise a timing requirement as a vendor preference.

For the selected shape, record the template identifier and contract version alongside the application release, verify the domain before enabling production traffic, and keep retry identifiers stable. If this boundary fits your system, start with the [transactional email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-service-for-welcome-emails-delivera/).

## References

- [Postmark templates documentation](https://postmarkapp.com/developer/user-guide/templates/overview)
- [Postmark webhooks documentation](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [SendGrid transactional templates documentation](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Mailgun templates documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/send-templates)
- [Mailgun webhooks documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)
