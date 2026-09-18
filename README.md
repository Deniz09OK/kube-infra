> Repo parent : [kube-securepods](https://github.com/Deniz09OK/kube-securepods)

# kube-infra

Repo GitOps **infra** du projet Epitech "KUBE — Sailing through the Clouds" (equipe SecurePods). Provisioning du cluster Kubernetes (Ansible) et deploiement des outils de plateforme, synchronise par ArgoCD.

## Contenu

- Provisioning Ansible (teardown/rebuild complet du cluster)
- Traefik (Ingress), ArgoCD (GitOps), Headlamp (dashboard)
- VictoriaMetrics (monitoring) + Loki/Grafana Alloy (logging)
- Keycloak + dex (OIDC), RBAC, Sealed Secrets, ValidatingAdmissionPolicy
- cert-manager (HTTPS), registre d'images prive

## Repo lie

- [kube-app](https://github.com/Deniz09OK/kube-app) — repo GitOps de l'application

## Documentation

Voir [AGENT.md](./AGENT.md) pour le detail complet et [CONTRIBUTING.md](./CONTRIBUTING.md) pour la convention de commits.
