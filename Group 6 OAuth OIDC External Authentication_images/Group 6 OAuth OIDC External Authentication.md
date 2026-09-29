Group6 Assignment: IMMICH OAuth/OIDC/External Authentication

![Image: image_001](./Group%206%20OAuth%20OIDC%20External%20Authentication_images/image_001.png)

The diagram represents the **OAuth/OIDC External Authentication use case for Immich**. It shows how a user can sign in to Immich through an external Identity Provider (IdP), rather than having Immich directly verify the user's password.

**Actors**

**User (Web or Mobile Client):** The primary actor. The user wants to access their Immich account using external authentication.

**External Identity Provider (IdP):** The external system responsible for authenticating the user. Depending on an Immich deployment, this can be an OIDC-compatible identity provider. Its role is to verify the user's identity and return authentication information to Immich.

**Main use case: Authenticate with OAuth/OIDC**

The central use case is **Authenticate with OAuth/OIDC**. Instead of entering credentials that Immich itself validates, the user is redirected to the configured external identity provider.

The basic interaction is:

**User → Immich → Identity Provider → Immich → User**

The process represented by the diagram is:

# 1. Initiate External Login
   The user chooses the external/OAuth login option in Immich. Immich begins the authentication process.
# 2. Authenticate with OAuth/OIDC
   Immich sends an authorization request to the configured Identity Provider. The user authenticates with that provider.
# 3. External Identity Provider authenticates the user
   The IdP performs the actual authentication. For example, it might require a username/password, MFA, passkey, or another authentication method depending on how the IdP is configured.
# 4. Handle OAuth Callback
   After authentication, the IdP redirects the user back to Immich through the configured callback mechanism. Immich processes the returned authorization information.
# 5. Validate OIDC Identity
   Immich verifies the authentication information returned through the OIDC flow and obtains the identity information necessary to associate the authenticated person with an Immich user.
# 6. Create/Link User Account
   For a first-time login, Immich may need to associate the external identity with an Immich account. This is represented as an <<extend>> relationship because it is conditional rather than something that necessarily occurs during every login.
# 7. Establish Immich Session
   Once authentication succeeds and the identity is associated with a valid Immich user, Immich establishes the authenticated session. The user can then access resources permitted by their Immich account.

**<<include>> and <<extend>>**

The diagram uses two UML relationships.

**<<include>>** means that the referenced behavior is part of the authentication process. For example, handling the OAuth callback and validating the returned identity are required activities in completing the external authentication flow.

**<<extend>>** represents conditional behavior. **Create/Link User Account** is shown this way because account creation or linking is relevant under particular circumstances, such as an initial external-authentication interaction, rather than every successful login.

**Security significance**

From a security-requirements perspective, the important trust boundary is between Immich and the external Identity Provider. Immich relies on the IdP to authenticate the user correctly, while Immich must securely process and validate the information returned through the OAuth/OIDC flow.

This leads naturally to security requirements such as protecting the authorization flow against CSRF/state manipulation, redirect URI abuse, token theft, replay, account-linking errors, and improperly validated OIDC tokens or claims.

**In one sentence:** The use case describes how an Immich user authenticates through an external OAuth/OIDC identity provider, after which Immich validates the returned identity, associates it with an Immich account when necessary, and establishes an authenticated session.

==================================================================

For each use case identify the actor/interaction, create the use-case diagram, identify relevant misusers/misuse cases, add the security countermeasures, derive security requirements, and then check Immich's documentation/code to see how well it actually supports those requirements.

==================================================================

For the **Immich OAuth/OIDC External Authentication** use case, the security analysis can be developed by introducing misuse cases and then adding countermeasures that become explicit security requirements.

I checked the current [Immich repository](https://github.com/immich-app/immich), its OAuth documentation, authentication service code, and published security advisories. The analysis below therefore distinguishes between what we would **require by design** and what Immich **actually implements as of September 2026**.

## 1. Original use case

The normal use case is:

**User → Immich → External Identity Provider → Immich session**

The legitimate flow is approximately:

User selects OAuth login → Immich creates an authorization request → user authenticates at the IdP → IdP redirects to Immich → Immich validates the response → Immich identifies/creates/links the user → Immich creates a session.

Immich documents OIDC support and recommends an **Authorization Code** flow with a confidential web client. It supports providers such as Authentik, Authelia, Okta, Google, and Keycloak.

# 2. Relevant misusers

For this particular use case, five useful misusers are:

| **Misuser** | **Objective** |
| --- | --- |
| **Remote attacker** | Obtain unauthorized access to another user's Immich account |
| **Network/MITM attacker** | Intercept or manipulate communication between Immich and the IdP |
| **Malicious OAuth/OIDC client/user** | Manipulate authorization callbacks, state, authorization codes, or redirect parameters |
| **Malicious/compromised Identity Provider** | Supply fraudulent identity or privilege claims |
| **Authenticated malicious user** | Link an external identity to an account improperly or exploit externally supplied profile data |

These actors give us several concrete misuse cases.

# 3. Misuse Case 1 — OAuth Login CSRF / Callback Manipulation

### Misuse case

**Attacker manipulates the OAuth callback**

The attacker attempts to submit a callback generated for a different authentication transaction or otherwise tricks the victim into completing an OAuth flow associated with the attacker.

Conceptually:

Attacker

|

v

[Manipulate OAuth Callback]

|

| threatens

v

[Authenticate with OAuth/OIDC]

The main problem is that Immich needs a reliable way to associate the callback with the login operation that originally generated it.

### Countermeasure

**Validate OAuth state**

Immich should generate an unpredictable state value when authentication starts and verify that the returned value matches the expected state.

### Derived security requirement — SR-1

**SR-1:** The Immich authentication system shall generate and validate an unpredictable OAuth state value for each external authentication transaction and shall reject callbacks for which the expected state is missing or invalid.

### Does Immich support it?

**Yes.**

The current AuthService.callback() retrieves the expected state and explicitly rejects a callback when it is absent:

const expectedState = dto.state ?? this.getCookieOauthState(headers);

if (!expectedState?.length) {

throw new BadRequestException('OAuth state is missing');

}

The expected state is subsequently passed into the OAuth repository for validation.

**Assessment: Supported.**

# 4. Misuse Case 2 — Authorization Code Interception

### Misuse case

**Attacker steals an OAuth authorization code**

Suppose an attacker obtains the authorization code returned by the Identity Provider. The attacker attempts to exchange that code for tokens and impersonate the legitimate user.

Attacker

|

v

[Steal Authorization Code]

|

| threatens

v

[Authenticate with OAuth/OIDC]

### Countermeasure

**PKCE — Proof Key for Code Exchange**

Immich can create a code\_verifier and derive a code\_challenge. The challenge participates in the authorization request, while the verifier is required during the code exchange.

Possession of the intercepted authorization code alone therefore should not be enough to complete authentication.

### Derived security requirement — SR-2

**SR-2:** The Immich authentication system shall use PKCE for OAuth/OIDC authorization-code authentication and shall reject the authentication transaction if the required PKCE code verifier is missing or invalid.

### Does Immich support it?

**Yes in the current authentication design.**

The callback explicitly requires the code verifier:

const codeVerifier =

dto.codeVerifier ??

this.getCookieCodeVerifier(headers);

if (!codeVerifier?.length) {

throw new BadRequestException(

'OAuth code verifier is missing'

);

}

The verifier is then passed into the OAuth processing code.

There is useful historical evidence here too: a 2025 compatibility regression caused some IdPs to reject Immich's PKCE exchange because the verifier was reportedly missing from the token request. That issue illustrates exactly why this requirement matters.

**Assessment: Supported in the current design.**

# 5. Misuse Case 3 — Forged or Manipulated OIDC Identity

### Misuse case

**Attacker supplies a fraudulent identity token/profile**

An attacker attempts to convince Immich that they are another user by manipulating identity information such as:

sub

email

role

profile

This is particularly serious because Immich uses OIDC identity information to map the external user to a local account.

Attacker

|

v

[Forge OIDC Identity]

|

| threatens

v

[Validate OIDC Identity]

### Countermeasures

The application should:

- validate the OIDC response;
- validate the issuer;
- validate token signatures;
- use trusted signing keys;
- validate the authorization transaction;
- avoid trusting arbitrary identity claims.

### Derived security requirement — SR-3

**SR-3:** The Immich authentication system shall cryptographically validate identity information received from the configured OIDC provider before using that information to identify, create, link, or authorize an Immich user.

### Does Immich support it?

**Largely yes, with an important historical vulnerability.**

Immich uses an OIDC client library for discovery and authorization-code processing and allows configuration of the ID-token signing algorithm. The documentation currently lists RS256 as the default ID-token signing algorithm.

However, Immich disclosed a significant 2026 OIDC vulnerability. Earlier versions unconditionally enabled allowInsecureRequests, weakening TLS certificate verification for OIDC discovery/token/userinfo/JWKS communication. That could undermine the trust relationship with the IdP.

The advisory identifies **v3.0.0 as the patched version**. Current configuration examples also show:

allowInsecureRequests: false

``` :chatgpt-content-reference{index="7"}

\*\*Assessment: Supported in current patched releases, but historically deficient. Deployments should use v3.0.0 or later for this particular fix.\*\*

---

# 6. Misuse Case 4 — Account Linking / Account Takeover

This is one of the most interesting misuse cases in Immich.

### Misuse case

\*\*Attacker causes an external identity to be linked to another user's Immich account.\*\*

Immich first attempts to locate the user using the OAuth subject:

```text

profile.sub

If that fails, the current authentication service can normalize the supplied email address and look for an existing local account with that email. If found and not already OAuth-linked, it associates that user with the OAuth subject.

Conceptually:

Attacker

|

v

[Manipulate Account Linking]

|

| threatens

v

[Create/Link User Account]

### Countermeasures

Account linking should only occur after a trusted OIDC identity has been validated. Existing OAuth associations must not silently be overwritten.

### Derived security requirement — SR-4

**SR-4:** The Immich authentication system shall link an external OIDC identity to a local Immich account only after successful validation of the external identity and shall prevent an existing OAuth identity association from being silently replaced by another identity.

### Does Immich support it?

**Partially to substantially supported.**

The current code explicitly checks for an existing OAuth association. If an account with the matching email already has an oauthId, Immich rejects the authentication rather than replacing it:

if (emailUser.oauthId) {

...

throw new BadRequestException(

'OAuth authentication failed'

);

}

``` :chatgpt-content-reference{index="9"}

However, because email-based account linking has security consequences, the strength of this mechanism ultimately depends on the integrity of the OIDC identity information and the configured IdP.

\*\*Assessment: Supported, with dependency on trusted IdP configuration and claim integrity.\*\*

---

# 7. Misuse Case 5 — Unauthorized Account Creation

### Misuse case

An attacker authenticates through the configured Identity Provider and obtains an Immich account even though an administrator did not intend to provision that person.

```text

Unauthorized User

|

v

[Automatically Create Account]

|

| threatens

v

[Create User Account]

### Countermeasure

**Configurable automatic registration**

Administrators should be able to prevent external identities from automatically becoming Immich users.

### Derived security requirement — SR-5

**SR-5:** The Immich authentication system shall provide administrators with the ability to disable automatic creation of local accounts from externally authenticated identities.

### Does Immich support it?

**Yes.**

Immich provides the Auto Register OAuth setting. Its documented default is currently true.

The server enforces it:

if (!user) {

if (!autoRegister) {

...

throw new BadRequestException(

'OAuth authentication failed'

);

}

}

``` :chatgpt-content-reference{index="11"}

Thus an administrator can disable automatic OAuth provisioning.

\*\*Assessment: Supported.\*\*

---

# 8. Misuse Case 6 — Privilege Escalation Through Role Claims

### Misuse case

An attacker attempts to manipulate an externally supplied role claim so that Immich treats the account as an administrator.

Immich supports an OAuth role claim, documented by default as:

```text

immich\_role

The expected values include user and admin.

Therefore:

Attacker / Compromised IdP

|

v

[Inject Admin Role Claim]

|

| threatens

v

[Authorize User]

### Countermeasure

Only role information from the configured, successfully validated IdP should affect Immich authorization.

### Derived security requirement — SR-6

**SR-6:** The Immich authentication system shall accept authorization-role claims only from a successfully validated OIDC identity and shall restrict accepted role values to recognized Immich roles.

### Does Immich support it?

**Supported, but the IdP becomes security-critical.**

The authentication service obtains the configured role claim and determines whether its value corresponds to admin; it can update the local user's administrative status based on that claim.

This means compromise or serious misconfiguration of the IdP's role mapping could propagate into Immich authorization.

**Assessment: Supported technically; security depends heavily on IdP claim governance.**

# 9. Misuse Case 7 — Man-in-the-Middle Against Immich ↔ IdP

### Misuse case

A network attacker intercepts communication between Immich and the Identity Provider and attempts to impersonate the provider.

Network Attacker

|

v

[Impersonate Identity Provider]

|

| threatens

v

[OIDC Authentication]

This is not hypothetical for Immich. A 2026 security advisory reported that OIDC discovery, token, userinfo and JWKS requests could operate with certificate verification disabled. The advisory describes the possibility of manipulating OIDC responses and potentially obtaining unauthorized sessions.

### Countermeasure

**Strict TLS certificate validation**

### Derived security requirement — SR-7

**SR-7:** The Immich authentication system shall validate TLS certificates for OIDC discovery, token, userinfo, and JWKS communications by default and shall not permit insecure certificate validation without an explicit administrative configuration.

### Does Immich support it?

**Current releases: yes; older affected versions: no.**

The vulnerability was formally disclosed in July 2026, and the advisory identifies **v3.0.0 as patched**.

This provides particularly strong traceability for your assignment:

**Misuse case → discovered real vulnerability → countermeasure → security requirement → implementation correction.**

# 10. Misuse Case 8 — SSRF Through OAuth Profile Picture

Another particularly useful case comes directly from Immich's security history.

### Misuse case

A malicious or compromised identity source supplies a specially crafted profile-picture URL. Immich's server then fetches that URL, potentially causing requests to internal network resources.

Malicious Identity

|

v

[Supply Malicious Picture URL]

|

v

[Immich Fetches URL]

|

v

[Access Internal Resource]

### Countermeasure

Validate externally supplied URLs before the Immich server fetches them, particularly blocking unsafe internal/private destinations.

### Derived security requirement — SR-8

**SR-8:** The Immich server shall validate externally supplied OAuth profile-resource URLs before initiating server-side network requests and shall prevent those URLs from being used to access prohibited internal resources.

### Does Immich support it?

A published Immich advisory confirms that the OAuth profile-picture synchronization feature previously had an **SSRF vulnerability** because the profile picture URL could be fetched server-side without sufficient URL/IP validation. The advisory lists versions through **1.129.0 as affected** and **v3.0.0 as patched**.

**Assessment: Historically not supported adequately; patched in v3.0.0.**

# 11. Security requirements traceability summary

For your assignment, I would present the final traceability this way:

| **ID** | **Misuse Case** | **Security Countermeasure** | **Derived Requirement** | **Current Immich Evidence** |
| --- | --- | --- | --- | --- |
| **SR-1** | OAuth callback/CSRF manipulation | OAuth state validation | System shall validate OAuth state | **Supported** — missing state rejected. |
| **SR-2** | Authorization-code theft | PKCE | System shall require and validate PKCE verifier | **Supported** — callback requires code verifier. |
| **SR-3** | Forged OIDC identity | Token/identity validation | System shall cryptographically validate OIDC identity | **Supported in patched releases**; TLS weakness existed historically. |
| **SR-4** | Account-link takeover | Controlled account linking | Existing OAuth association shall not be silently replaced | **Supported** by conflict check. |
| **SR-5** | Unauthorized account creation | Disable auto-registration | Administrator shall control external account provisioning | **Supported** via Auto Register. |
| **SR-6** | Role/privilege manipulation | Validated role claims | Only trusted, recognized OIDC role claims shall affect privileges | **Supported, IdP-dependent.** |
| **SR-7** | IdP MITM | TLS certificate verification | System shall validate TLS for OIDC communications | **Patched in v3.0.0** after a real vulnerability. |
| **SR-8** | OAuth profile-picture SSRF | URL/network destination validation | Server shall reject unsafe externally supplied resource URLs | **Patched in v3.0.0** after a real vulnerability. |

## Overall finding

Immich's current OAuth/OIDC architecture provides several meaningful controls that correspond directly to the misuse cases**: state validation, PKCE, OIDC identity processing, configurable automatic registration, protection against replacing an already-linked OAuth identity, role-claim handling, and back-channel logout support.** Its documentation also explicitly expects the Authorization Code flow.

The security-history check is important, however. Immich had at least two directly relevant OAuth/OIDC weaknesses disclosed in 2026**: improper TLS certificate validation in OIDC communications** and **SSRF through an OAuth profile-picture URL**. Both advisories identify **v3.0.0 as the patched release**, so these are excellent examples of misuse-case analysis predicting security requirements that proved necessary in the real implementation.

One architectural point is also worth documenting: Immich's security policy says that login-hardening controls such as **MFA and rate limiting are expected to be delegated to the OAuth server.** Therefore, those should be modeled as security requirements allocated to the **external Identity Provider/enabling system,** rather than assumed to be controls implemented internally by Immich.

Immich OAuth/OIDC documentation : <https://docs.immich.app/administration/oauth/>?
Immich authentication service source code https://github.com/immich-app/immich/blob/main/server%2Fsrc%2Fservices%2Fauth.service.ts?