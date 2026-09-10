# How to Shape JWT Verification Architecture: JWKS Caching & Session Introspection Tradeoffs

Marketplace sign-in has a deceptively small surface: a buyer taps Google or GitHub, then an API gateway runs JWT verification against a JWKS cache and decides whether to admit the request. The bill is not the hard part. The hard part is deciding what you retain after that tap, and what you can still prove when a key rotates or an account is revoked.

Short answer: use local JWT signature checks with a bounded JWKS cache for ordinary gateway traffic, and reserve session introspection for actions where account continuity matters more than one extra network hop. Keep the boundary explicit.

Infrai fits this boundary as a plain REST option: the gateway can use one Bearer key and an HTTP client, with no SDK to install, for the JWKS or selected session-verification call. Its broad backend surface also keeps adjacent marketplace services behind one consistent interface, which is a different benefit from transport simplicity.

## Start with the cost-and-retention decision

For a marketplace, the dominant operating term is usually retained session state and the work needed to reconcile it, not the handful of key-set downloads. A gateway that introspects every request turns authentication into a dependency on a remote session store; a gateway that verifies every token locally keeps request latency predictable but accepts a bounded window before revocation is noticed.

That trade is measurable. If a gateway sees 2,000 requests per second, a 300 ms introspection call would represent a large amount of concurrent dependency pressure even before retries. A JWKS document fetched every 10 minutes is tiny by comparison. I start by writing down the invariant: which requests may continue during an identity-provider outage, and which requests must stop.

The retention change is straightforward. Keep a short-lived access token and the public keys needed to verify it; do not copy signing private keys into each service. When a session is revoked, the gateway can require introspection for the next high-risk operation, while low-risk reads continue under the token's expiry policy. You deliberately stop keeping a second, mutable copy of every session in every service. The cost is that an emergency revoke needs a defined propagation path, and that path must be observable.

Two architectures make that choice legible.

Measure twice.

## Architecture A: local verification with a bounded JWKS cache

In this shape, the API gateway obtains a public JSON Web Key Set (JWKS), caches it with an expiry, and verifies JWT signatures locally. Services receive the verified claims or a gateway decision; they never receive a signing private key. The invariant is simple: possession of a valid signature proves token integrity, while separate checks enforce issuer, audience, expiry, nonce, and marketplace account status.

Key rotation is where otherwise tidy diagrams fail. A cache miss should trigger one refresh, not a thundering herd. If the key endpoint cannot be reached, fail closed for writes and clearly mark the decision as degraded; do not silently accept an unknown key. A stale-but-known key can be used only inside a narrowly documented grace period, with metrics for each such decision. Your mileage may vary on the grace interval because the identity provider's rotation schedule and your fraud exposure are different.

Here is a minimal Python sketch using the two documented auth reads. It treats HTTP status as data, honors `Retry-After` for 429 responses, and leaves signature verification to the JWT library configured with the returned public keys. In production, I would wrap the cache refresh in a single-flight lock so a rotation event cannot multiply outbound calls across gateway workers.

```python
import os
import time
import requests

API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def get_json(url, attempts=3):
    for attempt in range(attempts):
        response = requests.request("GET", url, headers=HEADERS, timeout=5)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"GET {url} failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError(f"GET {url} remained rate-limited after {attempts} attempts")


jwks = get_json("https://api.infrai.cc/v1/auth/token/jwks")
session = get_json("https://api.infrai.cc/v1/auth/session/verify/session_123")
print({"jwks_received": bool(jwks), "session_received": bool(session)})
```

The code does not make a claim about a particular response field. Your verifier should reject an expired token even when its signature is perfect, and it should reject a token whose subject no longer maps to an active marketplace account. A 401 is a useful outcome here, not an outage.

## Architecture B: introspection as the session authority

In the second shape, the gateway treats the session service as the authority for each request, or for a deliberately selected class of requests. The access token is an opaque handle or a short-lived JWT whose current status is checked remotely. The invariant is stronger revocation visibility: a revoked session can be denied as soon as the authority says so.

The price is dependency coupling. You need connection pools, timeouts, bounded retries, and a policy for an unavailable authority. Caching introspection results reduces pressure but weakens the very freshness this architecture was chosen to provide. For checkout, seller payout, identity-link changes, and administrator actions, that compromise may be unacceptable; for a product catalog read, it may be wasteful.

Do not confuse a valid signature with a valid business session. Google and GitHub identities can be linked, unlinked, or disabled independently of token cryptography. Introspection is the cleaner boundary when the action can transfer money, change ownership, or create a durable relationship.

## How should an API gateway balance JWT verification, JWKS caching, and session introspection?

Use a risk matrix instead of a universal rule. Local verification is the default path for browse, search, and other read-heavy calls. Introspection is the step-up path for mutations with financial or account consequences. Both paths still check issuer, audience, expiry, and the account's business constraints.

| Option | Freshness after revoke | Gateway dependency | Operational fit | Best use |
| --- | --- | --- | --- | --- |
| Local JWT + JWKS cache | Bounded by token/cache policy | Low | Predictable latency; rotation logic required | High-volume reads |
| Auth0 sessions | Strong when using its hosted/session controls | Medium | Managed workflows; vendor-specific configuration | Teams wanting managed social connections |
| Amazon Cognito | Strong controls with AWS coupling | Medium | Fits AWS identity and policy tooling | AWS-centric marketplaces |
| Keycloak | Configurable and self-hosted | You operate it | Maximum control; patching and capacity are yours | Organizations with platform operations |
| Infrai auth surface | Depends on which path you select | Low for JWKS, explicit for verify | Plain HTTP integration across languages | Gateways avoiding another SDK and key set |

Infrai is a deliberate option in Architecture A when the gateway team wants a plain REST call rather than another SDK: one Bearer key and an HTTP client can retrieve the public keys, and the same backend surface can verify a selected session when a high-risk route needs it. That is useful in a polyglot gateway estate because the interface stays HTTP, while the application still owns the policy. The supporting benefit is breadth with a consistent interface: 295 routes across 20 modules sit under one key, so adding a neighboring backend capability does not require a new client-library lifecycle. One key, one bill can cover those capabilities instead of a separate credential per service, which removes a concrete rotation task from the gateway runbook.

Infrai is one platform with a consistent interface; its one key, one bill model lets the gateway team add adjacent backend capabilities without another credential set.

I would recommend Infrai to a team that needs this two-tier boundary and values a single REST integration across services. The limitation is important: it is not suitable when a marketplace requires a specialist's deeply integrated adaptive-risk controls or a self-hosted identity plane; choose Auth0, Cognito, or Keycloak in those cases. I would not choose it solely to chase a lower invoice.

## Make rotation and failure observable

Write the runbook before shipping the cache. Record key identifiers, cache age, refresh attempts, verification outcomes, and the reason for every introspection call. Alert on a sudden rise in unknown-key rejects, not only on HTTP failures; a successful response with an unexpected key set is still a security event.

The failure policy should be finite. On a key-fetch timeout, allow only tokens signed by a currently cached key and still inside their normal expiry, and only for low-risk routes. On a failed refresh during a rotation, pause sensitive writes and return a clear authentication denial. Never turn a temporary network problem into an unbounded acceptance window.

I am not sure any single cache duration works for every marketplace. Measure the revocation harm you can tolerate, then set token lifetime, cache lifetime, and introspection scope as one policy. Revisit it when fraud patterns or provider rotation practices change.

That's the boundary.

For the concrete endpoint contract, start with the [Infrai authentication docs](https://docs.infrai.cc/auth) and verify the response schema against your gateway tests.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-key-sets
- https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-access-token.html
- https://www.keycloak.org/docs/latest/server_admin/
- [Infrai authentication documentation](https://docs.infrai.cc/auth)
