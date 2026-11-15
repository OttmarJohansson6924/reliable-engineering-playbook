# GDPR Evidence for Seller SMS Alert APIs in Europe With Inbound Support

A marketplace seller needs an order notification that the team can explain later. Short answer: for a US startup sending SMS alerts into Europe, start with a deterministic evidence check before dispatch. Compare API alternatives on the records they can supply for the intended destination, sender registration, data handling, and inbound replies. Let those requirements determine the shortlist before comparing commercial terms.

The useful output is a decision attached to one order event: which approved policy allowed this notification, which sender evidence it used, and where a reply would go. A successful API request alone cannot answer those questions. This is a good notebook-to-prod boundary because its inputs can become fixtures and its decisions can become regression expectations.

Keep it small.

## An order event with an evidence trail

An order event enters a notification worker, which selects a previously reviewed destination policy and the seller's communication preference. The worker checks the evidence references required by that policy, renders the approved transactional template, and passes an allowed notification to a generic transport adapter. It records the decision before dispatch and attaches subsequent transport results to that record. When the workflow requires replies, a separate inbound handler must connect the response to the seller's support workflow.

The following example implements only the decision step. Its policy is an illustrative marketplace requirement, not a statement of European law or a claim about any carrier. The fixture uses a pseudonymous seller reference and no phone number. Evidence identifiers stand for documents or review records stored elsewhere; the example does not validate those documents.

## What evidence should an SMS alert API retain for Europe and inbound support?

Run this example with Python 3.10 or later. It needs only the standard library and sends no messages. Three assertions turn the initial notebook experiment into a small evaluation harness: permit the reviewed fixture, block missing sender evidence, and block a reply-dependent workflow without a recorded reply route.

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True)
class Policy:
    reference: str
    destination: str
    sender_evidence_required: bool
    replies_required: bool


@dataclass(frozen=True)
class OrderAlert:
    event_id: str
    seller_ref: str
    destination: str
    template_ref: str
    preference_ref: str
    data_review_ref: str
    sender_evidence_ref: str | None
    reply_route_ref: str | None


@dataclass(frozen=True)
class Decision:
    event_id: str
    policy_ref: str
    allowed: bool
    reasons: tuple[str, ...]


def evaluate(alert: OrderAlert, policy: Policy) -> Decision:
    reasons = []
    if alert.destination != policy.destination:
        reasons.append("destination_policy_mismatch")
    for field in ("template_ref", "preference_ref", "data_review_ref"):
        if not getattr(alert, field):
            reasons.append(f"missing_{field}")
    if policy.sender_evidence_required and not alert.sender_evidence_ref:
        reasons.append("missing_sender_evidence")
    if policy.replies_required and not alert.reply_route_ref:
        reasons.append("missing_reply_route")
    return Decision(
        event_id=alert.event_id,
        policy_ref=policy.reference,
        allowed=not reasons,
        reasons=tuple(reasons),
    )


policy = Policy("policy-demo-r3", "DE", True, True)
alert = OrderAlert(
    event_id="order-demo-1042",
    seller_ref="seller-demo-17",
    destination="DE",
    template_ref="order-notice-r2",
    preference_ref="preference-demo-8",
    data_review_ref="review-demo-6",
    sender_evidence_ref="sender-demo-4",
    reply_route_ref="support-demo-2",
)

assert evaluate(alert, policy).allowed
assert evaluate(replace(alert, sender_evidence_ref=None), policy).reasons == (
    "missing_sender_evidence",
)
assert evaluate(replace(alert, reply_route_ref=None), policy).reasons == (
    "missing_reply_route",
)
print(evaluate(alert, policy))
```

A reference is a pointer, not a compliance verdict. The example's limitation is deliberate: it checks record completeness, while a production resolver must check the referenced artifact's applicability and validity. An arbitrary nonempty string passes this teaching example, so exposing these fields directly to an untrusted caller would defeat the intended control. Keep evidence resolution inside the trusted worker.

The separation matters for an AI application. A model can help draft an order-notice template for review, but the dispatch decision should consume explicit policy fields. Sending order details and compliance documents through a prompt on every notification adds model work without making this particular check more useful. Keep prompts out of the hot path unless they perform a defined task with measurable acceptance criteria.

## Compare the evidence behind sender and reply claims

For a startup evaluating European SMS alert APIs, a broad statement such as “European coverage” leaves the important questions unanswered. Ask for evidence covering the particular destination and sender configuration you intend to deploy. If registration is required for that configuration, what artifact records its status and scope? Who owns the registration, and what must be reviewed when the sender changes? Treat an unresolved answer as an unresolved deployment decision.

Don't silently convert it into approval.

Inbound support deserves its own acceptance check. Specify the actual conversation: a seller receives a new-order notice, replies with a question, and expects the marketplace to associate that reply with a support case. Ask whether the proposed sender configuration supports that path, how inbound events are authenticated, and which identifiers survive the handoff. Demonstrate the path using the intended configuration before marking the evidence complete. An outbound acceptance response is not a test of an inbound workflow.

For GDPR review, assemble a packet of questions for the responsible legal and privacy reviewers: which parties handle recipient data, which processing locations and transfer arrangements apply, what retention commitments exist, and how requests concerning that data will be handled. Record the reviewed documents and the review's scope. This article does not establish a lawful basis or certify a service's compliance; those conclusions require the actual processing arrangement and an appropriate review.

That is the comparison axis: evidence sufficient for this seller workflow. A service with a polished send example can still leave the team with unanswered registration or reply-routing questions. The trade-off is review effort versus unresolved deployment assumptions: extra integration work may be acceptable when the resulting records meet a requirement the marketplace actually has. Make that decision explicit before implementation expands.

## Keep message shape and operational state outside the prompt

Template approval includes message shape. The SMS segmentation reference documents 160 characters for a single GSM-7 message and 70 for a single UCS-2 message, with lower per-segment capacities for concatenated messages. Those are encoding boundaries, not a promise that every 160-character string fits into one segment.

Character choice matters.

For this order notice, prefer an approved template with bounded variable fields over unconstrained generated prose. Evaluate the rendered message, including seller-specific substitutions, before dispatch. A long item name or a character outside the expected alphabet is a concrete fixture worth keeping. This makes transport usage reviewable without turning changing unit prices into the design argument. Prompt-cost awareness belongs here too: draft and review template variants once, then render the accepted version deterministically.

Production needs more state than the example shows. Give the logical notification a durable identity and record each transport attempt separately. A timeout leaves an uncertain outcome; blindly repeating the send can produce duplicate seller alerts. Define how the adapter will reconcile an uncertain attempt, what evidence it needs before retrying, and when a human must investigate. Keep provider acceptance, later status events, and inbound replies as separate observations rather than collapsing them into a single “delivered” flag. These are proposed operational requirements, not guarantees about an unspecified API.

Keep the record lean. Store the policy and template versions, decision reasons, evidence references, and attempt identifiers needed to investigate the event. Decide separately whether recipient identifiers or message bodies must be retained, who can access them, and for how long. Copying the entire order into every diagnostic record makes the evidence packet harder to review and gives the privacy review more data flows to consider.

If email is an approved alternative channel, evaluate its sender requirements separately. The email sender guidelines in the references address authentication and related sending requirements; they do not establish SMS sender registration or inbound behavior. A channel change needs its own reviewed policy.

## Promote the check with its evidence

Before deployment, rerun the three fixtures and add destination mismatch, missing template approval, and an evidence reference that fails resolution. Keep the expected reasons with each fixture. Review policy changes alongside those expectations so that an accidentally permissive change is visible before the next order arrives. This is eval-driven development with a useful target: the dispatch decision.

Then walk one synthetic order through the worker, the transport adapter, the status handler, and the reply workflow. Confirm that the resulting record lets a reviewer trace the policy decision and each later observation. Assign an owner for stale evidence and a queue for blocked notifications; a blocked seller alert still needs an operational response.

The release criterion is concrete: the marketplace can explain why a particular seller notification was permitted, which evidence supported that decision, and how a reply reaches the right team. Once that is demonstrable, API comparisons have a defensible basis.

## References

The SMS reference supports the encoding limits discussed above. The email reference applies to the optional email channel. The evidence policy and fixtures are illustrative design choices, not country-specific registration rules.

## Further reading

- SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
- Email sender guidelines for a separately reviewed email channel: https://support.google.com/a/answer/81126
