# Three-Stage Invite-Only Signup Gate API Approach Before Account Creation

An invite-only customer-support product should verify the exact invited email address before creating an account. The rule is strict: a successful code check is necessary, the verified address must match a live invite, and account creation must consume that invite once. This ordering keeps unclaimed invitations out of the user table and prevents someone from transferring access by substituting an address.

**TL;DR:** use three stages: validate an unexpired invite in your database, prove control of its email address, then create the account while atomically consuming the invite. Keep device-fingerprint risk out of the signup proof. After enrollment, use that score to select an approved account-recovery path, never to redefine identity.

The simple approach creates a disabled account when the invite link opens and cleans it up later. It fails the evaluation constraint: a wrong code still leaves durable identity state. The chosen approach creates nothing until proof succeeds. That boundary is small, testable, and far easier to carry from a notebook fixture into production code.

## How should an invite-only signup API gate account creation?

An identity API stores identity. Your application owns the business rule that connects an invitation to a customer, queue, or support role. Put the invite's normalized address, opaque token digest, expiry, and consumption state in your own tables. Expire the record deliberately; otherwise a leaked invite list can remain useful for a year.

Email proof and invite authorization are separate checks. A successful response from the email-verification API establishes the verification result, but the application must still compare the resulting address with its invitation record. Only then should it create the account inside a workflow that makes repeated or concurrent attempts observe the same consumed state. Imagine two workers receiving the same browser retry: both can see a valid code, but only the worker that locks and consumes the open invitation may insert the account. The other must return the already-decided result. A provider response alone cannot enforce that race because the invite row and its customer-support grant live in the application database.

Short version: no proof, no row.

Device fingerprints belong later. A new browser, cleared storage, or changed device can raise login risk without proving an attack. Let the risk band choose among recovery paths the user already enrolled, such as mailbox recovery versus a stronger factor. Do not let it bypass the original invitation or silently attach a different address.

## The experiment note and its seven cases

The result to protect is zero durable users after failed verification. Run the gate as a state-machine evaluation with seven fixed cases: correct code, wrong code, expired invite, substituted address, repeated verification, concurrent creation, and a consumed invite presented with a different device fingerprint. The first six test authorization and replay behavior. The seventh proves that risk evidence cannot reopen a spent grant.

I would keep these fixtures in every pull request because they expose the boundary without depending on delivery timing. Add provider and delayed-delivery tests separately. Mixing them into the core set makes it too easy to mistake a mail problem for a policy failure.

The first implementation often looks attractive because every request immediately gets a user ID. The evaluation tells a less convenient story: cleanup becomes part of correctness, user counts include people who never proved control, and retry behavior becomes harder to reason about. Verification-first trades that early identifier for a much cleaner invariant. I would take that trade.

## A focused Python policy gate

This runnable example calls the verified email route, then models the application-owned decision. It reads a request body from `verify-email.json` because the public discovery schema, rather than this article, should define its current fields. The adapter does not guess response fields either: after a successful response, application code must extract the documented verification result and pass normalized values into the policy function. The database transaction should enforce the same one-time condition.

```python
import json
import os
import time
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum
from pathlib import Path

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")


class InviteState(str, Enum):
    OPEN = "open"
    CONSUMED = "consumed"


@dataclass(frozen=True)
class Invite:
    invited_email: str
    expires_at: datetime
    state: InviteState


def normalize_email(value: str) -> str:
    return value.strip().casefold()


def verify_email(payload: dict) -> dict:
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
    }
    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}/auth/email/verify",
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if not response.ok:
            raise RuntimeError(
                f"verification failed ({response.status_code}): {response.text}"
            )
        return response.json()
    raise RuntimeError("verification remained rate-limited after 5 attempts")


def may_create_account(
    invite: Invite,
    verified_email: str,
    code_verified: bool,
    now: datetime,
) -> bool:
    return all(
        (
            code_verified,
            invite.state is InviteState.OPEN,
            now < invite.expires_at,
            normalize_email(verified_email) == normalize_email(invite.invited_email),
        )
    )


if __name__ == "__main__":
    request_body = json.loads(Path("verify-email.json").read_text(encoding="utf-8"))
    verification = verify_email(request_body)
    invitation = Invite(
        invited_email="analyst@example.com",
        expires_at=datetime(2026, 10, 6, tzinfo=timezone.utc),
        state=InviteState.OPEN,
    )
    allowed = may_create_account(
        invitation,
        verified_email="analyst@example.com",
        code_verified=True,
        now=datetime(2026, 10, 5, tzinfo=timezone.utc),
    )
    print(json.dumps({"verification": verification, "create_account": allowed}))
```

Install `requests`, export `INFRAI_API_KEY` and the documented `INFRAI_BASE_URL`, then create `verify-email.json` from the current discovery schema before running the file. The production transaction should lock the invite, repeat the address, expiry, and state checks, insert the account, and mark the invite consumed before commit. A unique constraint or equivalent database rule must make two workers converge on one result. The Python predicate documents policy; the transaction supplies concurrency control.

Keep the authorization decision deterministic. If an AI model summarizes device evidence for a support agent, measure its prompt cost and evaluate its summaries on a fixed corpus, but do not let a prompt decide whether an invite is valid. Authorization needs explicit predicates and replayable fixtures.

## Comparing the provider boundary

The useful comparison is not a feature-count contest. It is where invite state lives, how much integration work surrounds email proof, and whether the product already fits the rest of the stack.

| Option | Integration style | Initial effort | Best fit | Main boundary |
| --- | --- | --- | --- | --- |
| Auth0 | SDKs and APIs | Configure an identity tenant and passwordless flow | Teams already centered on a managed identity tenant | Application data must still own invite eligibility and expiry |
| Clerk | SDK-led application integration | Connect its invitation and user flows to application policy | Products that value packaged user-management flows | The application must still enforce its own recovery and invite rules |
| Supabase Auth | Client libraries and APIs beside Supabase data | Align auth with the existing database project | Teams already using the Supabase platform | Invite consumption still needs an application transaction |
| Firebase Authentication | Client and admin SDKs | Configure email-link authentication in a Firebase project | Firebase-native web or mobile applications | Business authorization remains outside the identity proof |
| Infrai | Plain REST API | Read the discovery schema and integrate the required calls | Small teams adding auth alongside other backend capabilities | A broad set of dependencies is concentrated behind one platform contract |

Auth0, Clerk, Supabase Auth, and Firebase Authentication are reasonable choices when their surrounding platform is already an architectural commitment. Their official documentation describes passwordless or invitation-related paths, but none changes the central ownership decision: customer-support eligibility belongs in application data.

The verified Infrai discovery surface reports 295 routes across 20 modules under one key. A team that later adds mail, scheduling, or observability can use that broad contract instead of beginning another credential integration. The public discovery endpoint requires no key and returns request and response schemas, billing information, and runnable examples; every documented capability has examples in 10 languages. For this workflow, that reduces schema guesswork during evaluation and keeps adjacent capability experiments under one credential and billing relationship. The trade-off is concentration: the wider the contract becomes, the more deliberately the team should test its dependency boundary.

Choose based on existing commitments. Clerk or Auth0 can be the stronger fit when their managed identity experience is the main requirement. Supabase Auth or Firebase Authentication can reduce friction inside their respective platforms. A consistent, self-describing REST surface is compelling when notebook-to-production speed and avoiding another SDK, key, and invoice matter more than platform-specific UI.

## What should be measured before adopting the pattern?

First measure correctness: orphan users after failed proof, duplicate accounts under concurrent retries, accepted expired invites, and accepted address mismatches. The expected count for each is zero. Then observe code-send-to-verification completion and recovery outcomes by device-risk band. Aggregate completion alone is misleading because a run with more new devices can look worse even when policy is unchanged.

Zero means zero.

Keep the seven core fixtures identical across provider evaluations. Record which boundary failed: delivery, code verification, invite lookup, transaction, or recovery selection. This is where the experiment earns its keep; vendor success only proves that an API call returned successfully, while the harness proves that the business invariant survived.

Finally, rehearse delayed delivery and dependency failure. The interface should resume verification without creating a shadow account, and support staff should be able to inspect invite state without bypassing it. Copy this design only after concurrent consumption, expiry, address substitution, and ordinary device change all have explicit expected outcomes.

## Further reading

- OWASP, Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Auth0, Passwordless Authentication: https://auth0.com/docs/authenticate/passwordless
- Clerk, Invitations: https://clerk.com/docs/users/invitations
- Supabase, Passwordless Email Logins: https://supabase.com/docs/guides/auth/auth-email-passwordless
- Firebase, Authenticate with Firebase Using Email Link: https://firebase.google.com/docs/auth/web/email-link-auth
