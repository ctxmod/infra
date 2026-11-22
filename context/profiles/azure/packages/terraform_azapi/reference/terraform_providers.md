# Terraform on Azure: azapi and azurerm facts

Verified azapi and azurerm provider behavior from provisioning Azure
workloads (2026-10 to 2026-11). Each fact cost a failed apply, a failed run,
or a refuted fix; read this before re-deriving a provider behavior.

**Version currency.** The facts below were verified on azapi 2.x and azurerm
5.12. As of 2026-11-22 the latest releases are azapi 2.14.0, azurerm 5.14.0
and azuread 3.10.0; a fact pinned to a provider behavior (body merge shapes,
`ignore_changes` interplay, attribute keys) is re-verified after crossing a
minor. **Naming.** Microsoft Entra ID is the product name, but Terraform
spellings keep the old one: the provider is still `hashicorp/azuread` with
`azuread_*` resources, and the azurerm backend's Entra data-plane switch is
still `use_azuread_auth` (env `ARM_USE_AZUREAD`), which its docs now
recommend over access keys and SAS tokens. Search for the old spelling when
looking up configuration.

- **azapi 2.x body shapes.** `body` / `sensitive_body` take raw HCL objects;
  `jsonencode(...)` is a validation error. `sensitive_body` is merge-patched
  over `body` (sensitive wins) and is never stored in state. Array-valued
  merges must nest at the exact schema path — the AI Search index vectorizer
  lives under `vectorSearch.vectorizers`, not top level. (Verified
  2026-10-02.)
- **`azapi_update_resource` strips Container App inline secrets.** The
  resource does GET then PUT, and GET never returns secret values, so on any
  app holding an inline secret it fails `400 ContainerAppSecretInvalid`
  ("value or keyVaultUrl and identity should be provided") — proven
  2026-11-03 on an app with an inline `database-url` secret (an app with no
  inline secret never hits this). The working shape is
  `azapi_resource_action` with `method = "PATCH"` and the same body: a true
  merge-patch, no revision restart, secrets untouched, idempotent plan.
- **Container Apps custom-domain TLS binding needs the azapi patch.**
  `azurerm_container_app_custom_domain` creates the domain with
  `bindingType = Disabled`, and its `lifecycle { ignore_changes =
  [certificate_binding_type, container_app_environment_certificate_id] }`
  pins that Disabled value — the azurerm resource pair can never complete
  the binding, and re-running the apply does not bind the cert (the asuid
  TXT validation and the managed certificate both go green on their own).
  The working path is a count-gated `azapi_update_resource "domain_binding"`
  PATCHing `ingress.customDomains` to `bindingType = "SniEnabled"` with the
  managed-cert id (2026-10-04; domain verified live HTTP/2 200 afterward). Never predict azurerm-side binding
  here; the `ignore_changes` block is load-bearing — it keeps the
  out-of-band issued cert from being churned but freezes Disabled if the
  patch is absent. Any hostname or cert-routing change goes through the
  azapi patch, not the azurerm resources.
- **`count` migrations via `state mv`.** Adding `count` to a count-less
  resource changes every instance address — run `terraform state mv <addr>
  <addr>[0]` for each BEFORE the first apply (9 migrations in one root
  module, 2026-10-04). zsh gotcha: quote the target
  as `"${addr}[0]"` or the subscript eats the bracket.
- **Conditional branches both evaluate.** Never index a count-gated
  resource with `[0]` inside a conditional or template. Use
  `one(concat(res[*].attr, data[*].attr))` per ATTRIBUTE; whole-object
  splats propagate sensitivity (the search service `primary_query_key`
  marked an endpoint output sensitive and failed the plan), and `coalesce`
  swallows the null side in templates.
- **azurerm 5.12 attributes** (verified 2026-11-03): `azurerm_cognitive_account_project.endpoints`
  uses the key `"AI Foundry API"`, not `"Project"`. `project_management_enabled
  = true` requires an `identity` block on the account. A container app
  environment with `log_analytics_workspace_id` needs
  `logs_destination = "log-analytics"`. Container app probes use `timeout`
  and `failure_count_threshold`.
- **Backend grants are container-scoped.** The azurerm backend init lists
  blobs at CONTAINER level ("Failed to get existing workspaces: listing
  blobs"), so a blob-scoped Storage grant cannot initialize any backend —
  grant terraform-state identities at container scope. Keep
  `use_azuread_auth = true` in the committed backend block and pass
  `use_oidc=true` plus resource group/account via `-backend-config`; local
  OIDC credentials do not exist, so the fixed form breaks the bootstrap's
  local apply.
- **`ARM_*` env override trap.** Sourcing a `.env.local` that carries
  `ARM_*` overrides az-CLI auth for every later terraform run in that
  shell: an identical 403 then looks like a missing grant but is the WRONG
  IDENTITY (an SP with narrower scopes). Unset `ARM_*` for user-identity
  runs.
- **Container app environment drift.** `azurerm_container_app_environment`
  on Consumption self-provisions a workload profile named "Consumption" —
  declare it (and `workload_profile_name` on the app) or the second plan
  drifts. `zone_redundancy_enabled` requires `infrastructure_subnet_id`.
- **Storage containers are data-plane.** `azurerm_storage_container` needs
  the account key; with `shared_access_key_enabled = false`, create
  containers via `azapi_resource` on `.../blobServices/default` instead.

## Local CI runs (act)

Verified 2026-10-04 with act v0.2.89: a headless first run dies on the
interactive image picker — pass `-P
ubuntu-latest=catthehacker/ubuntu:act-latest`; the medium image has no az
CLI (bootstrap `aka.ms/InstallAzureCLIDeb` in the step); `github.repository`
derives from the git REMOTE, so a local-path clone yields a garbage image
tag — point origin at the real repo; `env.ACT == 'true'` guards work in
step-if while `!github.event.act` (with `-e event.json` carrying
`{"act": true}`) works in job-if; buildx works through
`--container-daemon-socket /var/run/docker.sock`. Bot Service Direct Line keys: `az bot directline show` omits sites, and `listChannelWithKeys` via
arm.botframework.com demands a client certificate — read the key from the
Terraform state blob instead.

## Sources

- https://github.com/Azure/terraform-provider-azapi/releases — azapi 2.14.0 (accessed 2026-11-22 via release tags)
- https://github.com/hashicorp/terraform-provider-azurerm/releases — azurerm 5.14.0 (accessed 2026-11-22 via release tags)
- https://github.com/hashicorp/terraform-provider-azuread/releases — azuread 3.10.0 (accessed 2026-11-22 via release tags)
- https://developer.hashicorp.com/terraform/language/backend/azurerm — `use_azuread_auth` / `ARM_USE_AZUREAD`, Entra ID recommended for data-plane auth (accessed 2026-11-22)
