# GitOps Configuration

Ce dépôt suit les principes GitOps pour le déploiement automatisé sur Kubernetes.

## 🔄 Workflow GitOps

### Principe
- Le dépôt Git est la **source unique de vérité** (Single Source of Truth)
- Toute modification des manifests Kubernetes déclenche automatiquement un déploiement
- Les déploiements sont reproductibles et traçables via l'historique Git

### Pipeline CI/CD

Le workflow `.github/workflows/gitops-deploy.yml` s'exécute automatiquement à chaque push sur la branche `main` et effectue :

1. **Validation** :
   - Vérification de la syntaxe YAML
   - Validation des manifests avec `kubectl --dry-run`
   - Détection des erreurs de configuration

2. **Déploiement** (à configurer) :
   - Connexion au cluster Kubernetes
   - Application des manifests dans l'ordre correct
   - Vérification du statut des Pods

## 🚀 Configuration du Déploiement Automatique

### Option 1 : GitHub Actions avec Secret KUBECONFIG

1. Obtenez votre kubeconfig :
```bash
kubectl config view --flatten --minify > kubeconfig.yaml
```

2. Encodez en base64 :
```bash
cat kubeconfig.yaml | base64 > kubeconfig-base64.txt
```

3. Ajoutez le secret dans GitHub :
   - Allez dans **Settings → Secrets and variables → Actions**
   - Créez un secret nommé `KUBECONFIG`
   - Collez le contenu de `kubeconfig-base64.txt`

4. Décommentez les lignes de déploiement dans `.github/workflows/gitops-deploy.yml`

### Option 2 : ArgoCD (Recommandé pour Production)

1. Installez ArgoCD sur votre cluster :
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Créez une application ArgoCD :
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fleet-management
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/samhanoun/Fleet-Management-K8s-GitOps.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

3. ArgoCD synchronisera automatiquement le cluster avec le dépôt Git

### Option 3 : Flux CD

1. Installez Flux :
```bash
flux bootstrap github \
  --owner=samhanoun \
  --repository=Fleet-Management-K8s-GitOps \
  --branch=main \
  --path=. \
  --personal
```

2. Flux détectera et appliquera automatiquement les changements

## 📋 Workflow de Développement GitOps

1. **Développer localement** :
```bash
# Modifier les manifests YAML
vim webapp-deployment.yaml
```

2. **Tester localement** :
```bash
kubectl apply --dry-run=client -f webapp-deployment.yaml
```

3. **Commiter et pousser** :
```bash
git add webapp-deployment.yaml
git commit -m "Update webapp replicas to 3"
git push origin main
```

4. **GitHub Actions valide** automatiquement les changements

5. **Déploiement automatique** (si configuré avec ArgoCD/Flux)

## 🔐 Bonnes Pratiques GitOps

### ✅ À FAIRE
- Commiter tous les changements de configuration dans Git
- Utiliser des branches pour les changements majeurs
- Faire des Pull Requests pour review
- Tagger les releases (ex: v1.0.0)
- Utiliser des namespaces séparés par environnement

### ❌ À ÉVITER
- Modifier les ressources directement avec `kubectl edit`
- Appliquer des manifests non versionnés
- Stocker des secrets en clair dans Git (utiliser SealedSecrets ou External Secrets)

## 🏷️ Structure GitOps Avancée (Optionnel)

Pour des déploiements multi-environnements :

```
fleet-management-k8s/
├── base/                    # Manifests de base
│   ├── mongodb-deployment.yaml
│   ├── webapp-deployment.yaml
│   └── ...
├── overlays/
│   ├── dev/                 # Configuration développement
│   │   └── kustomization.yaml
│   ├── staging/            # Configuration staging
│   │   └── kustomization.yaml
│   └── production/         # Configuration production
│       └── kustomization.yaml
└── .github/
    └── workflows/
        └── gitops-deploy.yml
```

Utilisez Kustomize pour gérer les différences entre environnements.

## 📊 Monitoring GitOps

### Vérifier l'état du déploiement

```bash
# Voir les derniers déploiements
kubectl rollout history deployment/fleetman-webapp

# Voir l'état actuel
kubectl get pods -w

# Voir les événements récents
kubectl get events --sort-by='.lastTimestamp'
```

### Rollback en cas de problème

```bash
# Via Git (recommandé)
git revert HEAD
git push origin main

# Ou via kubectl (temporaire)
kubectl rollout undo deployment/fleetman-webapp
```

## 🔗 Ressources

- [GitOps Principles](https://www.gitops.tech/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Flux Documentation](https://fluxcd.io/docs/)
- [GitHub Actions Kubernetes](https://github.com/marketplace/actions/kubernetes-cli-kubectl)
