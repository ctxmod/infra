# Infra

Platform infrastructure doctrine for ctxmod (`github.com/ctxmod/infra/context`). Requires `github.com/ctxmod/engineering/context`.

Profiles:

- `core` (default): `delivery` (CI workflows, deployment profiles, GitOps) and `security` (supply-chain signing, workload identity); opt-in package `gcp`.
- `kubernetes` (package `platform`) and `terraform` (package `practice`); each extends `core`.
- `azure`: verified Azure facts — `terraform_azapi`, `services`, `entra`, `power_platform`, `dotnet`; extends `core`.
- `github_actions`: GitHub Actions workflow hardening and verified GitHub Actions facts (package `workflows`); extends `core`.
- `shell`: tmux 3.4 platform facts (`tmux`) and the generated-zsh rule (`zsh`).

Select a profile by its qualified name, e.g. `github.com/ctxmod/infra/context/profiles/kubernetes`.
