# Traefik — Ingress Controller (SEC-19)

Point d'entree HTTP(S) unique du cluster. Il est installe depuis le chart officiel `traefik/traefik` **42.4.1** (Traefik **v3.7.19**), que kustomize rend via `helmCharts`. ArgoCD le synchronise ensuite avec le reste de `base/` (application `kube-infra`).

## Fichiers

| Fichier | Role |
|---|---|
| `kustomization.yaml` | Namespace `traefik` et inflation du chart (CRD incluses) |
| `values.yaml` | Configuration du chart (exposition, entrypoints, ressources, labels, securite) |
| `namespace.yaml` | Namespace `traefik` labellise |
| `middleware-redirect-https.yaml` | Middleware `redirect-https` (HTTP -> HTTPS), a activer route par route |

## Exposition

Le lab n'a pas de LoadBalancer cloud : la seule IP publique stable est celle de kube-1.

- **Pod epingle sur kube-1** (`nodeSelector node-role.kubernetes.io/control-plane`) avec **hostPort 80/443**, soit `http(s)://<IP publique kube-1>` sans port exotique. C'est aussi ce qu'il faut a Let's Encrypt (HTTP-01 sur le port 80).
- **Service NodePort** : `30080` (web) et `30443` (websecure), pour l'acces intra-VPC et le debug.
- **Un seul replica** : un seul pod peut tenir les hostPorts sur kube-1. Le rolling update se fait donc en `maxSurge: 0` / `maxUnavailable: 1`. Une mise a jour de Traefik implique une coupure de quelques secondes ; l'application metier n'est pas concernee.
- **Security Group (gere par le lab, hors GitOps)** : `kubequest2-group-26-sg` autorise deja en entree TCP **80** et **443** depuis `0.0.0.0/0`, ainsi que tout le trafic entre les noeuds du groupe (verifie le 2026-11-23). La plage NodePort n'est pas ouverte depuis Internet, d'ou le hostPort.

## Entrypoints

| Entrypoint | Port conteneur | Expose | Usage |
|---|---|---|---|
| `web` | 8000 | hostPort 80 / NodePort 30080 | HTTP |
| `websecure` | 8443 | hostPort 443 / NodePort 30443 | HTTPS (TLS actif) |
| `traefik` | 8080 | non | ping, API, dashboard |
| `metrics` | 9100 | non | metriques Prometheus (pour VictoriaMetrics) |

**Redirection HTTP -> HTTPS (prevue, non active)** : elle reste desactivee tant que cert-manager n'est pas en place, afin de garder la 404 attendue sur `http://` et de ne pas bloquer les challenges ACME. Deux options :

- **par route** : annotation `traefik.ingress.kubernetes.io/router.middlewares: traefik-redirect-https@kubernetescrd` sur l'Ingress ;
- **globale** : decommenter `ports.web.http.redirections` dans `values.yaml`, avec `allowACMEByPass: true`.

## Dashboard

Le dashboard est actif mais **non expose** : pas d'IngressRoute, et l'entrypoint `traefik` est absent du Service. Pour y acceder :

```bash
kubectl -n traefik port-forward deploy/traefik 8080:8080
# http://localhost:8080/dashboard/
```

## Pre-requis ArgoCD (`--enable-helm`)

Le rendu de `helmCharts` exige `kustomize.buildOptions: --enable-helm` dans `argocd-cm` (voir `base/argocd/patch-argocd-cm.yaml`).

- **Nouveau cluster** : le bootstrap `kubectl apply -k base/argocd --server-side` inclut deja l'option.
- **Cluster existant** : il faut appliquer l'option **une fois a la main**. Sinon ArgoCD ne peut plus generer les manifests de `kube-infra`, et donc pas non plus le patch qui l'active :

  ```bash
  kubectl apply -k base/argocd --server-side
  ```

## Verification (criteres d'acceptation)

```bash
kubectl -n traefik get pods -o wide        # Running, sur kube-1
kubectl -n argocd get application kube-infra   # Synced / Healthy
curl -i http://<IP publique kube-1>        # 404 page not found (repondu par Traefik)
```

Rendu local identique a ArgoCD (meme image, donc memes kustomize et helm) :

```bash
docker run --rm -v "$PWD:/work" -w /work --entrypoint kustomize \
  quay.io/argoproj/argocd:v3.5.6 build --enable-helm base/traefik
```
