# Azure service facts

Verified Azure service behavior (AI Search, PostgreSQL Flexible Server,
Azure OpenAI / Foundry models, roles, registries). Quota and price facts carry
their verification date and may drift — re-probe the usages API if an apply
fails on capacity.

- **AI Search RBAC auth is off by default.** `azurerm_search_service`
  defaults to `authOptions: apiKeyOnly`; data-plane RBAC stays dead until
  `authentication_failure_mode = "http401WithBearerChallenge"` is set
  (→ `aadOrApiKey`). Symptom: 403 with an empty body on
  `https://<svc>.search.windows.net/indexes(...)` despite valid role
  assignments. AI Search Free DOES honor Entra data-plane RBAC once the
  mode is set — the "try-for-free has no RBAC" doc claim is wrong.
  (Verified 2026-10-02.)
- **AI Search roles.** `Search Service Contributor` has NO dataActions —
  it covers index definitions only, via management-plane access. Document
  push needs `Search Index Data Contributor` (dataActions
  `indexes/documents/*`).
- **Data-plane token resource is `https://search.azure.com`.** A
  per-service `*.search.windows.net` token resource fails with
  AADSTS500011.
- **`retrievable` flips in place.** Microsoft Learn lists "Set the
  retrievable attribute on an existing field" among updates applied with no
  rebuild, and the Search REST reference says `retrievable` "can be changed
  on existing fields". `azapi_data_plane_resource.body` is optional +
  computed (not ForceNew), so the flip plans in place and an ordinary apply
  picks it up — no taint, no recreate, no reindex, no data loss. A
  recreate-based determination on this attribute was wrong and survived two
  independent reviews before a primary-source check caught it (erratum
  2026-10-09). General lesson, verified against the worked case: when a
  determination hinges on a cloud service rejecting an in-place attribute
  change, verify THAT attribute against the service's primary
  documentation — general field-immutability classes carry per-attribute
  exceptions.
- **PostgreSQL Flexible B1ms fails in swedencentral** (verified
  2026-11-03): `SkuNotAvailable` at create, even though `list-skus` offers
  B1ms there and the capabilities API shows `restricted: Disabled`.
  northeurope works. A failed create keeps the server name bound to its
  region, so a retry elsewhere returns 409 `InvalidResourceLocation` — use
  a new name. The azurerm SKU string is `B_Standard_B1ms`; the database
  resource is `azurerm_postgresql_flexible_server_database`.
- **Quota observed on one pay-as-you-go subscription** (usages API
  2023-05-01, verified 2026-10-02): `OpenAI.DataZoneStandard.gpt4.1-mini` limit **0** in
  swedencentral AND westeurope (subscription-wide);
  `OpenAI.Standard.gpt4.1-mini` 200K in swedencentral;
  `OpenAI.DataZoneStandard.text-embedding-3-small` 1000. Embeddings
  inference needs `api-version=2024-10-21` (2024-10-01 → 404). By
  2026-11-03 the regional gpt-4.1-mini Standard quota was nearly exhausted
  by another workload and the working choice was DataZoneStandard
  (quota 2000) — the DataZone numbers moved between the two dates, so
  re-probe before relying on either.
- **GPT-6 rejects `temperature=0`** (verified 2026-11-03 on gpt-6-luna,
  2026-09-22): HTTP 400, "Only the default (1) value is supported" — GPT-6
  models are reasoning models, so drop the parameter from the request.
  Strict `json_schema` structured output works unchanged on Chat
  Completions. The first live eval failed at 1/10 on this alone.
- **"Short context" is a billing category, not a model variant.** For
  GPT-5.6 and GPT-6 Standard pay-as-you-go, the input tokens of each
  request decide whether it bills as short- or long-context; every model
  handles both, and nothing in the deployment or the request selects one.
  A ~500-token prompt is always short-context. Do not hunt for a "short
  context" SKU.
- **GPT-6 pricing beats gpt-4.1-mini on a small eval workload.** Retail,
  swedencentral, DataZone, USD per million tokens (2026-11-03):
  gpt-6-luna short-context 0.12 in / 0.60 out; gpt-4.1-mini 0.44 in /
  1.76 out. gpt-6-luna emits ~1.7x the output tokens (reasoning tokens)
  yet still costs less per eval run: ~0.0017 USD vs ~0.0041. Monitor
  metrics on the cognitive account expose call counts only, never token
  counts — read `usage` from a direct SDK call.
- **Role display names move.** The built-in role "Azure AI User" now
  displays as "Foundry User" (verified 2026-11-03).
- **AcrPush is not enough for `az acr login`** — it lacks registry read;
  also grant Reader on the registry.
- **Async Foundry clients need aiohttp.** `azure.identity.aio` /
  `azure-ai-projects.aio` fail with ImportError at construction unless
  `aiohttp` is installed.
- **arm64 image builds.** On an arm64 build host, an emulated amd64 `npm ci`
  hangs — build the SPA stage with `FROM --platform=$BUILDPLATFORM`.
