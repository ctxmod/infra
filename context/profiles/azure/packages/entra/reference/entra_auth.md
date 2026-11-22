# Entra ID and ASP.NET Core auth facts

Verified Entra ID, Microsoft.Identity.Web (MIW) and ASP.NET Core
authentication and forwarded-header behavior on .NET 10.

- **Fresh-tenant identifier-URI policy.** A fresh Entra tenant's app policy
  rejects identifier URIs that lack the app id / tenant id / verified
  domain (`InvalidUniqueTenantIdentifierAsPerAppPolicy`). Exemption:
  `requested_access_token_version = 2` on the app's api block. v2 tokens
  then carry the URI as `aud`, so Microsoft.Identity.Web needs
  `AzureAd__Audience` set for that shape, and a stable `api://<prefix>-api`
  URI needs `AzureAd__Scope` published to the SPA (`api://<clientId>` is a
  Terraform self-reference — impossible). (Verified 2026-10-02.)
- **v2 access-token `aud` is ALWAYS the API client-ID GUID.** Per the
  Microsoft claims reference; MIW's `RegisterValidAudience` encodes
  v1 → `api://{ClientId}`, v2 → bare ClientId. Any "tokens carry the
  api:// scope URI as audience" diagnosis for a v2 registration is wrong on
  its face — the first round of a production 401 fix nearly shipped a no-op
  broadening justified by exactly that false premise.
- **The `(null)`-audience fingerprint means "no audience configured".**
  `www-authenticate: Bearer error="invalid_token", error_description="The
  audience '(null)' is invalid"` exists only in
  Microsoft.AspNetCore.Authentication.JwtBearer 8.0.29/10.0.0; MIW 4.15.0's
  delegate audience auto-population is inert on this path. A genuine
  aud=ClientId token 401ing this way proves the scheme validated with
  ValidAudience/ValidAudiences both null — an audience mismatch it is not.
  The shipped fix was REMOVAL of the `AzureAd__Audience` env override
  (Terraform app.tf), letting MIW default-select the client-ID audience:
  MIW 4.15.0 has no such binding key, the var is dead weight, and dead
  overrides mask this failure class. On any 401 with a delivered token,
  read the WWW-Authenticate error_description first and match the
  `(null)` fingerprint before theorizing about audiences. (Verified
  2026-10-04.)
- **Container App logs can be unreachable to a narrowly scoped
  principal.** `az containerapp logs show` needs
  `Microsoft.App/containerApps/getAuthToken/action` (a deployment SP may
  lack it) and `diagnostic-settings list` on the environment may be empty.
  Wire probes — curl plus WWW-Authenticate shapes — are then the working
  evidence route.
- **.NET 10 moved `ForwardedHeadersOptions`.** It now lives in
  `Microsoft.AspNetCore.Builder` (the ref pack's HttpOverrides.xml declares
  it there), while the `ForwardedHeaders` enum, `ForwardedHeadersDefaults`,
  `IPNetwork`, and the middleware stay in `Microsoft.AspNetCore.HttpOverrides`.
  Also renamed: `ForwardedHeadersOptions.KnownNetworks` →
  `KnownIPNetworks` (old name obsolete, ASPDEPR005). Failure signature: a
  file compiles with `using Microsoft.AspNetCore.HttpOverrides;` but
  CS0246 on `ForwardedHeadersOptions`. Web-SDK projects compile anyway via
  implicit usings; plain-SDK projects (e.g. xunit test projects using
  WebApplicationFactory) need BOTH an explicit
  `using Microsoft.AspNetCore.Builder;` AND `<FrameworkReference
  Include="Microsoft.AspNetCore.App" />` in the csproj — either half alone
  is insufficient. (Verified 2026-10-04 on .NET 10.0.12.)
- **Cleared proxy lists trust ANY peer.** Empirically proven on net10.0
  (hermetic fixture): `ForwardedHeadersMiddleware` with
  `KnownIPNetworks.Clear()` + `KnownProxies.Clear()` PROCESSES
  `X-Forwarded-For` from any direct peer, including a non-loopback
  TestServer peer — framework-source reading that suggests headers are
  ignored from unknown peers is wrong for v10. Semantics that ARE correct:
  rightmost-entry-only with `ForwardLimit = 1`; loopback peers process
  headers when the known lists are empty; a single-entry XFF selects the
  rate-limit bucket (PermitLimit=1: XFF 203.0.113.50 → 201, repeat → 429,
  headerless → the peer's own bucket). Residual risk shape: behind an
  ingress that APPENDS the client IP, per-client bucketing works for
  ingress-routed traffic; only a caller with direct network reach PAST the
  ingress can choose its bucket — on Azure Container Apps with external
  ingress and no IP restrictions that residual is publicly reachable in
  principle; record as Info, never claim "spoofing impossible". When
  validating forwarded-header trust, run the middleware directly in a
  fixture (construct `ForwardedHeadersMiddleware(next, loggerFactory,
  Options.Create(options))`, set `context.Connection.RemoteIpAddress`,
  invoke, read `RemoteIpAddress`) — do not derive trust semantics from
  source reading.
- **Hermetic MIW 4.x JWT fixture.** Four deltas make
  `AddMicrosoftIdentityWebApiAuthentication` run with real handler
  semantics against locally signed v2 JWTs (all proven in a hermetic
  xunit suite, 2026-10-04):
  1. AadIssuerValidator is installed as IssuerValidator and falls back to
     live OIDC metadata for the tenant — against a fake tenant it fails
     "The issuer '(null)' is invalid". Pin
     `TokenValidationParameters.ValidIssuers = [tenant-v2-issuer]`: the
     validator honors a ValidIssuers fast path and decides locally.
  2. Replace `ConfigurationManager` with
     `StaticConfigurationManager<OpenIdConnectConfiguration>` carrying the
     local RSA public key as `SigningKeys`, and set `IssuerSigningKey` to
     the private RSA key.
  3. Mutate `JwtBearerOptions` AFTER warming
     `IOptionsMonitor<JwtBearerOptions>.Get("Bearer")` — the named options
     instance is cached on first Get; post-Get mutation lands on the live
     instance the handler uses.
  4. `WebApplicationFactory.ConfigureWebHost` registrations resolve
     last-wins, but a factory-method local captured in the DI lambda is
     assigned AFTER the method returns (the caller's destructured copy
     stays null) — use a holder object whose property the lambda assigns
     (an `ApiHandle { Client, Store }` shape).
  MIW default gate, worth knowing: a token with NEITHER `scp` NOR `roles`
  is rejected 401 even when otherwise valid; but the VALUE of `scp` is
  never checked unless a required-scope policy is wired — any delegated
  token naming some scope passes. Keep the app delegates (audience/issuer
  validators) and disclose exactly these deltas.
