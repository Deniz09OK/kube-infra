> Repo parent : [kube-securepods](https://github.com/Deniz09OK/kube-securepods)

# kube-infra

Repo GitOps **infra** du projet **MediSync** (EPITECH — T-ESP-900 / equipe SecurePods). Ce depot couvre le provisioning du cluster Kubernetes sur AWS et le deploiement des composants de plateforme synchronises par ArgoCD.

## Infrastructure cible & etat valide

- **Region & securite** : AWS `eu-central-1` (`eu-central-1a`), security group `kubequest2-group-26-sg`, authentification SSO profil `kubequest2` (region SSO `eu-west-3`, compte `182309360430`).
- **Noeuds EC2 (`t4g.medium`, Amazon Linux 2023 `aarch64`, `dnf`)** :
  - `kube-1` (`i-0a7fdfcf2f4f55a8f`) : control-plane — IP privee `10.0.0.98`, IP publique `63.188.247.207`
  - `kube-2` (`i-0509546af0846e566`) : worker 1 — IP privee `10.0.0.212`, IP publique `3.121.235.199`
  - `kube-3` (`i-0c9362c7a33c83bc0`) : worker 2 — IP privee `10.0.0.213`, IP publique `3.68.83.96`
- **Stack cluster** : Kubernetes `v1.30.14`, `containerd` (`SystemdCgroup = true`), CNI Calico (`v3.28.0`, CIDR `192.168.0.0/16`), ArgoCD (`v3.5.6`) et Headlamp (`v0.46.0`).

## Acces SSM (SEC-11)

1. Prerequis : `aws` CLI et `session-manager-plugin` installes.
2. Connexion SSO puis session SSM interactive :

```bash
aws sso login --profile kubequest2
aws --profile kubequest2 ssm start-session --target <INSTANCE_ID>
sudo -i
```

- Contraintes observees : `ssm:SendCommand` non autorise (`AccessDeniedException`) et tunnel SSH via `AWS-StartSSHSession` indisponible (`Connection closed by UNKNOWN port 65535`).

## Provisioning Ansible (SEC-12)

- Playbook principal : `ansible/playbooks/site.yml` (`prerequisites`, `containerd`, `kubernetes`, `control_plane`, `workers`) avec inventaire `ansible/inventory/hosts.ini` et variables `ansible/inventory/group_vars/k8s_cluster.yml`.
- Verification syntaxique locale :

```bash
ANSIBLE_CONFIG=ansible/ansible.cfg ansible-playbook -i ansible/inventory/hosts.ini --syntax-check ansible/playbooks/site.yml
```

## Structure Kustomize & GitOps (SEC-14, SEC-15, SEC-16, SEC-17)

- `base/kustomization.yaml` : point d'entree Kustomize incluant `argocd/` et `headlamp/` (`traefik/` retire de la base sur la branche `feat/SEC-12-traefik-ingress`).
- `base/argocd/` : `Namespace` `argocd`, manifestes amont ArgoCD `v3.5.6` (`install.yaml`), `AppProject` (`kube-infra`, `kube-app`), `Application` `kube-infra` (`prune: true`, `selfHeal: true`, `ServerSideApply=true`, `CreateNamespace=true`), ajustement des ressources (`patch-resources.yaml`) et protection des CRD (`patch-crd-prune.yaml`, `Prune=false`).
- `base/headlamp/` : `Namespace` `headlamp`, `Deployment` durci (`headlamp:v0.46.0`, `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `seccompProfile: RuntimeDefault`, `HOME=/tmp/headlamp` sur `emptyDir`), `Service` `ClusterIP` (`4466`) et RBAC `headlamp-viewer` en lecture seule sans acces aux `secrets`.

### Bootstrap & reconciliation ArgoCD (`selfHeal` / `prune`)

```bash
kubectl apply --server-side --force-conflicts -k base/argocd/
kubectl apply --server-side --force-conflicts -k base/
```

- **`selfHeal`** : valide sur `headlamp` (passage a `0` replica retabli automatiquement a `1/1` par ArgoCD).
- **`prune`** : active (`automated.prune: true`) ; a valider en ajoutant une ressource de test dans `base/` -> sync -> suppression dans Git -> verification de la suppression automatique dans le cluster, puis nettoyage.

## Repo lie

- [kube-app](https://github.com/Deniz09OK/kube-app) — repo GitOps de l'application

## Documentation

Voir [AGENT.md](./AGENT.md) pour le contexte complet du projet et [CONTRIBUTING.md](./CONTRIBUTING.md) pour les conventions de commits (`<type>(SEC-XX): <description courte>`).
