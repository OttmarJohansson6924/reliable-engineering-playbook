# Where to Put Captcha — Signup, Login, and API Conversion Tradeoffs

TL;DR: Put a challenge on suspicious signup attempts and suspicious login attempts, not on every attempt in either flow. Keep the policy in the authentication API, run cheap controls such as rate limits first, and treat a successful challenge as one signal rather than proof of identity. For a media system, use the same risk decision when a stolen session tries to rotate a refresh token: reject replay, revoke the token family, and require fresh authentication. This preserves the conversion path for ordinary readers and contributors while raising the cost of automated abuse.

The least complex design that achieves that outcome has three parts. The browser asks the challenge provider for a short-lived response only when the API says a challenge is required. The API verifies that response server-side. The session service separately enforces refresh-token rotation and revocation, because captcha cannot make a stolen token safe. That boundary also keeps a challenge-provider outage from changing the semantics of session theft: an unavailable challenge can delay a risky interactive attempt, but it must never resurrect an invalidated refresh token or suppress replay detection.

## Should an API put captcha on signup, login, or both?

Use both signup and login as enforcement points, but trigger them from risk rather than page location. Signup abuse and credential attacks are different events, yet both can be automated. A public newsroom may see account creation used to manufacture comment identities, while its contributor login is a more valuable target for password spraying or credential stuffing.

The conversion trade-off is straightforward. An unconditional challenge adds a task for every human, including the large majority who may present no suspicious signal. A purely reactive login challenge leaves signup automation untouched. Risk-based placement spends friction only after controls such as per-account and per-network throttling, breached-password checks, or suspicious velocity indicate that another test is justified. OWASP describes captcha as defense in depth and recommends adaptive or risk-based authentication rather than relying on a challenge as the primary defense.

Friction is a budget.

Do not let the client make the final decision. A request field such as `captcha_required: false` is a display hint at best; the server recomputes policy from trusted observations. The response should also avoid revealing whether a newsroom contributor account exists. Generic authentication errors and consistent processing paths reduce account-enumeration clues.

## A runnable policy boundary

The useful unit is a small decision function that can be exercised in a notebook, replayed over an evaluation dataset, and then moved behind the production endpoint without rewriting the rule. This Python example uses explicit thresholds as application choices, not universal security constants. It also keeps the challenge adapter outside the policy so tests never need a live provider.

```python
from dataclasses import dataclass
from enum import Enum


class Action(str, Enum):
    ALLOW = "allow"
    CHALLENGE = "challenge"
    DENY = "deny"


@dataclass(frozen=True)
class AuthAttempt:
    flow: str
    failures_10m: int
    network_signups_1h: int
    known_device: bool
    token_replay: bool = False


def decide(attempt: AuthAttempt) -> Action:
    if attempt.token_replay:
        return Action.DENY
    if attempt.flow == "signup" and attempt.network_signups_1h >= 8:
        return Action.CHALLENGE
    if attempt.flow == "login" and attempt.failures_10m >= 4:
        return Action.CHALLENGE
    if not attempt.known_device and attempt.failures_10m >= 2:
        return Action.CHALLENGE
    return Action.ALLOW


evaluation_cases = [
    (AuthAttempt("signup", 0, 1, False), Action.ALLOW),
    (AuthAttempt("signup", 0, 12, False), Action.CHALLENGE),
    (AuthAttempt("login", 4, 0, True), Action.CHALLENGE),
    (AuthAttempt("refresh", 0, 0, True, token_replay=True), Action.DENY),
]

for attempt, expected in evaluation_cases:
    assert decide(attempt) == expected
```

The threshold values belong in configuration with version history. More important, the evaluation cases should include legitimate bursts: an editorial event can send many readers through signup at once, and a shared office network can make unrelated contributors look alike. IP address alone is a weak identity signal. Combine independent observations, minimize retained data, and document how long each signal survives.

Bots adapt.

Keep challenge verification server-side. The API checks the provider response, expected action or context, and freshness according to that provider's documented contract, then consumes it once. A timeout or unavailable verifier needs an explicit policy: low-risk traffic can be retried without silently treating failure as success, while high-risk authentication should fail closed. The UI can explain that another verification attempt is needed without exposing the score or threshold that triggered it.

## Captcha does not revoke a stolen session

A bot challenge operates at an interaction boundary. Refresh-token rotation operates at a credential boundary. Mixing those concepts creates a dangerous false positive: a thief who already holds a valid refresh token may also be capable of completing or outsourcing a challenge.

OAuth 2.0 Security Best Current Practice describes refresh-token rotation as issuing a new refresh token on every refresh and invalidating the previous one. If an invalidated token is later presented, the authorization server cannot know which party is legitimate, so it revokes the active refresh token associated with that grant. In implementation terms, store a token-family identifier and one-way token hashes, replace the current hash atomically, and revoke the family on replay.

This matters in a newsroom. After replay, terminate the affected contributor session family, require fresh authentication, and record a security event without writing raw tokens or challenge responses to logs. Other independent device sessions need a documented policy: preserving them reduces disruption, while account-wide revocation limits exposure when theft may extend beyond one device. That decision should follow the threat model, not the captcha result.

Captcha cannot settle it.

## Measure resistance without guessing at conversion

Start with an offline replay set containing labeled signup, login, and refresh events. The labels should distinguish known automation, legitimate retries, shared-network bursts, and confirmed token replay. Run policy versions over that fixed set before deployment. This is the notebook-to-production path that keeps a clever rule from becoming an untestable pile of endpoint conditionals.

Then stage the rule and watch challenge rate, completion rate, authentication success after challenge, repeated failures, signup activation, and token-family revocations. Segment by flow and risk reason. A single aggregate completion number can hide a login rule that is acceptable and a signup rule that blocks a particular accessibility path.

No invented benchmark helps here. Establish a baseline from the application's own traffic, set an error budget for legitimate-user friction, and compare policy versions against it. The policy computation is cheap; external verification adds latency and a network dependency, so challenge only after local controls have done their work. This also keeps prompt and model costs at zero for the security decision. Authentication policy should remain deterministic and auditable. Accessibility belongs in the release gate. W3C documents why visual and audio captcha mechanisms can exclude users and why alternative methods matter. Test keyboard operation, screen-reader announcements, expiration recovery, and a nonvisual path. A security control that locks out a contributor during a breaking story is an operational failure even when it blocks bots. Include those accessibility cases in the same release evaluation rather than treating them as a manual check after the risk thresholds have already shipped.

## Operational checklist, in prose

Before shipping, confirm that signup and login share one server-owned policy contract while retaining separate thresholds and metrics. Rate-limit by more than one key, return generic authentication errors, validate challenges only on the server, and never log secrets. Make challenge responses short-lived and single-use according to the selected verifier's contract. Keep a bypass procedure for support incidents, but protect it as a privileged operation with audit records rather than a client flag.

For session response, make refresh rotation atomic and test two near-simultaneous uses of the same old token. The expected result is one successful replacement followed by family revocation when replay is detected; race handling must not mint two valid descendants. Exercise key rotation, verifier timeouts, clock skew, storage failure, and rollback in staging. Alert on changes in challenge volume and replay detections, not on raw challenge data.

Finally, assign owners for the abuse policy, the accessibility review, and the incident runbook. Review thresholds after traffic shifts instead of placing a permanent challenge on everyone. **The decision is both flows, selectively:** risk signals decide when to add friction, while token rotation and revocation contain stolen-session damage.

## Further reading

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OAuth 2.0 Security Best Current Practice, RFC 9700: https://www.rfc-editor.org/rfc/rfc9700.html
- NIST Digital Identity Guidelines, Authentication and Lifecycle Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- W3C, Inaccessibility of CAPTCHA: https://www.w3.org/TR/turingtest/
