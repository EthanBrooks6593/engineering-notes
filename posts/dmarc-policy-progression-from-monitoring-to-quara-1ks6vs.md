# DMARC Policy Progression from Monitoring to Quarantine for Customer Owned Domains

Every enforcement rollout on a customer's domain is a race between two clocks, and optimizing the wrong one is how these projects stall for a quarter. The first clock is DNS: the TXT record you replace stays cached for whatever TTL the old one carried, so no policy change is truly instant. The second is the aggregate report loop, which by default is a full day wide. Cutover speed looks like a propagation problem and almost never is, so go with the report clock — publish `p=none` first, and treat each move toward quarantine as something the evidence buys you rather than something the calendar grants.

That ordering is the entire argument. The mechanics below are what make it survive contact with a few hundred tenants.

## Why the first record you publish is a listening post

In the B2B SaaS version of this problem you don't own the zone. A customer points `mail.theirbrand.example` at your platform, adds your DKIM selector, and from that moment their domain reputation and your sending infrastructure are welded together — but every DNS change still has to go through their IT ticket queue. The first TXT record therefore is not an enforcement decision. It is a request for evidence.

```text
# stage 1 - listen only, nothing is enforced
_dmarc.theirbrand.example.  300 IN TXT "v=DMARC1; p=none; rua=mailto:dmarc@reports.yoursaas.example; fo=1; adkim=r; aspf=r"

# stage 2 - enforce on a sample, hold subdomains back
_dmarc.theirbrand.example.  300 IN TXT "v=DMARC1; p=quarantine; pct=25; sp=none; rua=mailto:dmarc@reports.yoursaas.example"

# stage 3 - full enforcement
_dmarc.theirbrand.example.  300 IN TXT "v=DMARC1; p=reject; sp=reject; rua=mailto:dmarc@reports.yoursaas.example"
```

Two details in stage one decide whether you get any data back at all. The first is the reporting interval: `ri` defaults to 86400 seconds, and RFC 7489 is explicit that the value is a request, not a contract — report generators are free to send on their own cadence, so a policy step observed over three days is really three samples, not seventy-two hours of continuous signal.

The second detail is the one that produces a silent, empty dashboard.

If `rua` points at a mailbox outside the policy domain, and in a multi-tenant product it always does, the receiver is supposed to verify that the external destination agreed to accept those reports. The authorization lives on your side of the fence: a TXT record at `theirbrand.example._report._dmarc.reports.yoursaas.example` containing `v=DMARC1` (RFC 7489, section 7.1). Forget it during onboarding and conforming receivers drop the reports, you see nothing, and the natural instinct is to blame the customer's DNS — which is the one place the fault is not. Provisioning that record belongs in the same automated job that hands the tenant their DKIM selector, not in a runbook step someone performs by hand.

## Should a DMARC policy move to quarantine or reject first?

Quarantine, and the reason has nothing to do with DNS. A quarantined message lands in a junk folder, where a human can still retrieve it and where your support team can still be told about it. A rejected message is gone at SMTP time, and the sender learns about it through a bounce that nobody on the customer's side reads. If the mail stream carries password resets, OTP codes, or invoices, that difference is the difference between a confused user and an unreconstructable incident. I have never seen a rollout where skipping the quarantine stage bought back more time than the first misrouted OTP batch cost.

Sampling is the other lever, and it is weaker than it looks. `pct=25` asks receivers to apply the policy to a quarter of failing messages and to fall back to the next-lower policy for the rest, which is useful for smearing risk across a few report cycles. The trade-off is that you now have a probabilistic failure mode: a low-volume tenant sending 400 messages a day gets a sample too small to distinguish "our forwarding path breaks alignment" from "nothing happened today." The revision work in the IETF DMARC working group removes `pct` and adds a separate policy for non-existent subdomains; I would check the current draft status before building tooling that depends on sampling behavior either way.

What actually gates the step is a readiness rule you can compute from the reports themselves.

```python
THRESHOLDS = {"quarantine": 0.985, "reject": 0.999}

def enforcement_ready(days, step, min_days=14, min_volume=500):
    """days: newest-last list of {"total", "aligned", "unclassified"} per report day."""
    window = [d for d in days[-min_days:] if d["total"]]
    if len(window) < min_days:
        return False, f"only {len(window)} usable report days"

    volume = sum(d["total"] for d in window)
    if volume < min_volume:
        return False, f"{volume} messages is too thin to generalize from"

    worst = min(d["aligned"] / d["total"] for d in window)
    if worst < THRESHOLDS[step]:
        return False, f"worst day aligned at {worst:.4f}, need {THRESHOLDS[step]}"

    unknown = sum(d["unclassified"] for d in window)
    if unknown / volume > 0.001:
        return False, f"{unknown} messages from sources nobody has claimed"

    return True, "advance"
```

The `unclassified` counter is the one that earns its keep. Alignment rates look wonderful right up to the moment you notice that 0.4% of the volume comes from an IP range belonging to the customer's e-signature vendor, which nobody mentioned during onboarding because nobody thought of it as email.

Two consecutive passing evaluations, then the record changes. Not before.

## Where custom domain onboarding quietly breaks alignment

The failure modes cluster in a few places, and none of them are DMARC bugs — they are consequences of stapling your sending path onto a zone somebody else has been editing for a decade.

SPF is the most common. RFC 7208 caps a single evaluation at ten DNS-querying mechanisms and says implementations should return `permerror` beyond it, and a mid-size customer arrives with an office suite, a CRM, a helpdesk, and a payroll tool already in the chain. Your `include:` is the eleventh. The resulting `permerror` is not a soft failure: the SPF side of alignment is simply unavailable, so the whole policy now rests on DKIM, and one rotated key away from silence. Flattening the record helps and creates its own maintenance debt, because a flattened record is a snapshot of somebody else's infrastructure.

CNAME delegation has a sharper edge. A name that carries a CNAME cannot carry other records (RFC 1034, section 3.6.2), so a `_dmarc` CNAME pointing into your zone is only installable if that name is otherwise empty — and on an established domain it usually is not. The same constraint is what makes `selector._domainkey` delegation work so well for DKIM: that name is new, you own the target, and key rotation stops being a customer ticket.

Then there is mail you never sent. Forwarding rewrites the envelope and breaks SPF; some mailing lists modify the body and break the DKIM signature. ARC (RFC 8617, Experimental) exists to let an intermediary vouch for the original result, but adoption is uneven, and its output is an assertion you choose to trust rather than a verification you perform. Expect a persistent floor of aligned-fail traffic that no configuration change removes, and set the reject threshold above it rather than chasing it to zero.

Subdomains deserve their own sentence: `sp` governs them, and a domain with `p=reject` inherited by a forgotten `staging.theirbrand.example` is a fine way to discover that someone's build system has been sending release notes for years.

## Who should own the record and what that choice costs

Only after the progression is settled does the ownership question become answerable, because ownership is purely a question of how fast you can undo a mistake.

| Record ownership | Time to change a policy | Rollback path | Main limitation |
| --- | --- | --- | --- |
| Customer edits their own TXT | Hours to weeks, ticket-bound | Same queue, same delay | Enforcement schedule is set by their change process, not your data |
| `_dmarc` CNAME into your zone | Minutes, plus remaining TTL | One record in a zone you control | Needs an empty `_dmarc` name, and you now own policy for mail you do not send |
| Dedicated sending subdomain | Minutes, scoped to the subdomain | Independent of the parent domain | Parent domain stays unprotected, so lookalike abuse is unaddressed |

The middle row is the one worth arguing about. Taking the CNAME means a policy step costs a deploy instead of a negotiation, which is exactly what you want when a readiness gate fires at 02:00. It also means that when the customer's finance team starts blasting statements from a tool you've never heard of, your record is what quarantines them. That is not a good fit for a tenant with a large, unmapped sending estate; leave those at `p=none` with their own team holding the pen, and spend the effort on the reports instead.

## Rolling a fleet of customer domains forward

Batch by evidence, not by signup date. A cohort of tenants whose last fourteen report days are clean moves together; everyone else waits, and the wait isn't a failure state.

Lower the TTL before you need it. TTL only affects caches populated after the change, so dropping `_dmarc` to 300 seconds the day before a cutover is useful and dropping it during an incident is theater. Negative caching has the same property from the other direction — RFC 2308 ties it to the zone's SOA minimum, with a recommended ceiling of a few hours — so a record that briefly went missing keeps hurting after it returns.

Instrument the pipeline that eats the reports, not only the mail. A tenant whose aggregate reports stop arriving for 48 hours looks identical to a tenant with perfect alignment, and the second one is a lie. Alert on ingestion gaps per tenant, keep the raw XML for at least a quarter, and store the rendered record alongside the decision that produced it so a rollback is a revert rather than an archaeology project. Zone-as-code tooling such as octoDNS or DNSControl turns the fleet-wide change into a reviewable diff, which matters more than it sounds when the diff spans four hundred zones.

**The rollback budget is what makes enforcement safe, not the enforcement plan.** One record, one revert, bounded by a TTL you chose yesterday.

And keep one honest caveat in view: none of this protects the customer's other domains, their lookalikes, or anything a receiver chooses not to evaluate. Enforcement narrows one specific abuse path — unauthenticated mail claiming an aligned domain — and a rollout that promises more than that will be measured against a promise it cannot keep.

## Sources

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 7208, Sender Policy Framework, DNS lookup limits: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 1034, CNAME restrictions: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 2308, Negative caching of DNS queries: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 8617, Authenticated Received Chain: https://datatracker.ietf.org/doc/html/rfc8617
- DMARCbis, current IETF revision work: https://datatracker.ietf.org/doc/draft-ietf-dmarc-dmarcbis/
