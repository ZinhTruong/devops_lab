# DevOps Lab

This repository collects the Kubernetes manifests, Argo CD bootstrap configuration, and supporting overlays that operate the `instavote` blue/green + canary rollout example alongside the monitoring/ingress stacks you use for the lab.

## Repository layout

- `app/` – application artifacts for the `vote` service built as Argo Rollouts objects (base manifests plus staging/production overrides). These overlays are what the Argo CD environment apps actually deploy.
- `argocd/` – Argo CD configuration, split into:
  - `bootstrap/` – the root App-of-Apps manifests (`root-dev`, `root-stg`, `root-prod`) that live in the `argocd` namespace and point at `argocd/env/<env>`.
  - `env/<env>/` – self-contained bundles per environment. Each folder contains an `AppProject` definition, the env-specific rollout `Application`, and a kustomization that stitches them together for the corresponding bootstrap root.
- `ingress-nginx/`, `kube-prometheus-stack/`, `ELK/` – other infrastructure stacks managed in the same repo (presumably their own kustomizations, outside the app/argocd flow). They are built/managed separately as needed.

## Argo CD App-of-Apps workflow

1. Root apps in `argocd/bootstrap/` sync `argocd/env/<env>` for each environment (`dev`, `stg`, `prod`).
2. Each `argocd/env/<env>` directory builds an `AppProject` and one or more child `Application`s (currently `insta-stg` and `instavote-prod`) that point to `app/overlays/<env>`.
3. The rollout overlays supply the `Rollout`, `Service`, `PreviewService`, and (prod only) `Ingress` resources and patch them as needed.
4. Projects use sourceRepo whitelists and destination restrictions (namespaces, namespaces) so RBAC stays scoped to each cluster environment.

## Working with the repo

- To preview what Argo CD will apply: `kustomize build argocd/env/stg` (or `argocd/env/prod`) will emit the AppProject/Application pairs for that env.
- The actual workload manifests live under `app/base/` and `app/overlays/<env>/`; patch or extend those to change the deployed Rollout/Services.
- After you edit anything under `argocd/env/<env>`, sync the corresponding bootstrap app (`argocd app sync root-stg` / `root-prod`) so Argo CD refreshes the child apps.
- `argocd/app-of-app/root-*.yaml` exist for backwards compatibility, but the new bootstrap is under `argocd/bootstrap/` now.

## Teams & RBAC

- The `insta` project in staging allows every repo (`sourceRepos: ['*']`) and targets the `staging` namespace for the `insta-stg` app.
- The `instavote` project in production restricts source repos to this GitHub repo (both SSH and HTTPS forms) and only allows the `prod` namespace/destination. Role definitions (optional) should map to your `instavote-admins`/`instavote-viewers` groups if you enable Argo CD RBAC.

## Next steps

When adding new applications or environments:

1. Drop a new overlay under `app/overlays/<env>` or extend the `argocd/env/<env>` bundle.
2. Keep the environment folder self-contained (AppProject + Application + kustomization) so the root app can build it in one pass.
3. Update the bootstrap root app to point at the new `argocd/env/<env>` path and sync it via the Argo CD CLI/UI.
