# 🪪 SAML 2.0 & SCIM — Enterprise SSO and Provisioning

An interactive Reveal.js presentation on the **other half** of B2B identity. Twenty-year-old XML standards still running every enterprise SSO sale you'll ever do — and how SCIM 2.0 ties them to user lifecycle.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/SAML_and_SCIM/)

## 📄 [Markdown Version](presentation.md)

## 📚 Companion decks — [Introduction to OAuth](https://brendanjameslynskey.github.io/Introduction_to_OAuth/) · [Introduction to OpenID Connect](https://brendanjameslynskey.github.io/Introduction_to_OpenID_Connect/) · [Authentication Methods](https://brendanjameslynskey.github.io/Authentication_Methods/) · [Authorization Models](https://brendanjameslynskey.github.io/Authorization_Models/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Federate · Assert · Provision · De-provision |
| 02 | Topics | SAML, SCIM, when, operational |
| 03 | A Brief History | 2001 → 2026 — why SAML still runs the enterprise |
| 04 | Roles & Vocabulary | IdP, SP, principal, assertion, NameID, AttributeStatement |
| 05 | Metadata Exchange | XML metadata, the three import patterns, certificate rotation |
| 06 | SP-Initiated SSO | The 95% flow — sequence diagram, RelayState, bindings |
| 07 | Assertions | Structure, signature, validation checklist, OIDC mapping |
| 08 | NameID Formats | persistent / emailAddress / transient / unspecified |
| 09 | SAML Attacks & Defences | XSW, XXE, replay, comment injection, IdP-initiated CSRF |
| 10 | Single Logout | Why it usually doesn't work; CAEP as the modern replacement |
| 11 | SAML vs OIDC | Decision matrix |
| 12 | WS-Federation Legacy | WS-Fed, Kerberos, LDAP-bind, header-based SSO |
| 13 | SCIM — The Lifecycle Problem | Why SAML alone leaves dormant accounts |
| 14 | SCIM Protocol | Endpoints, filtering, PATCH, schema extensions |
| 15 | JIT vs SCIM Provisioning | When each, and the practical "use both" reality |
| 16 | Group Sync | The four traps and patterns that work |
| 17 | Real-World IdP Quirks | Entra, Okta, Workspace, OneLogin, Ping, ADFS, JumpCloud |
| 18 | IdP Brokers | WorkOS, Auth0, Stytch B2B, Frontegg, Keycloak — when to in-source |
| 19 | B2B Enterprise-Readiness Checklist | Procurement-team table-stakes |
| 20 | Summary | Three take-aways and references |

---

## Audience

- B2B SaaS engineers shipping their first enterprise customer ("can we do SSO?").
- Architects choosing between in-housing the SAML/SCIM stack and using a broker.
- Security teams reviewing SAML CVEs (XSW, XXE) and modernising to OIDC + SCIM + CAEP.
- Anyone who has to read or write a SAML assertion and doesn't want to start cold.

This deck assumes you've seen the OAuth/OIDC series; SAML and OIDC solve overlapping problems, and the comparisons are easier with both in your head.

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono · inline SVG diagrams.

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## See also

- [OAuth — A Gentle Primer](https://github.com/BrendanJamesLynskey/OAuth_Primer) · [Introduction to OAuth](https://github.com/BrendanJamesLynskey/Introduction_to_OAuth) · [OAuth for MCP Servers](https://github.com/BrendanJamesLynskey/OAuth_for_MCP)
- [Introduction to OpenID Connect](https://github.com/BrendanJamesLynskey/Introduction_to_OpenID_Connect) · [Advanced OpenID Connect](https://github.com/BrendanJamesLynskey/Advanced_OpenID_Connect)
- [Authentication Methods](https://github.com/BrendanJamesLynskey/Authentication_Methods) — passwords, MFA, passkeys.
- [Authorization Models](https://github.com/BrendanJamesLynskey/Authorization_Models) — what to do *after* the SSO assertion.
- [Cloud_aaS_04_SaaS_Architecture](https://github.com/BrendanJamesLynskey/Cloud_aaS_04_SaaS_Architecture) — multi-tenant SaaS architecture patterns.

## References

OASIS SAML 2.0 Core (Mar 2005) · SAML 2.0 Bindings · SAML 2.0 Profiles · SAML 2.0 Errata · OASIS SAML V2.0 Implementation Profile for Federation Interoperability · RFC 7642 (SCIM Use Cases) · RFC 7643 (SCIM Schema) · RFC 7644 (SCIM Protocol) · RFC 7517 (JWK) · XML-DSig Best Practices · OWASP SAML Security Cheat Sheet · OpenID SSF / CAEP · Auth0 / WorkOS / Stytch B2B SAML guides

## License

Educational use. Code examples provided as-is.
