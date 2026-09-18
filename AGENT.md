# AGENT.md — kube-infra

## Contexte

Repo GitOps **infra** du projet Epitech "KUBE — Sailing through the Clouds" (equipe SecurePods). Ce repo contient tout ce qui provisionne le cluster Kubernetes et deploie les outils de plateforme (hors application metier, qui vit dans `kube-app`). ArgoCD synchronise ce repo vers le cluster.

Repo parent / doc d'ensemble : voir `kube-securepods`.

## Infrastructure cible

- 3 VMs AWS :
  - `kube-1` : control-plane + worker, IP publique stable
  - `kube-2`, `kube-3` : workers, IP publique ephemere, IP privee VPC stable
- Acces : AWS SSM Session Manager uniquement (connection plugin `aws_ssm` ou tunnel SSH via SSM) — **pas de SSH public par defaut**
- Stockage partage : Amazon EFS, utilise pour les PersistentVolumes
- Pas de load balancer/DNS wildcard dedie -> exposition via NodePort + sslip.io/nip.io

## Contenu attendu du repo

- `ansible/` : playbooks et roles de provisioning du cluster. **Doit permettre un teardown/rebuild complet du cluster.**
- `manifests/` ou `charts/` (selon organisation choisie), synchronises par ArgoCD, pour :
  - **Traefik** — Ingress controller (choisi plutot que Gateway API)
  - **ArgoCD** — operateur GitOps (choisi plutot que FluxCD)
  - **Headlamp** — dashboard Kubernetes (choisi plutot que kubernetes-dashboard)
  - **VictoriaMetrics** — monitoring (choisi plutot que kube-prometheus)
  - **Loki + Grafana Alloy** — logging, **Grafana** — visualisation
  - **Keycloak** — Identity Provider self-hosted, + **dex** pour la federation OIDC
  - OIDC doit couvrir : Grafana, Headlamp, ArgoCD, kubectl, **et l'API Kubernetes elle-meme**
  - Mapping des groupes/roles OIDC vers des `Role`/`RoleBinding` Kubernetes (RBAC)
  - **Sealed Secrets** — gestion des secrets (choisi plutot que External Secrets)
  - **ValidatingAdmissionPolicy** (CEL) — admission control (impose par le sujet)
  - **cert-manager** + Let's Encrypt — HTTPS sur tous les endpoints exposes
  - Registre d'images prive et authentifie

## Decisions techniques actees (voir Journal de decisions Confluence pour le detail/justification)

| Besoin | Choix | Alternative ecartee |
|---|---|---|
| Ingress | Traefik | Gateway API |
| Dashboard | Headlamp | kubernetes-dashboard |
| Monitoring | VictoriaMetrics | kube-prometheus |
| Logging | Loki + Grafana Alloy | (impose) |
| GitOps operator | ArgoCD | FluxCD |
| Secrets | Sealed Secrets | External Secrets |
| Identity | Keycloak + dex | — |
| Admission control | ValidatingAdmissionPolicy (CEL) | (impose) |

Toute nouvelle decision structurante doit etre ajoutee au Journal de decisions Confluence avant d'etre implementee ici.

## Points d'attention / demo de soutenance

- Le teardown/rebuild complet du cluster via Ansible doit etre demontre.
- Le rollout progressif doit etre demontre avec une panne injectee (readiness probe qui echoue) pour prouver qu'il n'y a pas de bascule de trafic vers un pod non pret.
- Affinity/anti-affinity doit repartir les replicas sur plusieurs noeuds.

## Bonus (optionnels, non tickets)

kube-rbac-proxy pour la multi-tenancy Prometheus/Loki, evaluation de la securite des artefacts Helm, RBAC fin, durcissement NetworkPolicy, images Docker allegees, deploiement zero-downtime, documentation de mise a niveau du cluster.

## Conventions

- Commits : voir `CONTRIBUTING.md` — format `<type>(SEC-XX): <description courte>`
- Toute VM/manifest doit rester reproductible depuis zero via Ansible + ArgoCD (pas de configuration manuelle non versionnee)
- Secrets **jamais** en clair dans le repo — toujours via Sealed Secrets
