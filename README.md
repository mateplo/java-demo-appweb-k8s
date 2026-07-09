# java-demo-appweb-k8s

Manifestes Kustomize de déploiement de la webapp Java. C'est la moitié « k8s » du couple
dual-repo : le code et la CI sont dans `java-demo-appweb-code` (qui build et pousse l'image
sur GHCR) ; ici on décrit comment l'app tourne dans le cluster. Synchronisé par ArgoCD
(repo `classlab-infra`).

## Structure

```
base/                  # commun à tous les envs
  app.yaml             # Deployment + Service (Tomcat, port 8080 -> Service 80)
  httproute.yaml       # expose via le Gateway classlab, sur *.classlab.lan
  servicemonitors.yaml # scrape Prometheus de /metrics
  kustomization.yaml   # ressources + image surveillée par Image Updater
overlays/
  dev/                 # ns java-demo-appweb-dev, tag dev-*, hostname dev
  prod/                # ns java-demo-appweb-prod, tag main-*, + hpa.yaml (autoscaling)
```

## Utilisation

On n'applique rien à la main. ArgoCD crée une Application par env et synchronise
`overlays/<env>` ; un `git push` ici = un déploiement. Le champ `newTag` de chaque overlay
est réécrit automatiquement par Argo CD Image Updater à chaque nouvelle image.

Valider un env avant de committer (obligatoire, ça doit builder sans erreur) :

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod
```

Patcher un env : ajouter une entrée `patches:` dans `overlays/<env>/kustomization.yaml`,
ciblée par `kind` + `name`. Le hostname de l'HTTPRoute est déjà patché par env.

Correspondance branche → env → tag (doit rester cohérente avec la CI et l'Image Updater) :

| Overlay | Namespace | Branche | Tag épinglé |
|---------|-----------|---------|-------------|
| `dev`   | `java-demo-appweb-dev`  | `dev`  | `dev-<sha>`  |
| `prod`  | `java-demo-appweb-prod` | `main` | `main-<sha>` |

## Points d'attention

- La limite mémoire du conteneur est calibrée pour une JVM (512Mi). Ne pas la redescendre
  à des valeurs « nginx » : Tomcat se fait OOMKill au démarrage en dessous de ~256Mi.
- L'image GHCR doit être **publique** (aucun `imagePullSecret` n'est défini), sinon
  `ImagePullBackOff`.
- Le ServiceMonitor scrape le port nommé `http` sur `/metrics` — l'endpoint est exposé par
  le WAR (client Prometheus embarqué). Si tu retires les métriques côté app, retire aussi
  ce fichier.
