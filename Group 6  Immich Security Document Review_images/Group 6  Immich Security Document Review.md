## Group 6 Team Based Project - Immich Security Documentation Review

I reviewed the current Immich security policy, OAuth/OIDC documentation, system-settings documentation, and published security advisories. This provides a useful picture of both Immich's intended security model and areas where implementation vulnerabilities have historically occurred.

### 1. Security policy

Immich maintains a formal SECURITY.md that tells researchers to report vulnerabilities privately through GitHub Security Advisories or the project's security email. The policy also defines what the project considers inside and outside its security scope.

A particularly important design decision is that Immich treats several infrastructure responsibilities as administrator responsibilities. PostgreSQL, Redis, reverse-proxy configuration, and similar surrounding infrastructure are outside the project's stated vulnerability scope. Likewise, immich-machine-learning is intended to remain inaccessible outside the container network.

For authentication, the policy explicitly says that **login rate limiting and MFA should be delegated to an OAuth server** rather than being treated as native Immich login-hardening features.

**Security implication:** A secure Immich deployment depends on both Immich's controls and correctly secured external infrastructure.

## 2. Authentication documentation

Immich supports two important authentication mechanisms:

- local username/password authentication;
- third-party OAuth/OIDC authentication.

Administrators can disable local password authentication. Immich warns that disabling both password authentication and OAuth would prevent users from logging in, although the Server CLI can be used to restore password login.

For higher-security environments, this architecture allows authentication to be moved to an external Identity Provider that supplies controls such as MFA.

## 3. OAuth/OIDC security

Immich's OAuth documentation provides considerably more security-specific configuration information.

Immich uses **OpenID Connect (OIDC) over OAuth 2** and documents compatibility with providers including Authentik, Authelia, Okta, Google, and Keycloak. The recommended client configuration is:

Confidential client → Web application → Authorization Code grant

It also documents explicit redirect URIs for the web and mobile applications.

This is a positive feature because OAuth security depends heavily on restricting where authorization responses can be returned.

### Relevant OAuth controls

| **Control** | **Immich documentation** | **Security purpose** |
| --- | --- | --- |
| Authorization Code flow | Documented | Avoids exposing authentication tokens directly through the browser authorization response |
| Confidential client | Documented | Allows client authentication using a client secret |
| Redirect URI configuration | Documented | Helps prevent authorization responses being redirected to unauthorized applications |
| ID-token signing algorithm | Default RS256 | Supports cryptographic verification of ID tokens |
| OAuth scopes | Default openid email profile | Limits/request identity information |
| Auto Register | Configurable | Controls automatic account provisioning |
| Role Claim | immich\_role | Allows external authorization information |
| Backchannel logout | Supported | Allows the IdP to communicate logout events |
| Request timeout | Default 30 seconds | Limits excessively long OAuth HTTP operations |

One security-sensitive point is **Auto Register**, which defaults to enabled. An administrator who wants pre-provisioned accounts rather than automatic creation should therefore deliberately disable it.

## 4. MFA and brute-force protection

The documentation exposes an important limitation in Immich's security model.

Immich does **not treat native MFA and login rate limiting as responsibilities of its built-in authentication mechanism**. The project's security policy says these controls should be delegated to an OAuth server.

This position is also consistent with historical maintainer responses to requests for built-in 2FA and brute-force protection: users were directed toward an external OAuth authentication system.

From a security-requirements perspective, therefore, the requirement should be allocated across systems:

**SR-AUTH-01:** The operational Immich environment shall provide brute-force-resistant authentication and MFA for accounts requiring enhanced authentication assurance, either through the configured Identity Provider or another approved authentication control.

The distinction matters: **Immich supports integration with systems that provide these protections; the core application should not be documented as providing native MFA simply because OAuth is available.**

# 5. Security advisories reveal important historical weaknesses

The strongest part of the documentation review is comparing the intended security model with Immich's actual vulnerability history.

The project's advisory list currently contains vulnerabilities involving authentication, authorization, XSS, SSRF, redirects, API privileges, and archive handling.

### OAuth account hijacking

A particularly relevant 2025 advisory describes an OAuth account-hijacking vulnerability caused by failure to check the OAuth state parameter.

According to the advisory, this could be especially dangerous when OAuth account linking was involved: an attacker could potentially cause their external OAuth identity to become associated with the victim's Immich account. Versions before **v1.132.0** were affected; the advisory identifies **v1.132.0+** as patched.

This directly validates the security requirement:

**SR-AUTH-02:** Immich shall generate and validate an unpredictable OAuth state value before accepting an OAuth authorization response or linking an external identity.

This is an excellent example for a misuse-case assignment because there is a direct chain:

**Misuser → forged OAuth transaction → account-linking misuse → state validation requirement → real Immich vulnerability → implemented correction.**

## 6. OIDC TLS certificate validation

Immich also published a 2026 advisory involving TLS validation for OIDC communications.

The affected implementation used allowInsecureRequests in a manner that disabled certificate verification for OIDC discovery, token, userinfo, and JWKS requests. That weakened the trust boundary between Immich and its Identity Provider.

This produces another clear requirement:

**SR-AUTH-03:** Immich shall validate TLS certificates when communicating with OIDC discovery, authorization, token, userinfo, and JWKS endpoints unless an administrator explicitly enables an insecure development configuration.

This matters because cryptographically verifying an OIDC token does not provide the intended security if an attacker can interfere with the mechanisms used to obtain trusted OIDC configuration or keys.

# 7. OAuth SSRF vulnerability

Another 2026 advisory concerned **Server-Side Request Forgery (SSRF)** through OAuth profile pictures.

The OAuth identity provider could return a picture URL in the user's profile. Immich previously fetched that URL server-side without sufficient URL/IP filtering. Consequently, a malicious value could potentially make the Immich server issue requests to unintended destinations. Versions through 1.129.0 were affected, and the advisory identifies v3.0.0 as patched.

That gives us:

**SR-AUTH-04:** Immich shall validate externally supplied URLs before performing server-side requests and shall prevent OAuth-controlled URLs from reaching prohibited internal or private network resources.

This requirement goes beyond authentication itself and protects the **Immich server/network trust boundary**.

# 8. Authorization and role management

OAuth can also influence authorization.

Immich supports a configurable **Role Claim**, with immich role as the documented default claim name. The expected role values are user or admin.

This creates a significant trust relationship:

**Identity Provider → role claim → Immich authorization**

Therefore:

**SR-AUTH-05:** Immich shall only derive administrative privileges from role claims obtained through a successfully authenticated and validated OIDC transaction.

And:

**SR-AUTH-06:** Immich shall restrict externally supplied role values to recognized Immich authorization roles.

Administrators must also protect role mappings on the Identity Provider because compromising that configuration could affect authorization inside Immich.

# 9. Session and logout security

Immich supports an OIDC **backchannel logout URL**:

/api/oauth/backchannel-logout

When supported by the Identity Provider, this provides a mechanism for communicating logout information to Immich.

A suitable requirement is therefore:

**SR-AUTH-07:** Immich shall provide mechanisms for terminating authenticated sessions when logout is initiated locally or communicated by a properly authenticated external Identity Provider.

The documentation would benefit from more explicit administrator-facing guidance about session lifetime, session invalidation, cookie protections, token revocation behavior, and the precise security guarantees of local versus IdP-initiated logout.

# 10. Deployment security responsibility

The security policy establishes a significant architectural boundary around Immich itself.

The following are explicitly treated as deployment/administrator responsibilities:

- PostgreSQL;
- Redis and surrounding services;
- reverse proxy configuration;
- network exposure;
- container-network isolation;
- installations outside the project's official container images.

Therefore, the security model can be represented as:

Internet

|

v

Reverse Proxy

|

v

Immich Server

/ \

/ \

PostgreSQL Machine Learning

|

Container Network

|

| OIDC

v

Identity Provider

+ MFA

+ Rate limiting

+ Authentication policy

Immich itself cannot guarantee the security of the complete operational environment. Deployment controls form part of the overall system security.

# 11. Documentation strengths and gaps

Based on the reviewed documentation and security history, the strongest aspects are the **documented vulnerability-reporting process, explicit security scope, OIDC configuration guidance, authorization-code authentication, configurable redirect URIs, signing-algorithm configuration, account-provisioning controls, role claims, and backchannel logout**.

The main documentation gaps are different. Security guidance is distributed among OAuth documentation, system settings, SECURITY.md, and advisories rather than consolidated into a deployment-hardening guide. The documentation could more explicitly cover:

## 1. recommended production TLS/reverse-proxy configuration;
## 2. secure session and cookie expectations;
## 3. OAuth account-linking threat considerations;
## 4. mandatory or strongly recommended MFA at the IdP;
## 5. brute-force/rate-limiting expectations;
## 6. IdP role-claim hardening;
## 7. private-network isolation requirements;
## 8. upgrade guidance tied to important security advisories.

These are documentation observations rather than claims that every corresponding control is absent from the implementation.

## Conclusion

The Immich documentation shows a security architecture in which **Immich protects application resources while delegating stronger identity controls to an external OAuth/OIDC provider and deployment-level controls to the administrator.** This separation of responsibility is explicit in the project's security policy.

The security-advisory history also demonstrates why misuse-case analysis is valuable. OAuth state validation, OIDC TLS verification, externally supplied URL validation, redirect sanitization, and authorization enforcement have all had concrete security relevance in Immich.

**Immich has meaningful security documentation and mature vulnerability-reporting infrastructure, but secure operation depends substantially on correct OAuth/IdP and deployment configuration. Its historical advisories show that authentication trust boundaries and externally controlled data should remain priority areas for security requirements and testing.**

[Immich Security Policy and Advisories](https://github.com/immich-app/immich/security)
[Immich OAuth/OIDC Documentation](https://docs.immich.app/administration/oauth/)
[Immich System Settings Documentation](https://docs.immich.app/administration/system-settings/)
[Immich OAuth Account-Hijacking Advisory](https://github.com/immich-app/immich/security/advisories/GHSA-3832-6r8h-9cfm)
[Immich OAuth SSRF Advisory](https://github.com/immich-app/immich/security/advisories/GHSA-hq46-gw2v-q86p)