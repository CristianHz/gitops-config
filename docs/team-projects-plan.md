# Team-Based ApplicationSet Plan (Internal Developer Platform)

## Goal
Design Argo CD + ApplicationSet so each **team** is an Argo Project (for governance), and each app automatically gets **dev** and **staging** environments in the same cluster. Production will live in a separate cluster.

## Confirmed Requirements
- Team is the ownership boundary (example: `team-research`).
- App inside team (example: `react-skeleton`).
- Auto-create 2 environments now: `dev`, `staging`.
- Each environment must have separate access endpoint (you prefer different ports / NodePort style now).
- Database should be addable later (in-cluster or managed external).

## Recommended Approach
Use:
- **Argo Project per team** (RBAC + source/destination constraints).
- **ApplicationSet with matrix generator** for `{app} x {environment}`.
- **Kustomize overlays** for env differences (better for platform evolution than list-only YAML).
- **App-of-apps/team bootstrap** to register team project + appset from Git.
- **Infra and app separation** so platform services (DB charts, ingress, secrets operators) are managed independently from app releases.

Why Kustomize over pure Helm for this case:
- Simple native YAML patching for platform controls (namespaces, quotas, network policy, ingress, NodePort).
- Easy environment overlays (`dev`, `staging`) with minimal templating.
- Keeps app manifests readable for teams, while platform can enforce base controls.

## Target Repository Layout
```text
gitops-config/
  argocd/
    projects/
      team-research-project.yaml
      platform-infra-project.yaml
    appsets/
      team-research-apps.yaml
      platform-infra.yaml
  infra/
    shared/
      ingress-nginx/
      external-secrets/
      cert-manager/
    teams/
      team-research/
        databases/
          postgres-dev/
          postgres-staging/
  teams/
    team-research/
      react-skeleton/
        base/
          deployment.yaml
          service.yaml
          ingress.yaml
          kustomization.yaml
        overlays/
          dev/
            kustomization.yaml
            patch-deployment.yaml
            patch-service.yaml
            patch-ingress.yaml
          staging/
            kustomization.yaml
            patch-deployment.yaml
            patch-service.yaml
            patch-ingress.yaml
```

## Domain Split: Infra vs Apps
### Infra domain
- Purpose: shared platform capabilities and team infrastructure dependencies.
- Examples: database charts, ingress controller, cert-manager, secret operators, observability stack.
- Lifecycle: slower, stricter approvals, stronger guardrails.

### Apps domain
- Purpose: app runtime manifests and environment overlays.
- Examples: deployment/service/ingress/config of `react-skeleton`.
- Lifecycle: faster release velocity, team-owned changes.

## Ownership Model
| Domain | Owner | Argo Project | Typical Change Cadence |
|---|---|---|---|
| infra/shared | Platform team | platform-infra-project | low/moderate |
| infra/teams/team-research | Platform + Team agreement | platform-infra-project or team-research-project | moderate |
| teams/team-research/* | Team Research | team-research-project | frequent |

Rule of thumb:
- Provisioning databases belongs in `infra`.
- App only consumes DB through config/secrets contract.

## Naming and Isolation Model
- Team namespace pattern: `team-research-dev`, `team-research-staging`.
- App name inside namespace: `react-skeleton`.
- Argo Application name pattern: `team-research-react-skeleton-dev` and `...-staging`.

This avoids collisions and gives clear ownership boundaries.

## Access Strategy (Current Preference)
### Same cluster: NodePort per environment
- `dev` service type NodePort, e.g. `30080`.
- `staging` service type NodePort, e.g. `30081`.
- Access example:
  - `http://localhost:30080` -> dev
  - `http://localhost:30081` -> staging

### Future production
- Keep `ClusterIP + Ingress` in production cluster.
- Use real DNS/TLS there.

## Team Argo Project Design
Each team gets an Argo Project with:
- Allowed source repos: `https://github.com/CristianHz/gitops-config.git`
- Allowed destination namespaces: `team-research-*`
- Optional role bindings for team members
- Optional policy: deny cluster-scoped manifests except approved ones

Platform infra gets a separate Argo Project with tighter controls:
- Restrict destinations to infra namespaces.
- Allow controlled cluster-scoped resources.
- Require review gates for critical components.

## ApplicationSet Design
Each team appset should:
- Generate applications from matrix of:
  - app list (start with `react-skeleton`)
  - env list (`dev`, `staging`)
- Set source path to corresponding overlay.
- Set destination namespace per env.
- Enable automated sync + prune + self-heal.

## Environment Overlay Responsibilities
In overlays, define only differences:
- Image tag strategy per env (can use same image initially).
- Replica count (dev lower, staging moderate).
- NodePort per env.
- Ingress host/path per env (optional for local).
- ConfigMap/secret references.

## Database Expansion Plan
When `react-skeleton` needs a DB, support two patterns:

### Option A: In-cluster DB (fast local/dev)
- Example: Postgres Helm chart/operator under `infra/teams/team-research/databases/*`.
- Pros: simple local setup, fast onboarding.
- Cons: operational burden, backup/HA complexity.

### Option B: External managed DB (recommended for staging/prod)
- App receives DB endpoint/credentials via secret.
- Pros: managed backups/HA/security.
- Cons: networking/secrets setup needed.

### Contract Between Infra and App
- Infra publishes:
  - DB endpoint
  - DB port
  - credentials secret reference
- App overlay consumes those values only; no DB chart logic inside app folder.

### Decision Matrix
- Local dev speed: **A wins**
- Reliability/compliance: **B wins**
- Cost control early-stage: **A can win**
- Scale/operations maturity: **B wins**

Recommended path:
- Dev: in-cluster DB allowed.
- Staging: prefer managed DB once available.
- Prod: managed DB only.

## Rollout Phases
1. **Foundation**
- Create `argocd/projects/team-research-project.yaml`.
- Create `argocd/projects/platform-infra-project.yaml`.
- Create app appset for `react-skeleton` with dev/staging generation.
- Create infra appset skeleton (shared + team infra).

2. **Refactor app manifests**
- Move current `react-skeleton-config` into `teams/team-research/react-skeleton/base + overlays`.
- Keep same deployed behavior first.

3. **Cutover**
- Disable/remove old single app appset.
- Apply team appset and verify 2 generated apps in Argo.

4. **Add DB capability**
- Add DB provisioning under `infra/teams/team-research/databases`.
- Start with dev only; validate migration/secrets pattern.

5. **Prepare production cluster**
- Duplicate team project policy in prod Argo instance.
- Enable prod overlay only in prod cluster appset.

## Guardrails to Add Early
- ResourceQuota + LimitRange per namespace.
- NetworkPolicy default deny + explicit allow.
- Secret management standard (External Secrets or Sealed Secrets).
- Image tag policy: immutable tags (`sha-*`) only in staging/prod.

## CI/CD Integration Notes
Your current CI updates image tags in GitOps. Keep that and extend it to:
- Update `dev` overlay first.
- Optional promotion workflow `dev -> staging` via PR or manual dispatch.

## Success Criteria
- Creating `team-research/react-skeleton` once in Git auto-creates:
  - Argo app `...-dev`
  - Argo app `...-staging`
- Both sync automatically and expose separate endpoints.
- Team-level access controls enforced by Argo Project.
- Database can be added in infra without changing app domain structure.
- Infra and app deploy pipelines can evolve independently.

## Next Implementation Tasks (Practical)
1. Create `team-research` Argo Project manifest.
2. Create team `ApplicationSet` matrix manifest for `react-skeleton x {dev,staging}`.
3. Refactor existing react-skeleton manifests into base/overlays.
4. Define NodePort map (`dev=30080`, `staging=30081`).
5. Commit/push and validate generated applications in Argo UI.
