# Is SMS OTP enough for seller 2FA login? SIM swap, phishing, and PSD2 risks

SMS OTP is enough to stay compliant and nowhere near enough to be safe: use a phishing-resistant factor as the primary 2FA login gate for marketplace seller accounts, keep SMS as the documented recovery path, and let integration effort set the migration pace. Neither GDPR nor PSD2 nor NIST bans codes over the phone network — they just refuse to treat SIM swap and phishing risks as your carrier's problem. The deciding constraint isn't cryptography. It's the plumbing: number collection and validation, delivery receipts, per-number rate limits, carrier registration paperwork, and a human support path for the seller whose number got ported away on a Sunday afternoon.

That plumbing is the part teams underestimate.

The scenario I keep coming back to is small and boring. A marketplace tells a seller that a new order landed, the seller taps the notification, and a login gate decides whether this is really them before exposing the buyer's shipping address and the payout account. Same notification pipeline, two very different security stakes — the order alert is a message, the login is an authentication decision — and mixing the two into "we already send SMS, so let's send codes over it too" is how the shortcut usually gets approved.

## Should we keep SMS OTP for seller login, given SIM swap and phishing risks?

Keep it as a fallback. Retire it as the primary factor. Two attacks explain the split, and they have different shapes.

SIM swap moves the possession factor to the attacker without touching your system at all. Someone social-engineers a carrier or an authorized retailer into moving the number to a new SIM, and every code you send lands on their device. Your logs look perfect — correct number, delivered receipt, code entered inside the validity window — which is exactly why detection has to come from behavior, not from delivery status. Carriers have tightened the process, and the CTIA best practices describe the industry side of message handling, but the recovery workflow stays outside your trust boundary. You inherit the risk, you don't control the control.

Real-time phishing is the more common one. A proxy page collects the seller's password, forwards it, prompts for the six digits it just triggered, and replays them within the sixty-second window. Time-based codes from an authenticator app have the same weakness, since neither the code nor the human knows which origin asked for it. That's the whole argument for WebAuthn credentials: the browser signs a challenge scoped to the origin, so a lookalike domain gets a signature it can't use. Phishing resistance is a protocol property here, not a training problem.

Delivery reliability matters too, though it argues in a subtler direction. Codes that arrive late push sellers toward the "didn't get it" button, and every extra attempt widens the window an attacker can work in. I'd rather cut the number of codes in circulation than tune the retry policy.

## Where GDPR, PSD2 and NIST draw the privacy and payment lines

Three regimes get quoted in these arguments, and they say different things. Worth separating them before anyone claims compliance requires SMS.

| Framework | What it constrains | Where SMS OTP lands |
| --- | --- | --- |
| GDPR | Phone numbers are personal data: minimisation, purpose limitation, processor terms with every aggregator in the delivery chain | Allowed, but the number you collected for security can't quietly become a marketing list |
| PSD2 SCA (Reg. (EU) 2018/389) | Two independent elements, plus dynamic linking of amount and payee for payment transactions | A possession element on paper, provided independence and dynamic linking hold |
| NIST SP 800-63B | Authenticator assurance levels; out-of-band authentication over the public telephone network is treated as restricted | Permitted with a documented risk assessment and a migration plan, not recommended as your target state |

The nuance most marketplace teams miss is scope. PSD2 strong customer authentication applies to payment accounts and electronic payments, so a plain seller dashboard login often sits outside it — right up until that dashboard lets someone change the bank account that receives payouts. Then you're authenticating a change that moves money, and the dynamic linking requirement stops being someone else's problem. Decide that boundary deliberately and write it down, because "is the payout screen in scope" is a question your auditor will ask in a form you can't answer with a shrug.

GDPR pushes the same way for a different reason. Every SMS you send hands a phone number to an aggregator and its downstream carriers, which is a processing chain you have to document, and possibly an international transfer you have to justify. A passkey generates no such chain. When a data protection review asks which third parties see seller identifiers, "none, the credential never leaves the device" is a much shorter conversation than a routing diagram.

## The integration effort you're actually signing up for

Here's the honest part of the trade-off, and it's the reason SMS keeps winning kickoff meetings: sending a code is a single HTTP call, and everyone on the team already understands it.

That estimate is wrong, but not because the call is hard. It's wrong because the code path is the smallest piece. You also need phone number normalisation to E.164, verification that the number belongs to the person enrolling, per-number and per-IP throttles, a replay-safe store for the code hash, sender registration for the countries your sellers live in, and a fallback channel for sellers who travel. Email as that fallback drags in its own authentication chain — SPF records under RFC 7208, plus alignment for anything you sign — and now you're running two delivery systems for one gate.

Passkeys invert the shape of the work. The protocol side is heavier on day one: attestation handling, credential storage, account recovery design when a seller loses their only device. After that the operational surface is close to flat, because there's no delivery to monitor, no carrier rules to track, and no per-message spend that scales with your order volume. If you already run one of the platform authenticator flows in a mobile app, most of the cost is in the recovery UX rather than the ceremony itself.

So the pragmatic sequencing looks like this. Ship passkey enrollment first for the accounts that touch payouts, keep SMS as the documented recovery path with a slower risk-scored step-up, and stop treating "SMS is easier" as a permanent property — it's easier only until the first port-out dispute.

## A step-up rule you can implement in an afternoon

The decision I want in code is not "which factor is best" but "what does this request need right now". Risk scoring keeps the seller's daily flow at one tap while forcing a real ceremony on the actions that move money.

```python
from dataclasses import dataclass

# Factors ranked by phishing resistance, not by convenience.
STRENGTH = {"passkey": 3, "totp": 2, "sms_otp": 1}

@dataclass
class Session:
    seller_id: str
    action: str              # "view_order" | "change_payout" | "export_buyers"
    device_is_known: bool
    country_changed: bool
    enrolled: tuple          # factors this seller has registered

def required_strength(s: Session) -> int:
    if s.action in ("change_payout", "export_buyers"):
        return 3             # payout changes get a phishing-resistant factor, no exceptions
    if s.country_changed or not s.device_is_known:
        return 2
    return 0                 # a known device reading a new order alert: session cookie is enough

def choose_factor(s: Session) -> str | None:
    need = required_strength(s)
    if need == 0:
        return None
    usable = [f for f in s.enrolled if STRENGTH[f] >= need]
    if usable:
        return max(usable, key=lambda f: STRENGTH[f])
    # Nothing strong enough is enrolled: send them through enrollment,
    # rather than silently downgrading the gate to whatever is available.
    return "enroll_passkey"

new_order = Session("sel_8891", "view_order", True, False, ("sms_otp",))
payout_edit = Session("sel_8891", "change_payout", True, False, ("sms_otp",))

assert choose_factor(new_order) is None
assert choose_factor(payout_edit) == "enroll_passkey"
```

Two things about that snippet. The downgrade branch is the interesting one — most incidents I've read about come from a system that quietly accepted a weaker factor because the strong one wasn't enrolled yet. And the assertions at the bottom aren't decoration: this is the kind of policy that belongs in a test file from the first commit, since the rules will keep changing as legal scope shifts.

I'd also keep the score inputs boring on purpose. Device recognition and a country change are things you can explain to a support agent at 2am; a learned risk model is not, at least not until you can show its false-positive rate on real traffic.

## What to measure before you copy this

Treat the rollout the way you'd treat a model change: define the metrics first, then let the numbers decide the migration pace. Four are enough to start.

- Phishing-resistant enrollment share, split by seller cohort — the only metric that tells you whether the migration is real
- Step-up rate on payout-changing actions versus routine order views, which catches a policy that's either too loose or so strict sellers route around it
- Fallback usage: what fraction of successful logins still finish over SMS, and whether that share is trending down
- Recovery ticket volume and time-to-resolution after a lost device or a ported number, because this is where a passkey rollout quietly fails

Build a small replay harness before you change the policy. Take a week of real login attempts, label the handful that turned into disputes or support tickets, and run candidate rules over them offline — the same discipline as scoring a retrieval change, and the notebook is honestly the right place to start. You'll find your thresholds are either trivially permissive or catch a third of your legitimate weekend traffic, and you'd much rather learn that from a CSV than from sellers.

The catch is that none of this helps a marketplace whose sellers are mostly on feature phones or shared devices, where passkeys aren't a good fit and SMS is the only channel that reaches everyone. Stick with SMS there, document the restricted-authenticator reasoning, tighten the number-change flow, and revisit when your device mix moves. I'm not sure there's a clean answer for that segment yet — the honest position is that platform coverage, not protocol design, is the blocker.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://eur-lex.europa.eu/eli/reg_del/2018/389/oj
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://www.w3.org/TR/webauthn-2/
- https://datatracker.ietf.org/doc/html/rfc6238
- https://datatracker.ietf.org/doc/html/rfc7208
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
