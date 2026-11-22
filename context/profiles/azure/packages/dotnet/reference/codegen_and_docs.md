# .NET codegen and Microsoft docs routes

Verified OpenAPI/.NET code-generation tooling, .NET 10 SDK shape, and cheap
retrieval routes to Microsoft documentation.

- **hidi converts OpenAPI 3 to Swagger 2.0, first-party.**
  `dotnet hidi transform -d spec.yaml -v 2.0 -f json --co -o out.json`
  (Microsoft.OpenApi.Hidi 3.10.2) keeps `x-ms-summary`, `x-ms-visibility`,
  `x-ms-connector-metadata`, and maps oauth2 authorizationCode →
  `flow: accessCode`. `api-spec-converter` is stale (2021). hidi targets
  the .NET 8 runtime: on a .NET 10-only box set `"rollForward": true` on
  its tool-manifest entry. Its YAML parser rejects unquoted
  `[api://x/y]` flow scalars — quote them. (Verified 2026-10-01.)
- **NSwag builds controllers on .NET 10.** NSwag 14.7.1
  `openapi2cscontroller /ControllerStyle:Abstract` builds and runs on
  net10.0 via NSwag.MSBuild `$(NSwagExe_Net100)`.
- **.NET 10 SDK shape.** `dotnet new tool-manifest` writes
  `dotnet-tools.json` at the cwd, not `.config/`; `dotnet new sln` makes
  `.slnx`. On a host without dotnet, `dotnet-install.sh --channel
  10.0 --install-dir /tmp/...` works; where `npx` breaks through a host
  shim, call `node node_modules/<pkg>/bin/...` instead.
- **Cheap routes to Microsoft Learn content** (verified 2026-10-01; Learn
  pages cost 10-15K chars through a web-page reader):
  - Power Platform docs (ALM, pac CLI reference, admin): raw markdown at
    `https://raw.githubusercontent.com/MicrosoftDocs/power-platform/main/power-platform/<path>.md`
    — e.g. `developer/cli/reference/solution.md`, `.../includes/<cmd>-remarks.md`,
    `admin/pay-as-you-go-set-up.md`. Grep the flags directly.
  - Terraform power-platform provider docs:
    `raw.githubusercontent.com/microsoft/terraform-provider-power-platform/main/docs/resources/<name>.md`
    — the "Known Limitations" line sits in the frontmatter.
  - Copilot Studio docs: no public raw repo (MicrosoftDocs/copilot-studio-docs
    404s). `curl -A 'Mozilla/5.0'` the Learn URL, strip tags in python,
    regex for sentences around a keyword; a web-page reader with a CSS
    selector also works for one section.
  - GitHub API JSON from curl often contains raw control chars — parse
    with `json.load(f, strict=False)` from a file, not a pipe.
  - PyPI JSON (`pypi.org/pypi/<pkg>/json`) gives version, upload date and
    `requires_dist` for dependency-weight comparisons.
