# Architecture

## End-to-end flow

```mermaid
flowchart TD
    dev([Developer]) -->|open PR| pr[Pull Request]

    subgraph CI["CI gate (PR) — .github/workflows/ci.yaml"]
        pr --> lint[lint]
        pr --> test["unit tests + coverage"]
        pr --> sec["security: npm audit + Trivy fs"]
        pr --> k6ci["k6 smoke + load against ephemeral container"]
    end

    lint & test & sec & k6ci -->|all green, merge| main[(main branch)]

    subgraph CD["Delivery — .github/workflows/build.yaml"]
        main --> build["build multi-stage image"]
        build --> scan["Trivy image scan"]
        scan --> push["push to GHCR (tag = sha-xxxxxxx)"]
        push --> bump["kustomize edit set image\non staging overlay + git commit"]
    end

    bump --> repo[(gitops/overlays/staging)]
    repo -->|watches git| argo{{ArgoCD}}
    argo -->|auto-sync| stg[[staging namespace]]

    stg --> k6stg["k6 perf gate vs staging"]
    k6stg -->|pass| approve["promote-prod.yaml\nmanual + environment approval"]
    approve --> repoP[(gitops/overlays/production)]
    repoP -->|watches git| argo
    argo -->|manual sync| prod[[production namespace]]
```

## Why GitOps (and not `kubectl apply` in CI)

- **Git is the single source of truth.** The cluster state always matches what's
  committed. No "what's actually running?" mystery.
- **CI has no cluster credentials.** CI only pushes an image and edits a YAML
  tag. ArgoCD (inside the cluster) pulls changes. Smaller blast radius, fewer
  secrets in CI.
- **Rollback = `git revert`.** Roll back a deploy the same way you roll back code.
- **Drift correction.** ArgoCD `selfHeal` reverts manual cluster edits back to
  the committed state.

## Environment promotion

| Stage | Sync | Image tag source | Replicas |
| --- | --- | --- | --- |
| staging | ArgoCD automated (prune + selfHeal) | auto-bumped by `build.yaml` on every merge | 1 |
| production | manual sync + GitHub Environment approval | promoted by `promote-prod.yaml` (workflow_dispatch) | 3 |

The same image artifact (by digest/tag) that passed staging is what ships to
production — build once, promote the artifact, never rebuild per environment.
