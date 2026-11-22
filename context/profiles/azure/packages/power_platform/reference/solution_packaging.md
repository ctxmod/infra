# Power Platform solution packaging facts

Verified Power Platform custom connector, solution packaging and import
behavior (pac CLI, SolutionPackager, Dataverse Web API).

- **Custom connector = type 372** (`ECConnector`; 371 is `Connector`).
  File: `Connectors/<PrimaryName>.xml`, root element `<Connector>`,
  PrimaryName from the child element `<Name>`; composes into
  customizations.xml's `<Connectors>`. Verified 2026-10-03 by decompiling
  pac 2.12.2's `SolutionPackagerLib.dll` plus a pack/unpack round trip.
- **Two path forms, inverse mappings.** `Connector/<rest>` (singular,
  flat) is the IN-ZIP form; `Connectors/<PrimaryName>/**` (plural,
  folder-style) is the AUTHORING form that pack consumes and unpack
  writes. Non-.xml files under `Connectors/` whose relative path starts
  with `Connectors/<PrimaryName>` (prefix match, not boundary-aware) ride
  the zip at `Connector/<rest>`. Code that globs `Connector/*.json` inside
  an unpacked tree finds nothing (a real CI job's exactly-one-swagger
  assertion fired at 0 against a perfectly swagger-shaped packed artifact).
  Diagnose connector-path bugs by asking which of the two forms a path
  belongs to; a swap that must survive import happens BEFORE pack on the
  source tree, not after unpack.
- **Packed connector form.** A flat `Connector/` composite with
  `<name>_openapidefinition.json`, `_connectionparameters.json` (`{}` ok),
  `_policytemplateinstances.json` (`[]` ok), `_iconblob.png`, and the
  record in customizations.xml `<Connectors>` with lowercase fields +
  connectorid + file pointers; RootComponent type 372 carries
  `id="{guid}"` and the same schemaName. Ground truth:
  microsoft/CopilotStudioSamples OrderManagementMCPDemo solution. A
  folder-style composite (paconn swagger + apiProperties under
  `Connectors/<name>/`) packs but NREs at import.
- **The connector root-component warning is cosmetic.** "Following root
  components are not defined in customizations" is structurally cosmetic
  for connectors: Solution.xml type 372 keys the component as
  `ECConnector-<name>` while the folder processor registers it as type 371
  (`Connector-<name>`), so the cross-check can never match. The packer only
  hard-fails on the reverse condition (customizations content missing from
  root components).
- **bots and botcomponents sit in the classic packer's
  ignoreComponentsSet** — agent XML has no place in the classic layout;
  author agents as docs/build spec and pin them at the first real
  export/unpack.
- **Environment variables.** `EnvironmentVariables/<schemaname>/` definition
  XML composes into `<EnvironmentVariables>`; unpack rewrites to a flat
  `<schemaname>.xml` — both forms pack identically. In solutions, env vars
  unpack to lowercase
  `environmentvariabledefinitions/<schemaname>/environmentvariabledefinition.xml`
  (lowercase element names); the folded CamelCase `<Definition>` form is
  rejected pre-job.
- **Import prerequisites.** A hand-authored solution needs `version` AND
  `SolutionPackageVersion="9.2"` on the ImportExportXml root — without
  them the importer reports "Package Version: On-Premises" (missing
  version) or a masked 0x80040216. Connector schema names must be
  `<publisherprefix>_<Name>` (validation: "prefix with length between 2~8
  followed by '_' and alphanumeric name").
- **The swagger host is validated AT IMPORT.** The connector service
  validates the host and does NOT resolve `@environmentVariables(...)`
  there — deployment settings or a pre-created variable do not help. The
  connector must import with a literal host; the working pattern is a
  CI-only jq host-swap before pack.
- **Import debugging via importjob polling.** Dataverse Web API
  ImportSolution needs `ImportJobId` (a caller GUID); the operation has no
  Operation-Location header — poll
  `GET /importjobs(<id>)?$select=data,completedon`. `data` holds the XML
  result with the real per-component error text, which pac masks as "An
  unexpected error occurred". This is the fastest reliable debug loop.
- **Reference-import trick.** Importing Microsoft's own emptySolution zip
  (powerplatform-actions src/test/data, LFS via media.githubusercontent.com)
  splits caller/environment guilt from package guilt in one dispatch.
- **Local tooling caveats.** Environment-scoped GitHub variables are NOT
  visible to `jobs.<id>.if` (steps see them fine) — a job-level gate needs
  a repository-level mirror. A /tmp dotnet harness and a local pac shim
  no longer ran current pac builds (2026-10-08: bolt aborts,
  FileNotFoundException for Microsoft.Extensions.DependencyInjection
  10.0.0.0) — the decompile-backed form mapping above is the offline
  fallback. pac noise: a trailing `xdg-open: file 'dotnet' does not exist`
  after "Packed Solution." is harmless (exit 0); with a user-local .NET
  install, pac needs `DOTNET_ROOT=... PATH=<dotnet tools dir>:...` per
  invocation.
