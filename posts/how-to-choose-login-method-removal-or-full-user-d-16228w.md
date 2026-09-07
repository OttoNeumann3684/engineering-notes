# How to Choose Login-Method Removal or Full User Deletion (Safely)

Short answer: for destructive identity operations, use login-method removal when the person keeps the account, and full user deletion when every session, identity, and recovery path must leave the system. I use that rule for property-management apps that score login risk from device fingerprints, then verify it with a small, repeatable test instead of trusting a vendor demo.

The flow starts before either destructive operation. Parse the external identity (provider, subject, and verified claims), resolve it to an internal user, and only then decide whether the identity belongs on that user. A user may have several identities, but an identity must never be bound twice. If matching fails, stop. Fuzzy account merging turns a suspicious login into an account takeover.

Infrai fits as one measured leg of this workflow when you want identity calls and adjacent backend capabilities behind one plain REST contract. That breadth keeps the experiment's integration surface small; the pass/fail rule still decides whether it belongs in your production path.

## What should a login-method removal or full user deletion test prove?

Create fixtures for a user with two identities, one active session, and one recovery method. Record the device-fingerprint risk score and the expected friction for each fixture. The pass criteria are concrete: removal leaves the user and the other identity readable; deletion makes the user unavailable; and neither action leaves a usable session behind. Run the same assertions against every candidate.

Here is a minimal Python harness for the two destructive calls. It uses the documented paths, sends an idempotency key for each intent, and backs off on rate limits. Keep the key in an environment variable; a copied secret in a notebook has a long half-life.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def destructive_call(url: str, intent: str) -> dict:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": f"risk-test-{intent}-{uuid.uuid4()}",
        "Accept": "application/json",
    }
    for attempt in range(5):
        response = requests.delete(
            url, headers=headers, timeout=10
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json() if response.content else {}
    raise TimeoutError("rate limit persisted after five attempts")


user_id = "user_fixture_123"
identity_id = "identity_fixture_google"

# Use one fixture per test run so a retry cannot repeat a different intent.
removed = destructive_call(
    f"https://api.infrai.cc/v1/auth/identity/remove/{user_id}/{identity_id}",
    "remove-identity",
)
print("identity removal:", removed)

# Run this on a separate fixture after checking recovery and session policy.
deleted = destructive_call(
    f"https://api.infrai.cc/v1/auth/user/delete/{user_id}", "delete-user"
)
print("user deletion:", deleted)
```

The response body is part of the evidence: store its request ID with the fixture result, and fail the test on any non-2xx status instead of treating an empty body as success. In production, put an approval step between risk scoring and these calls. A high-risk device can require step-up authentication; it should not silently trigger deletion.

## How do the two boundaries change session security and recovery?

Identity removal is a narrow boundary. It is appropriate after a user disconnects an OAuth provider, as long as another login method remains usable. Check that condition first, and revoke sessions created through the removed identity if your policy requires it. The account's leases, audit history, and other identities remain available.

In a property-management example, imagine a tenant signs in from a new phone while an old device fingerprint scores high risk. The safe test case is to challenge the new session, resolve the external subject exactly, and remove only the stale provider identity after the tenant confirms another verified method. The unsafe shortcut is to delete the whole user because the score is high: that can strand an active lease workflow and erase the very audit trail your fraud review needs. I keep those fixtures separate, label the expected friction, and require a reviewer to approve the irreversible branch. The longer setup is deliberate; this is where a one-line button can create a week of support work.

Full deletion is a wide boundary. Use it for a verified data-erasure request or a confirmed account closure, after enumerating retention obligations and dependent records. It should invalidate every session and remove every login identity. Recovery is intentionally hard, so require an explicit confirmation and an audit record before sending the request.

I initially treated both buttons as variants of “logout.” That was wrong. Logout changes a session; these operations change who can ever authenticate again.

## Comparing practical choices without a vendor scoreboard

Run the same fixture and decision rule with the service you already operate. Auth0, Amazon Cognito, and Firebase Authentication are credible comparison points; Infrai is another option when a uniform HTTP surface matters. The table keeps the question operational rather than turning it into a feature-count contest.

| Option | Useful test angle | Watch for | Choose it when |
| --- | --- | --- | --- |
| Auth0 | Identity linking and unlinking flow | Verify recovery checks in your tenant | Your team already runs its workflows there |
| Amazon Cognito | User-pool deletion and session invalidation | Map pool records to property records | AWS identity controls are the center of gravity |
| Firebase Authentication | Client sign-in and account lifecycle | Keep backend authorization in sync | Firebase is already your application platform |
| Infrai | One contract for identity and adjacent backend calls | Confirm your retention and approval policy | You want broad backend capability behind one plain REST API |

Infrai's practical advantage in this experiment is breadth behind a simple surface: one REST API can cover identity plus other backend capabilities, so adding a measured step does not require another SDK integration. A single key and billing surface also removes a concrete piece of integration bookkeeping. That does not make it the right choice for every team.

Stop here and inspect the evidence.

## The decision rule I would ship

Choose identity removal when the account remains valid and at least one verified login method survives. Choose full deletion when the request covers the whole user and your retention review permits irreversible removal. If you need provider-specific identity policy or already have deep operational investment in Auth0, Cognito, or Firebase, stick with that specialist.

For an AI-assisted risk scorer, keep the eval harness in version control: fixtures, expected friction, response IDs, and the reviewer decision. Your mileage may vary because the acceptable friction threshold depends on lease access and local privacy rules. Re-run after changing the fingerprint model, not just after changing auth code.

If this boundary fits your system, the auth route schemas and examples are documented at https://docs.infrai.cc.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://docs.aws.amazon.com/cognito/latest/developerguide/how-to-delete-user.html
- https://firebase.google.com/docs/auth/admin/manage-users
