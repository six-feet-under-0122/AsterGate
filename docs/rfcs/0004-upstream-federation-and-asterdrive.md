# RFC 0004: Upstream Federation and AsterDrive Integration

- Status: Proposed
- Date: 2026-09-13
- Owners: AsterGate and AsterDrive maintainers
- Depends on: RFC 0001, RFC 0002, RFC 0003

## Summary

AsterGate will reuse AsterForge's existing upstream OAuth/OIDC driver mechanics and the proven
security boundaries in AsterDrive, and add family-specific LDAP/SAML2 upstream adapters under
RFC 0006. Organizations configure upstream connectors, while external identity resolution and
linking belong to the global AsterGate Account directory. A separate Membership decision admits
the Account to the client's organization. AsterDrive integrates as a normal OIDC client; migration
keeps its accounts intact and does not silently merge accounts by email.

## Current AsterDrive design

AsterDrive currently owns four external-auth persistence concerns:

- provider configuration;
- external identity linked to a local AsterDrive user;
- short-lived login flow with state, browser binding, nonce, PKCE verifier, and single consumption;
- email verification/recovery flow for profiles that cannot be safely resolved.

The public flow is:

```text
list enabled providers
  -> start provider authorization
  -> persist state-bound flow
  -> upstream callback and code exchange
  -> normalize identity_namespace + subject + profile
  -> resolve/link/provision AsterDrive user
  -> AsterDrive MFA/session/cookie completion
```

`aster_forge_external_auth` already owns product-neutral driver, descriptor, registry, discovery,
OIDC/OAuth, PKCE, nonce, claim normalization, and provider-specific mechanics for generic OIDC,
generic OAuth2, GitHub, Google, Microsoft, and QQ. AsterDrive correctly retains local-user policy,
provider persistence, callback routing, email trust, MFA/session completion, and audit.

## Reuse mapping

| Existing responsibility | AsterGate target | Ownership |
| --- | --- | --- |
| Forge driver and registry | use directly, extending only product-neutral capability | AsterForge |
| provider descriptor | admin UI and validation source | AsterForge mechanism + AsterGate presentation |
| AsterDrive provider row | organization-scoped `upstream_connectors` | AsterGate |
| AsterDrive external identity | global `account_external_identities` | AsterGate |
| AsterDrive login flow | upstream step inside AsterGate interaction | AsterGate |
| AsterDrive account resolution | global Account resolution, separate tenant Membership policy | AsterGate |
| AsterDrive MFA and cookies | global Account authentication and browser session | AsterGate |
| AsterDrive audit details | AsterGate-specific audit actions/details | AsterGate |

Existing OIDC/OAuth mechanics should be reused through Forge's API, not copied from AsterDrive.
Forge crates may be extended or added for product-neutral LDAP/SAML2 mechanics, with separate
protocol-family interfaces rather than forcing OAuth callback methods. Such work preserves
AsterDrive's behavior tests and leaves Account/Membership policy in AsterGate (RFC 0006).

## Connector model

Each upstream connector instance belongs to exactly one organization and contains:

- stable connector key and kind;
- display metadata and capability descriptor reference;
- encrypted protocol credentials (OAuth client secret, LDAP bind secret, or SAML signing material)
  and connector-specific options;
- protocol-appropriate trust anchors and endpoints (issuer/JWKS, SAML metadata/certificates, or
  LDAP TLS/bind/search configuration);
- scopes or attribute mappings appropriate to its protocol family;
- enabled state and profile synchronization policy;
- verified-identifier trust policy;
- global Account-linking and separate organization Membership-provisioning policy;
- optional organization/enterprise routing rules;
- optional, separately consented upstream token storage policy.

Connector kind and protocol are immutable after identities exist. Changing the verified upstream
issuer or another field that changes the global identity namespace requires an explicit migration
with collision analysis; rotating a client credential without changing the upstream trust domain
does not create a new Account identity.

The admin API returns redacted credential state, never the stored secret. Draft connection tests
accept an unsaved configuration without persisting it and report individual discovery/endpoint/key
checks. Testing a saved connector audits the action and does not mutate identity state.

## Federation interaction

```text
Application authorization request arrives at AsterGate
  -> resolve the OAuth client_id or protocol-specific SP registration to its Org
  -> validate request and create interaction
  -> select local or upstream authentication method
  -> start the selected OIDC/OAuth, SAML2, or LDAP family flow
  -> persist protocol-specific replay/binding/expiry facts under interaction
  -> validate callback/assertion or bound directory authentication
  -> adapter returns normalized, verified external identity evidence
  -> resolve or explicitly link global Account by namespace + subject
  -> establish global Account authentication and apply MFA policy
  -> resolve/create target organization Membership under separate authorization
  -> map trusted upstream group assertions to that Membership under organization policy
  -> enforce application group admission
  -> resume consent/authorization interaction
  -> issue AsterGate code, then AsterGate tokens to the client
```

An upstream token is never returned as the AsterGate access token. AsterGate validates the upstream
assertion, establishes an Account browser session, then issues its own tokens only after tenant
Membership and application admission succeed. The upstream issuer is an external identity
namespace, never AsterGate's OIDC issuer.

## Identity resolution policy

Resolution order is deterministic:

1. Match global `(identity_namespace, subject)` and load its Account; update only profile fields
   permitted by that Account's source policy.
2. If this is an authenticated link flow, require recent Account authentication, upstream proof,
   explicit consent, and collision checks before binding the external identity globally.
3. Otherwise, create a new Account only under the global registration policy, or enter bounded
   recovery. A matching email or phone may be displayed as a recovery hint but never auto-links.
4. Independently resolve the Account's Membership in the client-owned organization. Create one
   only through invitation, Management API provisioning, or an explicit trusted enterprise JIT
   policy with auditable evidence; email/domain equality alone does not suffice.
5. Apply the tenant group mapping and application admission policy. Missing, suspended, or denied
   Memberships receive no AsterGate authorization code or token.

An existing global external identity may authenticate without email in a later response, but
authentication never implies Membership in the target organization. A new Account without a
required verified identifier cannot bypass registration, Membership, or group admission policy.

## Social and enterprise connectors

Social and enterprise connectors may share a protocol-family driver, but not all policy:

- social connectors may be shown as general sign-in methods and support user-controlled linking;
- enterprise connectors are associated with organization/domain routing and may enforce exclusive
  login methods;
- enterprise connector MFA may be delegated to the IdP only under an explicit trust policy and
  authentication-context requirement;
- upstream directory groups map to stable local organization group IDs on an authorized
  Membership only through explicit connector policy; raw group names or email domains do not
  authorize application access;
- SAML connectors require separate metadata, assertion, signature, audience, recipient, replay,
  clock-skew, encryption, and certificate-rotation contracts.

SAML2 and LDAP have separate trust, replay, transport, and configuration contracts under RFC
0006. They are final architecture capabilities, not claims of current crate support. A SAML2
upstream SP adapter and a downstream SAML2 IdP adapter share tenant authorization services, not
an OAuth callback trait or independent Account store.

## Profile and upstream-token synchronization

Connector policy selects one of:

- populate profile only on first provisioning;
- update selected fields on each sign-in;
- never overwrite locally managed fields.

Each profile field has an owner/source policy. A later upstream login must not overwrite a user's
locally chosen name or avatar merely because the connector returned a value.

Saving upstream access/refresh tokens is disabled by default. When enabled, encrypted token records
have scopes, subject, expiry, rotation state, consent, and revocation lifecycle. Sign-in succeeds
independently of optional token persistence unless the application explicitly requires delegated
upstream API access.

## AsterDrive integration target

AsterDrive is registered as an AsterGate traditional web application or backend-for-frontend
client:

- exact AsterDrive callback and logout URIs;
- Authorization Code + PKCE;
- confidential client authentication at the token endpoint;
- requested OIDC identity scopes plus a dedicated AsterDrive API audience where required;
- AsterGate issuer/JWKS validation in AsterDrive;
- AsterDrive session creation after code exchange;
- AsterDrive product authorization remains local.
- the AsterDrive client is owned by one AsterGate organization and its admitted groups are
  configured explicitly. A separate tenant uses a separate client registration or an explicit
  multi-tenant RP contract; client parameters cannot switch organizations.

The preferred browser model is a backend callback in AsterDrive. Client secrets and refresh tokens
remain server-side; the frontend receives AsterDrive's own protected session cookie rather than
storing AsterGate bearer tokens in browser storage.

## Existing-account migration

Migration follows controlled steps:

1. Register AsterDrive in AsterGate and validate login with test accounts.
2. Keep AsterDrive local/external methods available while adding AsterGate as a new OIDC provider.
3. For an authenticated AsterDrive user, offer an explicit link to the returned AsterGate Account
   subject, with the organization context separately validated.
4. Verified email may assist recovery or show a candidate, but does not auto-link AsterDrive
   accounts or grant AsterGate Membership; require Account ownership proof and audit collisions.
5. Measure unlinked accounts and recovery outcomes before making AsterGate primary.
6. Disable legacy methods only after administrators have break-glass access and users have a proven
   migration/recovery path.
7. Retain legacy identity records through the rollback window; delete them only in a separate,
   explicit migration.

There is no automatic bulk merge based only on email. Existing AsterDrive user IDs, ownership,
workspaces, policy groups, files, and audit history do not move to AsterGate.

## Compatibility and rollback

The temporary dual-login period is a compatibility layer with a stated deletion condition: all
required accounts are linked or covered by recovery, AsterGate availability meets the deployment
target, logout/session behavior is verified, and operators have tested rollback.

If AsterGate is unavailable during migration, AsterDrive may retain explicitly configured legacy
login methods. After cutover, fail-open authentication is forbidden; accepting unverifiable tokens
would be worse than an availability failure.

## Required tests

- connector descriptor, normalization, secret redaction, and draft/saved connection tests;
- state/browser-binding mismatch, replay, expiry, PKCE, nonce, and provider-error tests;
- verified-email semantics for every built-in connector;
- global identity collision, authenticated linking, no-email-auto-link, Membership provisioning,
  and disabled-Account tests;
- same Account in multiple organizations, missing Membership, cross-organization group and grant
  reference rejection, and no-email-auto-enrollment tests;
- end-to-end AsterGate-as-OP to AsterDrive-as-client login, refresh, logout, MFA, and failure flows;
- dual-login migration, rollback, and account-collision fixtures;
- no raw tokens, codes, client secrets, or upstream payloads in logs/audit.
- LDAP transport/filter/credential and SAML metadata/signature/wrapping/recipient/audience/replay
  tests; no adapter may create a parallel Account, Membership, Group, or issuance fact.

## References

- AsterDrive external-auth design:
  `developer-docs/zh-CN/design/external-auth.md` in the AsterDrive repository
- AsterDrive auth flow state machine:
  `developer-docs/zh-CN/design/auth-flow-state-machine.md` in the AsterDrive repository
- AsterForge external-auth contract:
  `crates/aster_forge_external_auth/` in the AsterForge repository
- [Logto social sign-in](https://docs.logto.io/end-user-flows/sign-up-and-sign-in/social-sign-in)
- [Logto enterprise SSO](https://docs.logto.io/end-user-flows/enterprise-sso)
