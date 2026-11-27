# Hostnames exposes — convention sslip.io (SEC-20)

Le cluster n'a pas de nom de domaine. On utilise [sslip.io](https://sslip.io), un DNS public qui resout un nom contenant une IP vers cette IP :

```
argocd.63.188.247.207.sslip.io  ->  63.188.247.207
```

Ces noms sont de vrais noms DNS publics : Let's Encrypt peut donc emettre des certificats pour eux (cert-manager, challenge HTTP-01 sur le port 80).

## IP cible

`63.188.247.207` est l'**Elastic IP** de kube-1 (`eipalloc-01b09ac3ded352ead`). Elle reste la meme quand on arrete puis relance les VMs. kube-1 est le seul noeud qui recoit du trafic externe : Traefik y ecoute en hostPort 80/443 (voir [base/traefik/README.md](../base/traefik/README.md)). kube-2 et kube-3 n'ont pas d'IP publique stable et ne doivent jamais apparaitre dans un hostname.

## Convention

```
<service>.<IP kube-1>.sslip.io
```

- `<service>` : **un seul label DNS**, en minuscules, sans point (`grafana`, pas `grafana.monitoring`).
- Le suffixe `<IP kube-1>.sslip.io` n'est ecrit qu'a un seul endroit : `data.suffix` dans [base/cluster-domain.yaml](../base/cluster-domain.yaml).

| Service | Hostname | Ticket |
|---|---|---|
| whoami (service de test HTTPS) | `whoami.63.188.247.207.sslip.io` | SEC-21 |
| ArgoCD | `argocd.63.188.247.207.sslip.io` | |
| Grafana | `grafana.63.188.247.207.sslip.io` | |
| Headlamp | `headlamp.63.188.247.207.sslip.io` | SEC-38 / SEC-64 |
| Keycloak | `keycloak.63.188.247.207.sslip.io` | |
| dex | `dex.63.188.247.207.sslip.io` | |
| Registre d'images | `registry.63.188.247.207.sslip.io` | |
| Application (repo kube-app) | `app.63.188.247.207.sslip.io` | |

Le dashboard Traefik et VictoriaMetrics ne sont pas exposes. On y accede par `port-forward`, ou par Grafana pour les metriques.

## Ecrire un Ingress dans `base/`

On ecrit le host avec le placeholder `CLUSTER_DOMAIN`. Le rendu kustomize de `base/` (celui que fait ArgoCD) le remplace par le suffixe :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt   # quand cert-manager sera en place
spec:
  ingressClassName: traefik
  tls:
    - hosts: [grafana.CLUSTER_DOMAIN]
      secretName: grafana-tls
  rules:
    - host: grafana.CLUSTER_DOMAIN          # -> grafana.63.188.247.207.sslip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: {name: grafana, port: {number: 80}}
```

Pour un chart Helm rendu par `helmCharts`, on met la meme valeur `<service>.CLUSTER_DOMAIN` dans le `values.yaml` du composant. Le remplacement s'applique aussi aux Ingress generes par le chart.

Le mecanisme est la `replacements` de [base/kustomization.yaml](../base/kustomization.yaml). Elle decoupe chaque host sur les `.` et remplace le 2e segment (`index: 1`) par le suffixe. Regles qui en decoulent :

- **Tous** les Ingress de `base/` doivent suivre `<service>.CLUSTER_DOMAIN`. Un host ecrit en dur (`foo.example.com`) serait reecrit en `foo.<suffixe>.com`.
- Les hosts TLS (`spec.tls[].hosts[]`) ne sont remplaces que sur les Ingress qui portent l'annotation `cert-manager.io/cluster-issuer`. Un Ingress avec un bloc `tls` doit donc porter cette annotation.
- Un sous-dossier rendu seul (`kustomize build base/grafana`) garde le placeholder. Il faut toujours rendre `base/`.

## Hors perimetre de la `replacements`

Elle ne traite que les hosts d'Ingress. Les autres endroits qui auront besoin du nom devront etre ajoutes comme nouvelles `targets` de la meme source quand leur ticket arrivera : `url` d'ArgoCD dans `argocd-cm`, issuer OIDC de dex, hostname de Keycloak, etc. On garde ainsi un seul endroit a modifier.

Le repo **kube-app** est une autre Application ArgoCD (chart Helm) et ne passe pas par ce rendu. Son host `app.63.188.247.207.sslip.io` est donc a renseigner dans ses propres values. C'est le seul second endroit a mettre a jour si l'IP change.

## Changer d'IP ou passer a nip.io

1. Modifier `data.suffix` dans `base/cluster-domain.yaml` (par exemple `63.188.247.207.nip.io`, qui suit le meme format).
2. Mettre a jour le host dans les values de kube-app.
3. Merger : ArgoCD regenere tous les Ingress. cert-manager demandera de nouveaux certificats pour les nouveaux noms.

Attention : les certificats Let's Encrypt sont lies aux noms. Changer de suffixe trop souvent peut atteindre les limites de debit de Let's Encrypt.

## Verification

```bash
# Resolution DNS depuis l'exterieur
nslookup whoami.63.188.247.207.sslip.io        # -> 63.188.247.207

# Hosts rendus (meme image qu'ArgoCD)
docker run --rm -v "$PWD:/work" -w /work --entrypoint kustomize \
  quay.io/argoproj/argocd:v3.5.6 build --enable-helm base | grep -E "host:|sslip"

# Routage par Traefik : un Ingress avec ce host repond (le service sinon : 404 de Traefik)
curl -i http://whoami.63.188.247.207.sslip.io
```
