# SAML 2.0 & SCIM — Enterprise SSO and Provisioning

**The other half of B2B identity.**

*Twenty-year-old XML standards still running every enterprise SSO sale you'll ever do — and how SCIM 2.0 ties them to user lifecycle.*

```
IdP -> browser -> SP -> SCIM-provisioned account
```

Federate · Assert · Provision · De-provision

---

## Table of Contents

1. [Topics](#slide-01--topics)
2. [A Brief History](#slide-02--a-brief-history)
3. [Roles & Vocabulary](#slide-03--roles--vocabulary)
4. [Metadata Exchange](#slide-04--metadata-exchange)
5. [SP-Initiated SSO — The Common Flow](#slide-05--sp-initiated-sso--the-common-flow)
6. [Assertions — The Payload](#slide-06--assertions--the-payload)
7. [NameID Formats](#slide-07--nameid-formats)
8. [SAML Attacks & Defences](#slide-08--saml-attacks--defences)
9. [Single Logout (SLO) — Why It Usually Doesn't Work](#slide-09--single-logout-slo--why-it-usually-doesnt-work)
10. [SAML vs OIDC — Which When](#slide-10--saml-vs-oidc--which-when)
11. [WS-Federation Legacy](#slide-11--ws-federation-legacy)
12. [SCIM — The Lifecycle Problem](#slide-12--scim--the-lifecycle-problem)
13. [SCIM 2.0 Protocol — Endpoints & Operations](#slide-13--scim-20-protocol--endpoints--operations)
14. [JIT vs SCIM Provisioning](#slide-14--jit-vs-scim-provisioning)
15. [Group Sync — The Gnarliest Detail](#slide-15--group-sync--the-gnarliest-detail)
16. [Real-World IdP Quirks](#slide-16--real-world-idp-quirks)
17. [IdP Brokers — Don't Implement N IdPs Yourself](#slide-17--idp-brokers--dont-implement-n-idps-yourself)
18. [The "B2B Enterprise Readiness" Checklist](#slide-18--the-b2b-enterprise-readiness-checklist)
19. [Summary & References](#slide-19--summary--references)

---

## Slide 01 — Topics

### SAML 2.0
- 30 years of enterprise SSO
- Roles, metadata exchange
- SP-initiated vs IdP-initiated
- Bindings: Redirect, POST, Artifact
- Assertions, signing, encryption
- Logout (SLO)
- NameID formats

### SCIM 2.0
- The lifecycle problem
- Resource model — User, Group, schema extensions
- Endpoints, filtering, PATCH, bulk
- JIT vs SCIM provisioning
- Group sync

### When & why
- SAML vs OIDC
- WS-Federation legacy
- IdP brokers — Auth0, Entra, WorkOS, Okta
- Multi-tenant SP — per-tenant IdP connections

### Operational
- B2B enterprise-readiness checklist
- Real-world IdP quirks
- SAML attacks & defences
- Migration patterns

---

## Slide 02 — A Brief History

```
2001 → 2002 → 2005 → 2011 → 2014 → 2018 → 2026
 │      │      │      │      │      │       │
 │      │      │      │      │      │       └ SAML still required in 95% of enterprise
 │      │      │      │      │      └ OIDC overtakes new SaaS
 │      │      │      │      └ SCIM 2.0 — RFC 7643/4
 │      │      │      └ SCIM 1.0
 │      │      └ SAML 2.0 — OASIS
 │      └ SAML 1.1
 └ SAML 1.0 — XML pioneers
```

### Why SAML stuck

- Enterprise IT bought into Active Directory + ADFS in the 2000s; ADFS speaks SAML.
- Every "official" identity-provider product (Okta, OneLogin, Ping) shipped SAML-first because that's what their buyers had.
- Once a SAML connection works, no-one wants to touch it.
- Compliance regimes (SOC 2, HIPAA, FedRAMP) reference SAML in their control templates.

### Why OIDC didn't kill it

OIDC arrived a decade after SAML. New SaaS adopt OIDC for end-user social-login; established enterprise SaaS keep SAML for B2B SSO. Most B2B products in 2026 ship *both*. SCIM is more universal — used regardless of which auth protocol the SP picks.

---

## Slide 03 — Roles & Vocabulary

```
[ Principal (user) ] ── [ Identity Provider (IdP) ] ── [ Service Provider (SP) ]
                          authenticates · issues          consumes assertions ·
                          assertions                       grants access
```

Mapping to OIDC: IdP ≈ OP, SP ≈ RP, Assertion ≈ ID Token. Same shape, different wire format.

### Vocabulary
- **Assertion** — signed XML statement about the user.
- **NameID** — the SAML "subject" identifier.
- **AttributeStatement** — the user's attributes (email, name, groups).
- **AuthnContext** — how they authenticated (password / x509 / mfa).

### Endpoints (on the SP)
- **ACS URL** — Assertion Consumer Service. Where the IdP POSTs the assertion.
- **SLO URL** — Single Logout Service.
- **Metadata URL** — XML describing the SP for IdPs to import.

### Endpoints (on the IdP)
- **SSO URL** — where the SP sends users to authenticate.
- **SLO URL** — where the SP sends logout requests.
- **Metadata URL** — XML describing the IdP, plus its signing certificate.

---

## Slide 04 — Metadata Exchange

SAML doesn't have OIDC's `/.well-known` auto-discovery. Trust is bootstrapped by exchanging an XML **metadata** file, once, between the IdP and SP.

### Service Provider metadata (excerpt)

```xml
<EntityDescriptor entityID="https://app.example/saml">
  <SPSSODescriptor protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <KeyDescriptor use="signing">
      <ds:KeyInfo><ds:X509Data>
        <ds:X509Certificate>MIID…</ds:X509Certificate>
      </ds:X509Data></ds:KeyInfo>
    </KeyDescriptor>
    <NameIDFormat>urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress</NameIDFormat>
    <AssertionConsumerService
        Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
        Location="https://app.example/saml/acs" index="0"/>
    <SingleLogoutService
        Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
        Location="https://app.example/saml/slo"/>
  </SPSSODescriptor>
</EntityDescriptor>
```

### Three import patterns

- **URL-based** — both sides expose `/saml/metadata`; admins paste the URL into the other side's UI; SP refreshes the IdP's metadata periodically. **Best.**
- **File-based** — admin downloads the IdP's XML, uploads it to the SP. Manual.
- **Per-tenant blob** — multi-tenant SPs accept either an IdP metadata URL or pasted IdP issuer / SSO URL / certificate.

### Certificate rotation — the silent breaker

IdP certs have expiry dates (1–3 years). If the SP imported metadata *once* at setup, the day the cert expires every login fails. Use URL-based metadata so the SP refreshes nightly.

---

## Slide 05 — SP-Initiated SSO — The Common Flow

```
Browser                 SP (your app)              IdP (Okta/Entra…)
   │                          │                          │
1. ├─GET /dashboard───────────▶│                          │
2. ◀─302 → IdP SSO URL with SAMLRequest─                 │
3. ├──GET IdP/sso?SAMLRequest=…&RelayState=…────────────▶│
4. ◀────────login + MFA at the IdP──────────────────────
5. ◀────────HTML form auto-POSTs SAMLResponse to SP/acs──
6. ├─POST /saml/acs (SAMLResponse)──▶│                   │
7. ◀─validate, set session cookie, 302 → /dashboard──    │
```

RelayState carries the original requested URL through the round-trip.

### IdP-initiated SSO

User starts at the IdP (their company "app dock"), clicks the SP's tile, IdP POSTs a SAMLResponse to the SP's ACS without any prior request. Operationally common but has security caveats — see SAML attacks.

### Bindings

- **HTTP-Redirect** — request goes in the URL (deflated + base64). Used for short SAMLRequests.
- **HTTP-POST** — request/response in a form body. Used for SAMLResponses.
- **Artifact** — small reference travels via the browser; SP fetches the real assertion server-to-server. Rare; nice security property.

---

## Slide 06 — Assertions — The Payload

```xml
<Assertion ID="_a72f…" IssueInstant="2026-05-06T08:42:13Z" Version="2.0">
  <Issuer>https://idp.acme.com/</Issuer>

  <ds:Signature>…RSA-SHA256 over the assertion or response…</ds:Signature>

  <Subject>
    <NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
      alice@acme.com
    </NameID>
    <SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <SubjectConfirmationData
          NotOnOrAfter="2026-05-06T08:47:13Z"
          Recipient="https://app.example/saml/acs"
          InResponseTo="_req_0xa1b2"/>
    </SubjectConfirmation>
  </Subject>

  <Conditions NotBefore="2026-05-06T08:42:13Z" NotOnOrAfter="2026-05-06T08:47:13Z">
    <AudienceRestriction><Audience>https://app.example/saml</Audience></AudienceRestriction>
  </Conditions>

  <AuthnStatement AuthnInstant="2026-05-06T08:42:00Z">
    <AuthnContext>
      <AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</AuthnContextClassRef>
    </AuthnContext>
  </AuthnStatement>

  <AttributeStatement>
    <Attribute Name="email"><AttributeValue>alice@acme.com</AttributeValue></Attribute>
    <Attribute Name="firstName"><AttributeValue>Alice</AttributeValue></Attribute>
    <Attribute Name="groups">
      <AttributeValue>Engineering</AttributeValue>
      <AttributeValue>Admins</AttributeValue>
    </Attribute>
  </AttributeStatement>
</Assertion>
```

### Validation checklist
- Signature verifies against the IdP's public key from metadata.
- `Issuer` matches the configured IdP entityID.
- `Audience` = your SP's entityID.
- Not before / not on or after window.
- `InResponseTo` matches the request you actually sent.

### Map to OIDC
- `NameID` ≈ OIDC `sub`.
- `AuthnContextClassRef` ≈ OIDC `acr`.
- `AuthnInstant` ≈ OIDC `auth_time`.
- `AttributeStatement` ≈ ID-token claims.

---

## Slide 07 — NameID Formats

| Format URN | Meaning | Best for |
|---|---|---|
| `…format:emailAddress` | The user's email address as the identifier. | Easy to integrate; breaks the day a user changes email or surname. |
| `…format:persistent` | Stable, opaque, per-SP identifier; survives email/name change. | **Production B2B SSO. Use this if you can.** |
| `…format:transient` | New identifier on every assertion. | Privacy-first portals; rarely seen in B2B. |
| `…format:unspecified` | Whatever the IdP feels like. | Legacy; brittle. Avoid. |
| `…format:X509SubjectName` | An X.509 DN, for smart-card / PIV. | Government, defence. |

### "My users keep getting duplicate accounts"

Almost always: SP keyed accounts by `NameID = emailAddress`, the user's email changed in the IdP, SP created a new local account on next login. Fix: key on `persistent`; treat email as a mutable attribute.

### "Persistent" is per-SP, not per-user

The same user looks like different `persistent` NameIDs to different SPs (by design — privacy). If you operate two related SPs and want the same identifier in both, you need an explicit cross-SP claim or a separate "userPrincipalName" attribute.

---

## Slide 08 — SAML Attacks & Defences

| Attack | How | Defence |
|---|---|---|
| XML Signature Wrapping (XSW) | Inject extra unsigned `<Assertion>` alongside the signed one. Library validates signed; SP code reads unsigned. | Use a library that resolves references via `Reference URI` + `ID`, not document position. Reject documents with multiple `<Assertion>` elements. |
| XML eXternal Entity (XXE) | SAML XML parser fetches a remote DTD; attacker exfiltrates files. | Disable DOCTYPE / external entity resolution in the XML parser. |
| Comment injection in NameID | `alice<!--ignore-->@evil.com` normalised differently between sig-validation and the application; user bound to wrong account. | Reject NameIDs containing comments / unexpected characters; canonicalise before comparison. |
| Replay | Re-using a captured assertion within its validity window. | Cache `AssertionID` for the validity window; reject duplicates. Pin `InResponseTo` and `NotOnOrAfter`. |
| IdP-initiated CSRF | Attacker provides a SAMLResponse to user's browser; SP creates session as attacker's account. | Default-disable IdP-initiated SSO. If you must enable it, pin `RelayState` to a session-bound token. |
| Algorithm downgrade | IdP signs with SHA-1 / RSA-1024 / unsigned response wrapper. | Pin `SignatureMethod` ≥ RSA-SHA256; reject SHA-1; require entire `<Response>` or assertion to be signed. |

### Don't write a SAML library

Use a maintained one. `passport-saml`, `python3-saml`, `pysaml2`, `spring-security-saml2`, `onelogin/php-saml`, `itfoxtec.identity.saml2`. They all have known issues; track their CVE feeds.

---

## Slide 09 — Single Logout (SLO) — Why It Usually Doesn't Work

### The promise

User clicks "Log out" anywhere → IdP propagates a logout request to every SP the user is currently signed into → all sessions die at once.

### The reality

- **Front-channel SLO** — IdP redirects the browser through every SP in turn. One slow / dead SP stalls the chain.
- **Back-channel SLO** — IdP makes server-to-server calls to every SP's logout endpoint. Many SPs never implemented it.
- Most production deployments either disable SLO or accept that "log out" only logs out of one app + the IdP.

### What actually works

- **IdP session timeout** — short sessions at the IdP side mean users re-authenticate frequently; SPs become read-from-IdP within a bounded window.
- **Force re-auth** on the IdP for sensitive actions (`ForceAuthn=true`) — equivalent to OIDC `prompt=login`.
- **CAEP / Shared Signals Framework** — push events from IdP to SPs ("session revoked"); the modern replacement.

### Practical advice

Don't promise users that "Log out" kills every app they're in. Tell them what it actually does. If you operate the IdP, invest in CAEP-style propagation; SLO is a footgun.

---

## Slide 10 — SAML vs OIDC — Which When

| Concern | SAML 2.0 | OIDC |
|---|---|---|
| Wire format | XML, base64-encoded, browser POST | JSON, URL parameters, Bearer JWTs |
| Crypto | XML-DSig (notoriously brittle) | JWS / JWE (compact) |
| Discovery | Manual XML metadata exchange | `/.well-known/openid-configuration` |
| Native / mobile apps | Painful — XML in a webview | Designed for it (PKCE + system browser) |
| API delegation | None — auth only, OAuth alongside | Built on OAuth — same token both jobs |
| Logout | SLO often broken; CAEP retrofit | Back-Channel Logout works at scale |
| Provisioning | JIT from assertion attributes; SCIM bolted on | JIT from ID-token claims; SCIM bolted on |
| Library quality | Variable; many CVEs | Mature, narrowly-scoped libraries |
| Enterprise IdP support | Universal — every IdP, 20 years | Universal — same IdPs added OIDC by 2018 |
| Buyer expectation in 2026 | Required by every enterprise procurement | Increasingly accepted as equivalent |

### When to support SAML

- B2B and any prospect's procurement form mentions "SSO".
- Integrate into Workday / Salesforce / SAP / ServiceNow ecosystems.
- Customer's IdP is ADFS or a 2010-era SSO product.

### When OIDC is enough

- Pure-consumer app — "Sign in with Google".
- Greenfield SaaS where you own the customer base.
- Mobile or native client apps.
- API-first products where you'd need OAuth anyway.

---

## Slide 11 — WS-Federation Legacy

### WS-Federation
- Microsoft's pre-OIDC federation — sibling of SAML.
- Default for ADFS < 2016 and SharePoint on-prem.
- Wraps a SAML assertion in a different XML envelope.
- Most modern ADFS / Entra deployments now offer SAML and OIDC alongside.

### Kerberos / IWA
- "Integrated Windows Authentication" — domain-joined machines authenticate transparently using Kerberos tickets.
- Still everywhere on intranets.
- Often the *back-end* of an ADFS / Entra IdP.

### JWT / proprietary header SSO
- Older enterprise apps use a proprietary HTTP header (`X-Remote-User`, `X-Forwarded-Email`) populated by an upstream auth proxy.
- Be careful: an app that trusts `X-Remote-User` directly is trivial to spoof if the proxy isn't enforced.

### LDAP / AD bind
- The original B2B "SSO" — SP authenticates the user by binding to the customer's LDAP / AD with the user's credentials.
- Means the SP *handles passwords directly* — the password anti-pattern.
- Replaced by SAML once IdPs were mature; some ancient products still ask for LDAP credentials.

---

## Slide 12 — SCIM — The Lifecycle Problem

SAML signs a user in. It says nothing about *creating* their account, *updating* their group memberships, or *de-provisioning* them on their last day. **SCIM 2.0** is the protocol that does that.

### The problem

- HR fires Bob at 17:00 Friday. IT removes him from AD.
- Bob still has accounts in 47 SaaS products.
- SAML-only: Bob can't log in any more, but his data is still there, his licences are still being paid, and a forgotten API key in his profile still works.
- SCIM-provisioned: IdP issues `PATCH /Users/{id} {"active": false}` at 17:00:05. Account suspended; data accessible to admins; licence freed.

### SCIM = "System for Cross-domain Identity Management"

- RFC 7643 (core schema), RFC 7644 (protocol), RFC 7642 (use cases). Final 2015.
- JSON over HTTPS — looks more like a REST API than SAML.
- IdP (or HR system) is the *client*; SP is the *server* exposing `/Users` and `/Groups`.

### A typical SCIM resource

```json
GET /scim/v2/Users/72f81b9e
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id": "72f81b9e",
  "userName": "alice@acme.com",
  "name": { "givenName": "Alice", "familyName": "Example" },
  "emails": [{ "value": "alice@acme.com", "primary": true }],
  "active": true,
  "groups": [
    { "value": "g1", "display": "Engineering" },
    { "value": "g2", "display": "Admins" }
  ],
  "meta": { "resourceType": "User",
            "created": "2024-09-12T10:00:00Z",
            "lastModified": "2026-05-03T14:22:11Z" }
}
```

---

## Slide 13 — SCIM 2.0 Protocol — Endpoints & Operations

| Verb & Path | Job |
|---|---|
| `GET /Users?filter=…` | List users (with filtering, paging). |
| `GET /Users/{id}` | Read a single user. |
| `POST /Users` | Create a new user. |
| `PUT /Users/{id}` | Replace a user (full payload). |
| `PATCH /Users/{id}` | Partial update — the everyday verb. |
| `DELETE /Users/{id}` | Hard delete (rare; most IdPs use `active: false`). |
| `POST /Bulk` | Atomic batch of operations. |
| `GET /Schemas` · `/ResourceTypes` · `/ServiceProviderConfig` | Self-describing metadata. |

### Filtering & PATCH

```
GET /Users?filter=userName eq "alice@acme.com"

PATCH /Users/72f81b9e
{
  "schemas":["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    { "op":"replace", "path":"active", "value":false },
    { "op":"add",     "path":"groups", "value":[{"value":"g3"}] },
    { "op":"remove",  "path":"groups[value eq \"g2\"]" }
  ]
}
```

### Schema extensions

```
"schemas": [
  "urn:ietf:params:scim:schemas:core:2.0:User",
  "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"
],
"urn:ietf:params:scim:schemas:extension:enterprise:2.0:User": {
  "employeeNumber": "12345",
  "department":     "Engineering",
  "manager":        { "value": "1f2c…", "displayName": "Bob Boss" }
}
```

---

## Slide 14 — JIT vs SCIM Provisioning

### Just-in-Time (JIT) provisioning

- SP creates the user account on first SAML / OIDC sign-in, populating attributes from the assertion / ID token.
- No separate provisioning channel; setup is "configure SSO and you're done".
- Cannot create users *before* they sign in.
- Cannot reliably *de-provision* — if a user no longer signs in, the account just sits dormant.

### SCIM provisioning

- IdP pushes user lifecycle events to the SP independently of sign-ins.
- Pre-create accounts (welcome email arrives Monday morning).
- Group changes propagate within seconds.
- De-provisioning is reliable.
- Required by enterprise procurement on every deal > 500 seats.

### In practice — both

Modern B2B SaaS supports both: SCIM for proper lifecycle, JIT as a fallback for users who slip through. The two should agree on the same identifier so the JIT user matches the SCIM-managed one.

---

## Slide 15 — Group Sync — The Gnarliest Detail

Provisioning a user is easy. Keeping their group memberships in sync across systems is where SCIM rollouts go sideways.

### The four traps

- **N×M mapping** — IdP has 5,000 AD groups; the SP has 12 roles. Mapping must be defined per-customer.
- **Nested groups** — IdP supports them; SCIM 2.0 doesn't natively. Most IdPs flatten before push.
- **Group name drift** — IdP renames a group; SCIM `displayName` updates but the SP's app code keys on the name not the `id`.
- **Filtered sync** — IdP only pushes groups whose name matches a prefix. Drop the filter, every AD group lands at the SP.

### Patterns that work

- **Customer-controlled mapping table** — UI in the SP lets the customer's admin map "AD group X → SP role Y".
- **Key on group `id`, not `displayName`.**
- **Per-tenant SCIM endpoint** — `/scim/v2/{tenant_id}/Users` — so each customer's IdP only touches its own users.
- **Idempotent PATCH** — re-applying the same operation must be safe, because IdPs replay on retry.

### Observability

Log every SCIM operation with: tenant, IdP request id, SP user id, operation type, before/after diff, outcome. When a customer reports "Bob still has access", you can trace it back.

---

## Slide 16 — Real-World IdP Quirks

### Microsoft Entra ID (formerly Azure AD)
- SAML *and* OIDC, on a per-app basis; SCIM via "Enterprise Apps".
- SCIM payloads sometimes omit `userName` on update.
- Group sync is filtered; admin must explicitly assign each group to the app.
- Microsoft's SCIM is the most-deviated.

### Okta
- The reference SAML/SCIM IdP — most SaaS test against Okta first.
- "Lifecycle Management" SKU is required for outbound SCIM.
- Profile mappings via Okta Expression Language can produce surprising assertion attribute values.

### Google Workspace
- SAML supports only IdP-initiated for many of its built-in apps.
- SCIM for outbound provisioning is supported but less mature than Okta / Entra.

### OneLogin / Ping / JumpCloud / ADFS
- **OneLogin** — long-standing, solid SAML; SCIM coverage varies.
- **Ping (PingFederate / PingOne)** — strong in regulated industries; very strict on signatures.
- **JumpCloud** — popular with smaller orgs.
- **ADFS** — still everywhere, on-prem; expects WS-Fed often, SAML on demand.

---

## Slide 17 — IdP Brokers — Don't Implement N IdPs Yourself

If your B2B SaaS will ever connect to more than two customer IdPs, don't build the SAML / SCIM stack *n* times. Use a broker.

### What a broker does

- Speaks SAML / OIDC / SCIM to the customer's IdP.
- Speaks one normalised protocol (usually OIDC) to your app.
- Each customer is a "connection" with its own metadata.
- Handles per-IdP quirks centrally.

### Brokers worth knowing

- **WorkOS** — "SSO + SCIM + Audit Logs as a service". The reference for B2B.
- **Auth0 (Okta CIC)** — Enterprise Connections — full broker plus customer login UX.
- **Frontegg / Stytch B2B / Descope** — newer entrants in the same space.
- **Keycloak** — open-source IdP that itself works as a broker for upstream IdPs.

### When to in-source

- You have a real cost reason — > ~10k SSO connections, broker pricing tips over.
- You handle classified or regulated data the broker can't touch.
- You need to ship the SP on-prem (broker is hosted SaaS).

### If you do in-source

Adopt a per-tenant abstraction from day one — every customer is an isolated SAML connection with its own metadata, certs, expiry, attribute mapping, SCIM endpoint, audit log destination. This is what the brokers spent years building.

---

## Slide 18 — The "B2B Enterprise Readiness" Checklist

### SSO
- SAML 2.0 SP-initiated & IdP-initiated.
- OIDC for newer customers.
- Per-customer metadata exchange via URL *or* file *or* manual.
- Per-customer X.509 cert with rotation reminders.
- NameID format: `persistent` default, others on request.
- Forced-auth / step-up via `ForceAuthn` / `acr_values`.
- Custom claim → role mappings configurable by the customer admin.

### Provisioning
- SCIM 2.0 server with at least `/Users`, `/Groups`, `/ServiceProviderConfig`.
- Bearer-token or OAuth-protected SCIM endpoint.
- Filtering, paging, PATCH (incl. nested PATCH).
- Soft-delete via `active: false`.
- JIT fallback for non-SCIM tenants.

### Audit & Compliance
- Per-tenant audit log streamed to customer (Splunk, Datadog, S3).
- SOC 2 Type II report on file.
- ISO 27001, where applicable.
- Data-residency options (EU / US / regional).
- DPA / sub-processor list / GDPR responses.

### Operational
- Per-customer admin UI for everything above.
- SLAs on SCIM propagation latency (< 5 min for de-provisioning).
- Test IdP — your customers' IT teams want to validate without touching their prod tenant.
- Friendly support response when "SAML stopped working last night" turns out to be cert rotation.

---

## Slide 19 — Summary & References

### What we covered

- Why SAML still runs the enterprise — 30 years of momentum
- Roles, endpoints, metadata exchange
- SP- and IdP-initiated SSO, bindings, RelayState
- Assertion structure, signature, validation, NameID formats
- SAML attacks (XSW, XXE, replay, comment injection) and defences
- Single Logout — and why CAEP is the modern replacement
- SAML vs OIDC — when each
- WS-Federation, Kerberos, LDAP-bind, header-based SSO
- SCIM 2.0 — the lifecycle problem, resource model, endpoints
- JIT vs SCIM provisioning, group sync gotchas
- Per-IdP quirks (Entra, Okta, Workspace, OneLogin, Ping)
- IdP brokers — when to use, when to in-source
- B2B enterprise-readiness checklist

### Three take-aways

1. **SAML is not optional for B2B.** Even greenfield SaaS that prefers OIDC ships SAML the moment a serious customer asks.
2. **SSO without SCIM is half a feature.** If you can't reliably de-provision, your enterprise contract has a security gap auditors will flag.
3. **Use a broker.** WorkOS, Auth0, or Keycloak as IdP-broker buy you years of "we already handle that quirk" — until in-sourcing has a real ROI.

### One-line takeaway

SAML is the enterprise tax; SCIM is the enterprise check. Pay both up-front and your B2B sales cycle gets shorter.

### References

OASIS SAML 2.0 Core (Mar 2005) · SAML 2.0 Bindings · SAML 2.0 Profiles · SAML 2.0 Errata · OASIS SAML V2.0 Implementation Profile for Federation Interoperability · RFC 7642 (SCIM Use Cases) · RFC 7643 (SCIM Schema) · RFC 7644 (SCIM Protocol) · XML-DSig Best Practices · OWASP SAML Security Cheat Sheet · OpenID SSF / CAEP · Auth0 / WorkOS / Stytch B2B SAML guides
