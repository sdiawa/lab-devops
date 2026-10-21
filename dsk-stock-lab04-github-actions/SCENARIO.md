# Lab 04 — De GitHub Actions à Kubernetes : construire, publier et déployer DSK Stock

> **Niveau :** intermédiaire, avec rappels pour débutants. **Durée indicative :** 4 à 6 heures pour une première réalisation, hors installation de K3s et téléchargement des images.  
> **Fil conducteur :** partir d'un dépôt Git, vérifier le code, fabriquer des images Docker, les publier dans GHCR, puis déployer automatiquement une application complète dans K3s.  
> **Principe :** vous travaillez sur **votre propre infrastructure**. Les noms et adresses de l'exemple sont à remplacer ; aucun accès à une VM ou à un cluster n'est fourni.

## Votre mission

Vous rejoignez une équipe qui maintient **DSK Stock**, une application de gestion de stock. Aujourd'hui, chaque mise à jour exige plusieurs manipulations : lancer des tests, construire les images, les publier, se connecter au serveur et relancer le déploiement. Une erreur de manipulation ou une étape oubliée peut interrompre le service.

Votre mission consiste à automatiser cette chaîne avec **GitHub Actions** et à prouver que la version exécutée dans Kubernetes correspond bien à un commit Git identifié. Vous devrez également expliquer ce qui se passe lorsqu'un test échoue, lorsqu'un runner n'a pas les permissions nécessaires, ou lorsqu'un nom DNS ne se résout pas.

### Compétences visées

À la fin du lab, vous saurez :

1. distinguer **CI**, **CD**, **runner**, **job**, **step**, **workflow** et **artifact/image** ;
2. comprendre la différence entre un **fork**, une **branche**, une **pull request** et un **push** sur `main` ;
3. expliquer pourquoi `verify` doit réussir avant la publication d'images ;
4. construire et publier des images versionnées dans **GitHub Container Registry (GHCR)** ;
5. installer un **runner self-hosted** et le relier à GitHub sans lui donner les droits administrateur du cluster ;
6. utiliser **ServiceAccount**, **Role** et **RoleBinding** pour autoriser un déploiement dans un seul namespace ;
7. déployer avec **Helm** en injectant le SHA du commit comme tag d'image ;
8. expliquer les rôles respectifs de **ClusterIP**, **LoadBalancer**, **Ingress** et **Traefik** ;
9. diagnostiquer un problème de déploiement avec `kubectl`, `helm`, `curl` et les logs GitHub Actions ;
10. vérifier l'application jusqu'à la création et la consultation d'un produit.

## 1. Comprendre l'architecture avant d'exécuter les commandes

```text
Développeur
    │ git push / pull request
    ▼
GitHub — dépôt forké
    │
    ▼
GitHub Actions
    ├── verify (runner GitHub)
    │     lint + tests + TypeScript + build + helm lint
    │
    ├── images (runners GitHub, après verify)
    │     Docker Buildx : backend + frontend, amd64/arm64
    │     └── GHCR : images taguées avec le SHA du commit
    │
    └── deploy (runner self-hosted, push sur main uniquement)
          │ kubeconfig restreint / RBAC
          ▼
        Helm upgrade --install
          ▼
        K3s / namespace stock-lab04
          ├── Deployment frontend ── Service ClusterIP :80
          ├── Deployment backend  ── Service ClusterIP :3001
          ├── StatefulSet PostgreSQL ── PVC
          ├── Job de migration de base de données
          └── Ingress stock (Traefik)
                  ▲
Navigateur ── DNS ── Traefik (Service LoadBalancer) ──┘
```

**CI (Continuous Integration)** : on vérifie automatiquement le code à chaque changement. Une CI réussie ne signifie pas que l'application a été déployée.  
**CD (Continuous Delivery/Deployment)** : on prépare ou réalise automatiquement la livraison. Dans ce lab, le job `deploy` réalise effectivement un déploiement sur K3s.  
**Runner** : machine qui exécute un job GitHub Actions. Les jobs `verify` et `images` utilisent des machines fournies par GitHub ; `deploy` utilise une machine que vous administrez.  
**GHCR** : registre d'images OCI/Docker de GitHub. Il stocke les images ; il ne les exécute pas.  
**Helm** : gestionnaire de packages Kubernetes. Un *chart* contient des templates YAML et des valeurs permettant de produire les ressources Kubernetes.

### Pourquoi utiliser le SHA du commit ?

Le tag `ghcr.io/COMPTE/lab-devops-backend:<SHA>` permet de relier une image à une révision du dépôt. Contrairement à `latest`, le tag ne change pas à chaque nouveau déploiement **dans le fonctionnement prévu de ce lab**. Pour une garantie forte d'immuabilité, on peut aller plus loin et déployer par digest `sha256` et/ou protéger les tags du registre.

## 2. Préparer votre environnement

### 2.1 Pré-requis

- Un compte GitHub autorisé à créer un fork et à utiliser Actions/GHCR.
- Une machine Linux avec `git`, `kubectl`, `helm` et un accès réseau à GitHub/GHCR.
- Un cluster **K3s** fonctionnel, avec **Traefik** installé et une StorageClass disponible.
- Un namespace de travail, ici `stock-lab04`.
- Une application déjà fonctionnelle avec frontend, backend et PostgreSQL, ou les manifests/charts nécessaires pour l'installer.
- Un accès administrateur Kubernetes **uniquement pour la préparation** des droits du runner.

**Exemple de réalisation** : VM Ubuntu 24.04, K3s, IP `10.0.0.50`, domaine de lab `stock.stockage.dsk`. Ces paramètres sont **des exemples**, pas des valeurs universelles. Un domaine local n'est pas résolu automatiquement par les DNS publics.

### 2.2 Contrôler les outils

```bash
# Sur la machine d'administration Linux
git --version
kubectl version --client
helm version --short

# K3s : le kubeconfig administrateur est habituellement ici.
# À utiliser seulement par l'administrateur, jamais dans GitHub.
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
kubectl get storageclass
kubectl get pods -n kube-system
```

**Pourquoi `KUBECONFIG` ?** `kubectl` et Helm doivent savoir à quel cluster se connecter et avec quelle identité. K3s crée généralement `/etc/rancher/k3s/k3s.yaml`, un fichier très privilégié : ne le copiez pas dans le dépôt et ne le fournissez pas au runner CI/CD.

**Validation :** les nœuds sont `Ready`, la StorageClass existe, et Traefik est opérationnel si vous comptez utiliser un Ingress.

### 2.3 Fork et clone

Sur GitHub, ouvrez `https://github.com/sdiawa/lab-devops`, cliquez sur **Fork**, puis clonez **votre** dépôt :

```bash
mkdir -p "$HOME/devops-labs"
cd "$HOME/devops-labs"
git clone https://github.com/VOTRE_COMPTE/lab-devops.git
cd lab-devops
git remote -v
git branch --show-current
ls dsk-stock-lab04-github-actions
```

**Pourquoi un fork ?** Vous ne devez pas modifier le dépôt de l'auteur. Votre fork est un espace où vous pouvez pousser du code, déclencher les workflows et expérimenter sans affecter les autres apprenants.

**Validation :** `origin` pointe vers `VOTRE_COMPTE/lab-devops`. La branche utilisée est `main`, ou vous adapterez le workflow.

## 3. Découvrir le workflow GitHub Actions

Le workflow **actif** doit être placé dans `.github/workflows/` **à la racine du dépôt Git**, et non dans le sous-dossier du lab. Le scénario d'origine fournit un modèle `workflow.yaml` dans le lab 04.

```bash
# Depuis la racine du dépôt
mkdir -p .github/workflows
cp dsk-stock-lab04-github-actions/workflow.yaml \
   .github/workflows/stock-labs.yaml

sed -n '1,220p' .github/workflows/stock-labs.yaml
```

> **Attention :** si votre fork contient déjà un workflow personnalisé et fonctionnel, ne l'écrasez pas avec `cp`. Comparez les fichiers et intégrez uniquement les modifications nécessaires.

### 3.1 Déclencheurs

```yaml
on:
  pull_request:
  push:
    branches: [main]
  workflow_dispatch:
```

- `pull_request` : exécuter les vérifications d'une proposition de modification. On ne doit pas déployer cette proposition directement sur le cluster.
- `push` sur `main` : exécuter la CI, publier les images puis déployer si les jobs précédents ont réussi.
- `workflow_dispatch` : lancer manuellement un workflow ; dans notre variante, le job `deploy` est **réservé aux push sur `main`**.

### 3.2 Job `verify` : détecter les problèmes tôt

Le job installe Node.js et `pnpm`, utilise le lockfile, exécute lint, Jest, Vitest, TypeScript, build et `helm lint`.

```yaml
verify:
  runs-on: ubuntu-latest
  defaults:
    run:
      working-directory: dsk-stock-lab04-github-actions
  steps:
    - uses: actions/checkout@v6
    - uses: actions/setup-node@v6
      with:
        node-version: "22"
    - run: npm install -g pnpm@8.15.5
    - run: pnpm install --frozen-lockfile
    - run: pnpm lint
    - run: pnpm --filter dsk-stock-backend exec jest --runInBand --passWithNoTests
    - run: pnpm --filter dsk-stock-frontend exec vitest run
    - run: pnpm --filter dsk-stock-frontend exec tsc --noEmit
    - run: pnpm build
    - uses: azure/setup-helm@v5
      with:
        version: v3.18.4
    - run: helm lint chart -f chart/values.lab04.yaml
```

**Explications :**

- `checkout` télécharge le code du commit sur le runner.
- `--frozen-lockfile` empêche une modification implicite des versions des dépendances pendant la CI.
- `lint` détecte certaines erreurs de qualité et de syntaxe ; ce n'est pas un test métier.
- Jest/Vitest vérifient les tests existants ; `--passWithNoTests` peut accepter l'absence de tests sur la cible concernée, donc ne garantit pas une bonne couverture.
- `tsc --noEmit` contrôle les types sans produire de fichiers.
- `helm lint` vérifie la structure du chart, mais ne prouve pas que les Pods démarreront.

**Validation :** dans GitHub → **Actions**, le job `verify` doit être vert. En cas d'échec, ouvrir l'étape rouge et lire la **première erreur significative**.

### 3.3 Job `images` : construire et publier

Ce job utilise une matrice `backend`/`frontend` et `docker/build-push-action`. Il produit des images pour `linux/amd64` et `linux/arm64`.

```yaml
images:
  needs: verify
  runs-on: ubuntu-latest
  permissions:
    contents: read
    packages: write
  strategy:
    matrix:
      component: [backend, frontend]
```

**Pourquoi `needs: verify` ?** Si les contrôles échouent, il n'y a pas de nouvelle publication. **Pourquoi `packages: write` ?** Le `GITHUB_TOKEN` doit pouvoir pousser des images dans GHCR. Les droits d'écriture ne sont pas nécessaires au job de vérification.

**Sur pull request :** construire sans pousser (`push: false`). **Sur push `main` :** se connecter à GHCR et publier une image taguée avec `${{ github.sha }}`.

Exemple de références :

```text
ghcr.io/VOTRE_COMPTE/lab-devops-backend:<SHA_COMPLET>
ghcr.io/VOTRE_COMPTE/lab-devops-frontend:<SHA_COMPLET>
```

**Validation :** les deux jobs `images (backend)` et `images (frontend)` sont verts ; les packages existent dans GHCR. Un package privé nécessite une configuration d'authentification au téléchargement, par exemple un `imagePullSecret` ; la visibilité du dépôt Git ne rend pas automatiquement le package public.

## 4. Préparer Kubernetes pour le déploiement automatique

### 4.1 Créer ou vérifier le namespace

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl create namespace stock-lab04 --dry-run=client -o yaml | kubectl apply -f -
kubectl get namespace stock-lab04
```

Un **namespace** regroupe logiquement les ressources du lab. Ce n'est pas une machine virtuelle et ce n'est pas, à lui seul, une frontière de sécurité complète.

### 4.2 Secrets applicatifs : ne jamais les commiter

Le chart utilise un Secret existant nommé `stock-secrets`, référencé par :

```yaml
existingSecret: stock-secrets
```

Ce Secret contient les paramètres sensibles nécessaires aux composants. **Ne commitez jamais** mots de passe, JWT, tokens GitHub, fichiers `.kube/config` ou clés privées. Pour vérifier la présence sans afficher le contenu :

```bash
kubectl get secret stock-secrets -n stock-lab04
```

Le Secret doit être créé par la procédure sécurisée de votre environnement. Dans le lab réalisé, il existait déjà avant l'automatisation.

### 4.3 RBAC : donner au runner juste les permissions utiles

**RBAC** signifie *Role-Based Access Control*. Il répond à la question : « Quelle identité a le droit de faire quelle action sur quelle ressource ? »

- **ServiceAccount** : identité Kubernetes utilisée par l'automatisation.
- **Role** : permissions dans un namespace.
- **RoleBinding** : relie le ServiceAccount à la Role.
- **ClusterRole/ClusterRoleBinding** : permissions potentiellement à l'échelle du cluster ; inutiles ici.

Exemple pédagogique de RBAC correspondant aux ressources gérées par notre chart :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: stock-lab04-deployer
  namespace: stock-lab04
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: stock-lab04-deployer
  namespace: stock-lab04
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets", "services", "serviceaccounts", "persistentvolumeclaims", "pods", "pods/log", "events"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: stock-lab04-deployer
  namespace: stock-lab04
subjects:
  - kind: ServiceAccount
    name: stock-lab04-deployer
    namespace: stock-lab04
roleRef:
  kind: Role
  name: stock-lab04-deployer
  apiGroup: rbac.authorization.k8s.io
```

Enregistrez ce manifeste dans un dossier d'infrastructure versionné, par exemple `infra/rbac/stock-lab04-deployer.yaml`, puis appliquez-le **en tant qu'administrateur** :

```bash
kubectl apply -f infra/rbac/stock-lab04-deployer.yaml
kubectl auth can-i get ingresses -n stock-lab04 \
  --as=system:serviceaccount:stock-lab04:stock-lab04-deployer
kubectl auth can-i create ingresses -n stock-lab04 \
  --as=system:serviceaccount:stock-lab04:stock-lab04-deployer
kubectl auth can-i create clusterrolebindings \
  --as=system:serviceaccount:stock-lab04:stock-lab04-deployer
```

**Résultat attendu :** `yes`, `yes`, puis `no` pour la création de ClusterRoleBindings. La Role présentée est volontairement large **dans ce namespace** pour les besoins du lab ; elle doit être réduite en production. La permission de gérer les Secrets et les workloads est sensible.

**Incident réellement rencontré :** le job Helm échouait avec `forbidden: cannot get resource "ingresses"`. Le runner savait déployer des Deployments, mais ne pouvait pas consulter l'Ingress ajouté au chart. La correction consistait à ajouter les permissions sur `networking.k8s.io/ingresses` dans la Role, pas à donner au runner le kubeconfig administrateur.

### 4.4 Installer un runner self-hosted

Dans GitHub → dépôt → **Settings → Actions → Runners → New self-hosted runner**, choisissez Linux x64 et suivez **les commandes d'installation affichées par GitHub pour votre dépôt et votre version**. Le token d'enregistrement est temporaire : ne le recopiez pas dans le scénario ni dans Git.

Bonnes pratiques :

1. Créer un utilisateur Linux dédié, par exemple `github-runner`, **sans `sudo`**.
2. Installer le runner dans `/home/github-runner/actions-runner` et son service systemd selon les instructions GitHub.
3. Vérifier qu'il apparaît **Idle** dans GitHub et que son service tourne.
4. Ne pas exécuter de code de pull request non fiable sur ce runner connecté au cluster.

```bash
systemctl list-units --type=service 'actions.runner*'
# Remplacer le nom du service par celui affiché ci-dessus :
systemctl status NOM_DU_SERVICE_RUNNER --no-pager
```

**Pourquoi un runner self-hosted ?** Le job de déploiement doit atteindre l'API Kubernetes de votre réseau privé. Un runner GitHub standard ne dispose généralement pas de cette connectivité. **Attention :** un runner self-hosted exécute du code provenant des workflows du dépôt ; contrôlez qui peut modifier les workflows et pousser sur la branche de déploiement.

### 4.5 Donner au runner un kubeconfig restreint

L'administrateur génère un token pour le ServiceAccount et crée un kubeconfig pointant vers l'API K3s, avec la CA du cluster et cette identité. Le fichier est installé par exemple dans `/home/github-runner/.kube/config`, avec permissions `600`, propriétaire `github-runner`.

**Ne copiez jamais** `/etc/rancher/k3s/k3s.yaml` au runner : ce fichier donne des privilèges administrateur. Les tokens créés avec `kubectl create token` sont temporaires ; dans notre lab, un service et un timer systemd renouvellent périodiquement le token. Cette automatisation doit être surveillée et testée après redémarrage.

Vérifications **sans afficher le token** :

```bash
sudo -u github-runner env KUBECONFIG=/home/github-runner/.kube/config \
  kubectl get deployments -n stock-lab04

sudo -u github-runner env KUBECONFIG=/home/github-runner/.kube/config \
  kubectl auth can-i update deployments -n stock-lab04

sudo -u github-runner env KUBECONFIG=/home/github-runner/.kube/config \
  helm list -n stock-lab04
```

**Validation :** le runner voit les Deployments du namespace et Helm peut lire la release, mais il ne possède pas les droits d'administration globale.

## 5. Comprendre le chart Helm de DSK Stock

Arborescence attendue dans `dsk-stock-lab04-github-actions/chart/` :

```text
chart/
├── Chart.yaml
├── values.yaml
├── values.lab04.yaml
└── templates/
    ├── app.yaml
    ├── postgres.yaml
    ├── migrate.yaml
    ├── secret.yaml
    └── ingress.yaml        # si l'Ingress est géré par le chart
```

**Ce que chaque fichier apporte :**

| Fichier | Rôle |
|---|---|
| `Chart.yaml` | Métadonnées et version du chart. |
| `values.yaml` | Valeurs par défaut. |
| `values.lab04.yaml` | Paramètres du lab : images, stockage, nom de domaine, Secret existant. |
| `app.yaml` | Deployments et Services du frontend/backend. |
| `postgres.yaml` | Base PostgreSQL et volume persistant. |
| `migrate.yaml` | Job appliquant les migrations du schéma. |
| `secret.yaml` | Création conditionnelle d'un Secret, désactivée si `existingSecret` est fourni. |
| `ingress.yaml` | Routage HTTP vers les Services, si présent dans votre version. |

Exemple de valeurs adaptées au lab :

```yaml
frontend:
  apiBaseUrl: "/api"
existingSecret: stock-secrets
ingress:
  enabled: true
  className: traefik
  host: stock.stockage.dsk
```

**Pourquoi `/api` au lieu de `http://IP:PORT/api` ?** Le navigateur contacte le même nom de domaine pour le frontend et l'API. Traefik envoie `/api` au backend et `/` au frontend. Cela évite une dépendance à un port-forward temporaire et simplifie les questions de CORS.

Dans cette application, `entrypoint.sh` génère `/usr/share/nginx/html/env.js` au démarrage du conteneur frontend. Le code React lit cette configuration runtime ; on peut donc changer l'URL de l'API par configuration du Pod, sans nécessairement reconstruire le bundle frontend.

```bash
helm lint dsk-stock-lab04-github-actions/chart \
  -f dsk-stock-lab04-github-actions/chart/values.lab04.yaml

helm template stock dsk-stock-lab04-github-actions/chart \
  -n stock-lab04 \
  -f dsk-stock-lab04-github-actions/chart/values.lab04.yaml \
  > /tmp/stock-lab04-rendered.yaml
```

**Validation :** `helm lint` ne signale aucune erreur. Le rendu contient les ressources attendues, dont un Ingress si vous l'avez activé. Le rendu peut contenir des informations sensibles selon vos valeurs : ne le publiez pas sans contrôle.

## 6. Ajouter le job CD au workflow

Ajoutez le job ci-dessous **au même niveau d'indentation que** `verify` et `images`, sous `jobs:`. Il attend les deux jobs et ne s'exécute que pour un `push` sur `main`.

```yaml
  deploy:
    name: Deploy to K3s
    needs: [verify, images]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: [self-hosted, Linux, X64]
    permissions:
      contents: read
    concurrency:
      group: stock-lab04-deploy
      cancel-in-progress: false
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Verify Kubernetes access
        env:
          KUBECONFIG: /home/github-runner/.kube/config
        run: |
          test "$(kubectl auth can-i update deployments -n stock-lab04)" = "yes"
          helm status stock -n stock-lab04

      - name: Deploy with Helm
        env:
          KUBECONFIG: /home/github-runner/.kube/config
          IMAGE_TAG: ${{ github.sha }}
        run: |
          helm upgrade --install stock \
            ./dsk-stock-lab04-github-actions/chart \
            --namespace stock-lab04 \
            --values ./dsk-stock-lab04-github-actions/chart/values.lab04.yaml \
            --set-string backend.image=ghcr.io/VOTRE_COMPTE/lab-devops-backend:${IMAGE_TAG} \
            --set-string frontend.image=ghcr.io/VOTRE_COMPTE/lab-devops-frontend:${IMAGE_TAG} \
            --atomic --wait --wait-for-jobs --timeout 10m

      - name: Verify deployment
        env:
          KUBECONFIG: /home/github-runner/.kube/config
        run: |
          kubectl rollout status deployment/backend -n stock-lab04 --timeout=180s
          kubectl rollout status deployment/frontend -n stock-lab04 --timeout=180s
          helm status stock -n stock-lab04
```

> **Adapter avant usage :** remplacer `VOTRE_COMPTE` par le propriétaire réel des images. L'étape `helm status` suppose qu'une release `stock` existe déjà : pour un premier déploiement sur un cluster vierge, la remplacer par une vérification d'accès ne nécessitant pas cette release. Le namespace et le Secret doivent exister avant le premier déploiement.

**Explication des options Helm :**

- `upgrade --install` : met à jour la release, ou l'installe si elle n'existe pas.
- `--set-string` : force les tags d'images du commit en cours.
- `--wait` : attend la disponibilité des ressources prises en charge.
- `--wait-for-jobs` : attend également les Jobs, notamment la migration.
- `--atomic` : en cas d'échec de l'upgrade, tente de revenir à la release précédente ; **n'annule pas les changements déjà appliqués à la base de données**.
- `--timeout 10m` : limite l'attente à dix minutes.

**Point de vigilance :** une migration de base doit être compatible avec un éventuel rollback applicatif. Tester la stratégie de migration avant d'utiliser ce modèle en production.

### Vérifier la syntaxe du workflow

`actionlint` permet de détecter certaines erreurs de structure GitHub Actions avant le push. Il ne remplace pas une exécution réelle.

```bash
# Si actionlint est déjà installé :
actionlint .github/workflows/stock-labs.yaml
# Dans l'environnement du lab, il était installé ici :
# /root/go/bin/actionlint .github/workflows/stock-labs.yaml

git diff --check
```

## 7. Versionner et déclencher le pipeline

```bash
# Depuis la racine du dépôt
git status --short
git add .github/workflows/stock-labs.yaml dsk-stock-lab04-github-actions/chart/
git diff --cached --check
git diff --cached --stat
git commit -m "feat(lab04): add GitHub Actions CD deployment to K3s"
git push origin main
```

**Avant `git add` :** vérifier qu'aucun fichier de secrets, kubeconfig, token ou mot de passe n'est présent. Un commit publié dans Git peut rester accessible dans l'historique même après suppression d'un fichier.

### Lire les résultats GitHub Actions

Dans **Actions → DSK Stock labs**, observer :

| Job | Ce qu'il prouve | Résultat attendu |
|---|---|---|
| `verify` | Les contrôles automatisés configurés passent. | Vert |
| `images (backend)` | L'image backend est construite et publiée. | Vert |
| `images (frontend)` | L'image frontend est construite et publiée. | Vert |
| `Deploy to K3s` | Helm déploie puis vérifie les Deployments. | Vert |

**Exemple observé pendant la réalisation :** le commit `70f5977` a réussi les quatre jobs ; Helm est passé de la révision 1 à la révision 2 et les images déployées portaient le SHA complet du commit. C'est un **exemple de preuve**, pas un résultat garanti pour tous les apprenants.

### Contrôles sur K3s

```bash
kubectl get deployments -n stock-lab04
kubectl get pods -n stock-lab04 -o wide
kubectl get jobs -n stock-lab04
kubectl get svc -n stock-lab04
kubectl get ingress -n stock-lab04
helm history stock -n stock-lab04
kubectl get deployments -n stock-lab04 \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image'
```

**À expliquer :**

- `Running` : le processus du Pod est en cours ; cela ne garantit pas le fonctionnement métier.
- `READY 1/1` : la sonde de disponibilité indique que le conteneur est prêt.
- `Job Complete` : la migration a terminé avec succès.
- `helm history` : permet d'identifier la révision déployée et les éventuels rollbacks.
- `IMAGE` : doit contenir le SHA du commit ayant déclenché la livraison.

## 8. Exposer l'application avec Traefik et Ingress

### 8.1 Distinguer les ressources

| Ressource | Fonction | Exemple du lab |
|---|---|---|
| Service `ClusterIP` | Accès interne au cluster | `frontend:80`, `backend:3001`, `postgres:5432` |
| Service `NodePort` | Exposition sur un port d'un nœud | Port éventuellement alloué au Service Traefik |
| Service `LoadBalancer` | Point d'entrée réseau pour Traefik | `10.0.0.50:80` dans l'exemple |
| `Ingress` | Règles HTTP fondées sur hôte et chemin | `stock.stockage.dsk`, `/api`, `/` |
| Contrôleur Traefik | Lit les règles Ingress et route réellement les requêtes | Pod Traefik dans `kube-system` |

**Attention :** Ingress n'est **pas** un type de Service. Un objet Ingress ne transporte pas lui-même le trafic ; c'est le contrôleur Traefik qui applique ses règles. Le fait que Traefik ait des NodePorts alloués n'implique pas que l'utilisateur accède à l'application par ces ports.

### 8.2 Exemple de template Ingress

Le chart peut contenir un template conditionnel tel que :

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: stock
spec:
  ingressClassName: {{ .Values.ingress.className | quote }}
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 3001
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
{{- end }}
```

**Pourquoi `/api` d'abord ?** La route la plus spécifique doit atteindre le backend. Le préfixe `/` permet d'afficher le frontend. Kubernetes définit une priorité des correspondances selon la longueur du chemin et le type de match ; ne dépendez pas uniquement de l'ordre visuel dans le YAML.

### 8.3 Vérifier Traefik et l'Ingress

```bash
kubectl get ingressclass
kubectl get svc traefik -n kube-system
kubectl get ingress stock -n stock-lab04
kubectl describe ingress stock -n stock-lab04

# Test du frontend sans dépendre de la configuration DNS locale :
curl -I --resolve stock.stockage.dsk:80:10.0.0.50 \
  http://stock.stockage.dsk/

# Test d'une route API protégée :
curl -i --resolve stock.stockage.dsk:80:10.0.0.50 \
  http://stock.stockage.dsk/api/products
```

**Résultat attendu :** HTTP `200` sur le frontend. HTTP `401 Unauthorized` sur `/api/products` **sans token** est ici une réponse normale : l'API est atteinte, mais exige une authentification. HTTP `404` sur `/api` seul peut également être normal si aucune route n'y est déclarée. Une erreur `404` n'est donc pas automatiquement une panne d'Ingress.

### 8.4 DNS : pourquoi cela fonctionne sur le serveur mais pas sur le Mac ?

L'Ingress choisit une route à partir du **nom d'hôte HTTP**. Il ne crée pas d'entrée DNS. Votre poste doit résoudre `stock.stockage.dsk` vers l'adresse de Traefik.

**Solution de dépannage locale sur macOS :**

```bash
sudo nano /etc/hosts
```

Ajouter une ligne correspondant **à votre environnement** :

```text
10.0.0.50 stock.stockage.dsk
```

Puis vérifier :

```bash
dscacheutil -q host -a name stock.stockage.dsk
curl -I http://stock.stockage.dsk/
```

**Pour un lab partagé**, il vaut mieux créer un enregistrement dans le DNS interne plutôt que modifier le fichier `hosts` de chaque poste. **Sécurité :** dans cette version du lab, le site est en HTTP ; les identifiants transitent sans chiffrement. Utilisez des comptes de test, ne l'exposez pas publiquement en l'état et prévoyez HTTPS/TLS.

## 9. Recette fonctionnelle : prouver que l'application marche

Un workflow vert et des Pods `Running` ne suffisent pas. Il faut vérifier le parcours réel depuis le navigateur.

1. Ouvrir `http://stock.stockage.dsk` sur un poste capable de résoudre le nom.
2. Se connecter avec un **compte de test**, ou en créer un selon les fonctions disponibles.
3. Ouvrir **Produits** et créer un article de test : nom `Clavier USB`, SKU `CLAV-USB-001`, catégorie `Informatique`, emplacement `Stock A1`, quantité `10`, prix `25`, seuil minimum `3`.
4. Revenir au dashboard : si la base était vide, attendre **1 produit**, **250 €** de valeur totale et **0 alerte**.
5. Actualiser la page et vérifier que le produit reste présent.
6. Tester si possible une entrée/sortie de stock, puis une sortie impossible (quantité supérieure au stock), conformément à la recette métier du parcours.

**Pourquoi actualiser ?** Cela aide à distinguer un simple affichage local dans le navigateur d'une donnée récupérée après une nouvelle requête. Pour démontrer une persistance durable, on peut ensuite redémarrer proprement le Pod applicatif et revérifier les données, sans supprimer le PVC PostgreSQL.

**Preuves à capturer :** capture GitHub Actions, `helm history`, images déployées, Ingress, page produits et dashboard. Ne capturez jamais un mot de passe, un token ou le contenu d'un Secret.

## 10. Incidents réels et méthode de diagnostic

### Incident A — Helm échoue sur `ingresses is forbidden`

**Symptôme :** les jobs `verify` et `images` réussissent, mais `deploy` échoue immédiatement pendant `helm upgrade`.

**Cause :** la Role du ServiceAccount ne contient pas `networking.k8s.io/ingresses`. Helm doit pouvoir lire les ressources qu'il gère.

**Diagnostic :**

```bash
kubectl auth can-i get ingresses -n stock-lab04 \
  --as=system:serviceaccount:stock-lab04:stock-lab04-deployer
kubectl get role stock-lab04-deployer -n stock-lab04 -o yaml
```

**Correction :** mettre à jour la Role versionnée, l'appliquer en tant qu'administrateur, revérifier `can-i`, puis déclencher un nouveau run. **Ne pas** remplacer le kubeconfig restreint par le kubeconfig administrateur.

### Incident B — Le navigateur affiche « Congratulations! Nginx Proxy Manager »

**Symptôme :** la page obtenue ne correspond pas à DSK Stock, alors que l'Ingress semble correct.

**Cause possible observée :** le nom d'hôte pointe vers un autre reverse proxy ou vers une configuration qui ne connaît pas cet hôte. Cela ne prouve pas une panne du backend.

**Diagnostic :** comparer `getent hosts`, `dscacheutil`, l'IP de Traefik, le fichier `hosts` et un test `curl --resolve` ciblant directement Traefik.

### Incident C — `/api` retourne 404

**Explication :** l'API NestJS a un préfixe `/api`, mais pas nécessairement de contrôleur sur la racine `/api`. Tester une **route réelle** telle que `/api/products`. Un `401` sans authentification peut être attendu.

### Incident D — Le frontend utilise encore une ancienne URL API

**Diagnostic :**

```bash
kubectl exec -n stock-lab04 deployment/frontend -- \
  cat /usr/share/nginx/html/env.js
```

**Correction :** vérifier `frontend.apiBaseUrl: "/api"` dans les valeurs Helm et l'environnement du Deployment, puis redéployer. Dans ce projet, `entrypoint.sh` génère le fichier de configuration au démarrage.

### Incident E — `ImagePullBackOff`

**Causes fréquentes :** tag absent, package GHCR privé, autorisation de pull manquante ou problème réseau. Examiner les événements du Pod :

```bash
kubectl describe pod NOM_DU_POD -n stock-lab04
kubectl get events -n stock-lab04 --sort-by=.lastTimestamp
```

### Incident F — Migration de base échouée

```bash
kubectl get jobs -n stock-lab04
kubectl logs -n stock-lab04 job/NOM_DU_JOB
```

Ne relancez pas aveuglément une migration non idempotente. Identifiez l'erreur SQL/Prisma et l'état réel du schéma avant de corriger.



## 10 bis. Atelier guidé : reproduire l'installation complète de A à Z

> Cette partie complète les étapes précédentes. Elle détaille les manipulations qui avaient été seulement résumées : utilisateur du runner, enregistrement GitHub, kubeconfig, rotation de token, Secret PostgreSQL/JWT, résolution DNS sur **la VM et le PC**, et contrôles à chaque étape. **Ne rejouez pas les commandes de création sur un environnement déjà opérationnel sans comprendre leur effet.** Les commandes marquées *première installation* sont destinées à un cluster neuf.

### A. Inventaire et responsabilités des machines

| Élément | Exemple du lab réalisé | Où exécuter ? | Pourquoi ? |
|---|---|---|---|
| VM Ubuntu / K3s | `lab`, `10.0.0.50` | Terminal SSH de la VM | Héberge le cluster, Traefik, l'application et le runner |
| Poste de travail | Mac | Terminal macOS et navigateur | Teste le DNS et l'accès utilisateur |
| Dépôt Git | `prod0751/lab-devops` | GitHub et clone de la VM | Contient les sources, chart et workflow |
| Namespace | `stock-lab04` | Kubernetes | Isole logiquement les ressources de l'exercice |
| Release Helm | `stock` | Kubernetes | Historique des installations/mises à jour |
| Ingress | `stock.stockage.dsk` | Kubernetes + DNS client | Point d'entrée HTTP de l'application |
| Secret | `stock-secrets` | Kubernetes | Identifiants de la base et secret JWT |
| Runner | `lab-k3s-runner` | VM + GitHub | Exécute le job CD dans le réseau privé |

**À expliquer à l'apprenant :** le `root` Linux, le compte GitHub `prod0751`, l'utilisateur système `github-runner` et le ServiceAccount Kubernetes `stock-lab04-deployer` sont **quatre identités différentes**. Les confondre mène à des erreurs de permission et à des failles de sécurité.

```bash
# Sur la VM Ubuntu (avec un compte administrateur)
hostname -I
cat /etc/os-release | head -6
free -h
nproc
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get storageclass
sudo k3s kubectl get pods -n kube-system
```

**Pourquoi ?** Valider que K3s fonctionne avant d'accuser GitHub Actions d'un problème d'infrastructure. Dans notre lab, la StorageClass par défaut était `local-path`. Ce stockage est local au nœud : il ne remplace pas une stratégie de sauvegarde.

### B. Créer le Secret applicatif `stock-secrets` — première installation

Le Secret n'est **pas** un compte utilisateur GitHub et ne sert **pas** à connecter le runner. Il fournit au backend, au Job de migration et à PostgreSQL les variables attendues par leurs conteneurs.

**Important :** les noms exacts des variables doivent être vérifiés dans `chart/templates/postgres.yaml`, `app.yaml`, `migrate.yaml`, les fichiers Prisma et `chart/templates/secret.yaml`. Le chart d'origine utilisait une configuration PostgreSQL et JWT ; ne devinez pas un nom de variable si votre version a changé.

```bash
cd /root/devops-labs/lab-devops
# Lire les noms de variables attendus sans afficher de Secret existant
grep -RnE 'DATABASE_URL|POSTGRES_(USER|PASSWORD|DB)|JWT_SECRET|secretKeyRef|secretRef' \
  dsk-stock-lab04-github-actions/chart/templates \
  dsk-stock-lab04-github-actions/apps/backend \
  --exclude-dir=node_modules --exclude-dir=dist | head -80
```

Pour le **schéma habituel de cette application**, les clés sont `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `DATABASE_URL`, `JWT_SECRET`. Si la vérification précédente confirme ces clés, créer le Secret de façon interactive **sans écrire les valeurs dans Git ni dans l'historique shell** :

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl create namespace stock-lab04 --dry-run=client -o yaml | kubectl apply -f -

# Première installation uniquement : ne remplacez pas un Secret existant
kubectl get secret stock-secrets -n stock-lab04 && echo 'Secret déjà présent : ne pas écraser'

# À exécuter SEULEMENT si le Secret est absent et si les clés ont été confirmées :
read -r -p 'Utilisateur PostgreSQL : ' DB_USER
read -r -p 'Base PostgreSQL : ' DB_NAME
read -r -s -p 'Mot de passe PostgreSQL : ' DB_PASSWORD; echo
JWT_SECRET_VALUE="$(openssl rand -hex 32)"
export DB_USER DB_NAME DB_PASSWORD JWT_SECRET_VALUE
python3 - <<'PYSECRET'
import os
from urllib.parse import quote
u=quote(os.environ['DB_USER'], safe='')
p=quote(os.environ['DB_PASSWORD'], safe='')
d=quote(os.environ['DB_NAME'], safe='')
# DATABASE_URL au format Prisma/PostgreSQL, service postgres dans le même namespace
with open('/tmp/stock-lab04-db-url', 'w') as f:
    f.write(f'postgresql://{u}:{p}@postgres:5432/{d}?schema=public')
os.chmod('/tmp/stock-lab04-db-url', 0o600)
PYSECRET
kubectl -n stock-lab04 create secret generic stock-secrets \
  --from-literal=POSTGRES_USER="$DB_USER" \
  --from-literal=POSTGRES_PASSWORD="$DB_PASSWORD" \
  --from-literal=POSTGRES_DB="$DB_NAME" \
  --from-file=DATABASE_URL=/tmp/stock-lab04-db-url \
  --from-literal=JWT_SECRET="$JWT_SECRET_VALUE"
rm -f /tmp/stock-lab04-db-url
unset DB_USER DB_NAME DB_PASSWORD JWT_SECRET_VALUE
```

**Attention à la sécurité :** les arguments `--from-literal` peuvent être visibles temporairement dans la liste des processus ; pour un environnement sensible, utilisez un mécanisme de gestion de secrets dédié (Vault/ESO, Sealed Secrets ou équivalent) plutôt que cette procédure pédagogique. Un Secret Kubernetes est encodé en base64, **pas chiffré par nature** ; configurez le chiffrement au repos et des RBAC stricts pour un cluster de production. Ne copiez jamais la sortie de `kubectl get secret -o yaml` dans un compte rendu.

```bash
kubectl get secret stock-secrets -n stock-lab04
kubectl get secret stock-secrets -n stock-lab04 \
  -o go-template='{{range $k,$v := .data}}{{printf "%s\n" $k}}{{end}}'
```

**Résultat attendu :** le Secret existe et les cinq **noms** de clés sont présents ; leurs valeurs ne sont pas affichées. Vérifiez que `existingSecret: stock-secrets` est défini dans `values.lab04.yaml`. **Ne relancez pas cette création** si la base est déjà utilisée : modifier le mot de passe sans migration peut empêcher PostgreSQL de démarrer ou le backend de se connecter.

### C. Installer le runner GitHub Actions sur la VM — première installation

**Pourquoi ne pas tout exécuter sur `ubuntu-latest` ?** Les runners GitHub peuvent construire les images, mais ils ne sont pas nécessairement connectés au réseau privé `10.0.0.0/24`. Notre runner auto-hébergé est installé sur la VM K3s. Cette cohabitation est acceptable pour un lab, mais séparer runner et cluster serait préférable dans une architecture plus robuste.

```bash
# Administrateur sur la VM Ubuntu
sudo useradd --create-home --shell /bin/bash github-runner
id github-runner
sudo install -d -o github-runner -g github-runner -m 700 \
  /home/github-runner/actions-runner
# L'utilisateur ne doit pas appartenir au groupe sudo
id -nG github-runner
```

Si `github-runner` existe déjà, **ne recréez pas le compte**. Ouvrez ensuite GitHub → `prod0751/lab-devops` → **Settings → Actions → Runners → New self-hosted runner** → Linux → x64. GitHub affiche des commandes adaptées à la version courante : URL de téléchargement, somme SHA-256, extraction et `config.sh` avec un **token d'enregistrement temporaire**. Reprenez **ces commandes officielles** dans le répertoire du runner, en tant qu'utilisateur non root. Ne collez jamais le token dans un document ou dans Git.

```bash
# L'administrateur ouvre une session sous l'identité du runner
sudo -iu github-runner
cd ~/actions-runner
# Exécuter ici les commandes de téléchargement et de vérification fournies par GitHub.
# Puis la commande ./config.sh --url ... --token ... fournie par GitHub.
# Réponses typiques : nom = lab-k3s-runner ; groupe = Default ; workdir = _work.
exit
```

Installer ensuite le service systemd en tant qu'administrateur (les scripts `svc.sh` sont fournis par le paquet officiel) :

```bash
cd /home/github-runner/actions-runner
sudo ./svc.sh install github-runner
sudo ./svc.sh start
sudo ./svc.sh status
systemctl list-units --type=service 'actions.runner*'
```

**Validation :** dans GitHub, le runner apparaît **Online / Idle**. Sur la VM, le service est `active (running)`. Dans notre réalisation, le service était `actions.runner.prod0751-lab-devops.lab-k3s-runner.service`. Il n'est pas nécessaire de lancer `./run.sh` à la main si systemd gère déjà le runner.

**Risque pédagogique à discuter :** un workflow qui s'exécute sur un runner self-hosted peut lire les fichiers auxquels cet utilisateur a accès. Réservez le job `deploy` aux pushes de confiance sur `main`, contrôlez les collaborateurs et évitez les PR non fiables. Les protections de branche sont recommandées, même si nous avons choisi de ne pas les activer pendant ce lab.

### D. Installer Helm et vérifier kubectl pour le runner

```bash
# Sur la VM
command -v helm && helm version --short
command -v kubectl && kubectl version --client
sudo -u github-runner bash -c 'command -v helm; helm version --short; command -v kubectl'
```

Dans notre lab, Helm était installé dans `/usr/local/bin/helm` et `kubectl` dans `/usr/local/bin/kubectl`. **Attention :** `kubectl version --client` peut afficher un avertissement indiquant qu'il ne peut pas lire `/etc/rancher/k3s/k3s.yaml` ; ce n'est pas nécessairement une panne du client. Pour les commandes réseau du runner, indiquez explicitement **son** `KUBECONFIG` restreint.

### E. Créer l'identité Kubernetes du runner et ses permissions RBAC

Appliquez le manifeste ServiceAccount/Role/RoleBinding de la section 4.3. Le rôle autorise les ressources du chart dans **`stock-lab04` uniquement**, dont `ingresses` du groupe `networking.k8s.io`.

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get serviceaccount stock-lab04-deployer -n stock-lab04
kubectl get role,rolebinding -n stock-lab04
kubectl auth can-i get ingresses -n stock-lab04 \
  --as=system:serviceaccount:stock-lab04:stock-lab04-deployer
kubectl auth can-i create clusterrolebindings \
  --as=system:serviceaccount:stock-lab04:stock-lab04-deployer
```

**Attendu :** `yes` pour les Ingress du namespace et `no` pour les ClusterRoleBindings. Lors de notre déploiement, un oubli des droits Ingress a provoqué `UPGRADE FAILED ... ingresses is forbidden`. Le bon correctif est **d'ajouter la règle RBAC manquante**, pas d'accorder `cluster-admin`.

### F. Construire le kubeconfig restreint du runner

Le kubeconfig est une **configuration de connexion** : URL de l'API Kubernetes, certificat CA, identité et contexte. Un token ServiceAccount de durée limitée remplace les identifiants administrateur. Dans notre cas, runner et API K3s sont sur la même VM : `https://127.0.0.1:6443` est donc approprié. **Sur une VM runner distincte, cette adresse serait fausse** : il faudrait l'adresse réseau de l'API et un certificat compatible.

```bash
# Exécuter en administrateur sur la VM K3s
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
sudo install -d -m 700 -o github-runner -g github-runner /home/github-runner/.kube

# Le token est temporaire ; ne l'affichez pas, ne le commitez pas.
TOKEN="$(kubectl -n stock-lab04 create token stock-lab04-deployer --duration=24h)"
# Extraction du CA du kubeconfig administrateur (la donnée CA est publique)
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > /tmp/stock-lab04-ca.crt

# Création d'un kubeconfig neuf et indépendant du kubeconfig admin
TMP_CONFIG="$(mktemp)"
chmod 600 "$TMP_CONFIG"
kubectl config --kubeconfig="$TMP_CONFIG" set-cluster lab-k3s \
  --server=https://127.0.0.1:6443 \
  --certificate-authority=/tmp/stock-lab04-ca.crt --embed-certs=true
kubectl config --kubeconfig="$TMP_CONFIG" set-credentials stock-lab04-deployer --token="$TOKEN"
kubectl config --kubeconfig="$TMP_CONFIG" set-context stock-lab04 \
  --cluster=lab-k3s --user=stock-lab04-deployer --namespace=stock-lab04
kubectl config --kubeconfig="$TMP_CONFIG" use-context stock-lab04
sudo install -m 600 -o github-runner -g github-runner "$TMP_CONFIG" \
  /home/github-runner/.kube/config
rm -f "$TMP_CONFIG" /tmp/stock-lab04-ca.crt
unset TOKEN TMP_CONFIG

sudo -u github-runner env KUBECONFIG=/home/github-runner/.kube/config \
  kubectl get deployments -n stock-lab04
sudo -u github-runner env KUBECONFIG=/home/github-runner/.kube/config \
  kubectl auth can-i update deployments -n stock-lab04
```

**Résultat attendu :** le runner voit les Deployments et répond `yes` pour leur mise à jour. Le fichier est lisible seulement par `github-runner` (et root). Si `kubectl` échoue avec `Unauthorized`, vérifiez la validité du token ; si `Forbidden`, vérifiez les RBAC ; si `connection refused`, vérifiez K3s et l'URL de l'API.

### G. Renouveler automatiquement le token : script et timer systemd

Un token créé avec `--duration=24h` finit par expirer. Un déploiement qui fonctionne aujourd'hui peut échouer demain. Dans notre lab, nous avons créé un **service systemd root** et un **timer toutes les 12 heures**. Le script suivant reconstitue ce principe sans donner à l'utilisateur runner accès au kubeconfig administrateur.

```bash
sudo tee /usr/local/sbin/refresh-stock-runner-token.sh >/dev/null <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
DEST=/home/github-runner/.kube/config
TMP="$(mktemp /home/github-runner/.kube/.config.XXXXXX)"
trap 'rm -f "$TMP"' EXIT
TOKEN="$(kubectl -n stock-lab04 create token stock-lab04-deployer --duration=24h)"
cp "$DEST" "$TMP"
kubectl config --kubeconfig="$TMP" set-credentials stock-lab04-deployer --token="$TOKEN" >/dev/null
chown github-runner:github-runner "$TMP"
chmod 600 "$TMP"
mv -f "$TMP" "$DEST"
unset TOKEN
SCRIPT
sudo chown root:root /usr/local/sbin/refresh-stock-runner-token.sh
sudo chmod 700 /usr/local/sbin/refresh-stock-runner-token.sh
sudo bash -n /usr/local/sbin/refresh-stock-runner-token.sh
```

```ini
# /etc/systemd/system/stock-runner-token-refresh.service
[Unit]
Description=Refresh restricted K3s token for GitHub runner
After=k3s.service
[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/refresh-stock-runner-token.sh
```

```ini
# /etc/systemd/system/stock-runner-token-refresh.timer
[Unit]
Description=Refresh stock runner token every 12 hours
[Timer]
OnBootSec=5min
OnUnitActiveSec=12h
Persistent=true
Unit=stock-runner-token-refresh.service
[Install]
WantedBy=timers.target
```

Créez ces deux fichiers `.service` et `.timer` avec `sudo nano`, puis :

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now stock-runner-token-refresh.timer
sudo systemctl start stock-runner-token-refresh.service
systemctl status stock-runner-token-refresh.service --no-pager
systemctl list-timers --all | grep stock-runner
sudo journalctl -u stock-runner-token-refresh.service -n 15 --no-pager
sudo -u github-runner env KUBECONFIG=/home/github-runner/.kube/config \
  kubectl get deployments -n stock-lab04
```

**Sécurité et limites :** le timer doit être surveillé ; l'API peut accorder une durée inférieure aux 24 h demandées ; la rotation ne résout pas les problèmes de disponibilité de K3s. Ce montage est une solution **pédagogique de lab**, pas un modèle universel de gestion d'identités en production.

### H. Tester le runner et la chaîne GitHub Actions

Le workflow contient trois jobs logiques : `verify` (tests), `images` (publication GHCR), puis `deploy` (Helm). Le déploiement ne doit s'exécuter qu'après réussite des deux premiers, sur un push vers `main`.

```yaml
# Extrait du job deploy
needs: [verify, images]
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
runs-on: [self-hosted, Linux, X64]
```

**Pourquoi ?** `needs` ordonne les jobs, `if` protège le déploiement contre un déclenchement par pull request et `runs-on` choisit le runner connecté au cluster. La commande Helm `--atomic --wait --wait-for-jobs` attend les workloads et les Jobs ; elle peut restaurer la release en cas d'échec, **mais ne restaure pas automatiquement les migrations SQL déjà exécutées**.

```bash
# Depuis la racine Git
/root/go/bin/actionlint .github/workflows/stock-labs.yaml  # si installé ici
helm lint dsk-stock-lab04-github-actions/chart \
  -f dsk-stock-lab04-github-actions/chart/values.lab04.yaml
git diff --check
git status --short
```

Ne copiez pas aveuglément un chemin `actionlint` qui n'existe pas : `command -v actionlint` permet de vérifier l'installation. Dans GitHub → **Actions**, cherchez les jobs `verify`, `images (backend)`, `images (frontend)`, `Deploy to K3s` et examinez les logs si l'un échoue.

### I. Expliquer la configuration réseau sur la VM ET sur le PC

**Piège fréquent :** créer un Ingress Kubernetes ne crée pas de DNS. Le navigateur a besoin de convertir `stock.stockage.dsk` en `10.0.0.50`. Chaque machine possède sa propre résolution de noms. Modifier `/etc/hosts` **sur la VM ne modifie pas celui du Mac**, et inversement.

**1. Sur la VM Ubuntu (si votre DNS local ne connaît pas le nom) :**

```bash
sudo nano /etc/hosts
# Ajouter : 10.0.0.50 stock.stockage.dsk
getent hosts stock.stockage.dsk
curl -I http://stock.stockage.dsk/
```

**2. Sur le Mac :**

```bash
sudo nano /etc/hosts
# Ajouter la même correspondance : 10.0.0.50 stock.stockage.dsk
dscacheutil -q host -a name stock.stockage.dsk
curl -I http://stock.stockage.dsk/
```

Si la résolution semble mise en cache sous macOS, vous pouvez renouveler le cache DNS :

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

**3. Tester Traefik même sans DNS :**

```bash
curl -I --resolve stock.stockage.dsk:80:10.0.0.50 \
  http://stock.stockage.dsk/
curl -i --resolve stock.stockage.dsk:80:10.0.0.50 \
  http://stock.stockage.dsk/api/products
```

Le paramètre `--resolve` force `curl` à joindre l'IP tout en envoyant le **bon Host HTTP**. Un `200` sur `/` indique que le frontend est accessible ; un `401` sur `/api/products` **sans authentification** prouve que la route API est atteinte et protégée. En revanche, `404` sur `/api` seul peut être normal car le backend ne définit pas forcément cette route.

**4. Vérifier les objets Kubernetes :**

```bash
kubectl get svc traefik -n kube-system
kubectl get ingress stock -n stock-lab04
kubectl describe ingress stock -n stock-lab04
kubectl get svc -n stock-lab04
kubectl exec -n stock-lab04 deployment/frontend -- \
  cat /usr/share/nginx/html/env.js
```

Dans notre lab, Traefik utilisait un Service `LoadBalancer` exposant `10.0.0.50:80` et un NodePort sous-jacent. Le navigateur utilisait **le port 80 du LoadBalancer**, pas directement le NodePort. Le frontend et le backend étaient des Services `ClusterIP` ; l'Ingress distribuait `/` vers le frontend et `/api` vers le backend. Le fichier `env.js` devait contenir `apiBaseUrl: "/api"`, et non une URL de `port-forward` temporaire.

**5. Si le navigateur montre la page « Congratulations! Nginx Proxy Manager » :** le nom pointe probablement vers un autre proxy ou une autre IP que Traefik. Comparez la résolution DNS, l'IP du Service Traefik et le résultat de `curl --resolve`. Ne modifiez pas l'Ingress au hasard.

**6. Pour une infrastructure durable :** remplacez les entrées `/etc/hosts` manuelles par un enregistrement dans le DNS interne (par exemple votre serveur `dnsmasq`). Cela évite de répéter la configuration sur chaque poste. **Le site est actuellement en HTTP** : utilisez uniquement des identifiants de test et ajoutez HTTPS/TLS avant une utilisation réelle.

### J. Vérification de bout en bout et preuve de persistance

```bash
helm history stock -n stock-lab04
kubectl get deployments,pods,jobs,ingress -n stock-lab04
kubectl get deployments -n stock-lab04 \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image'
```

**Résultats observés pendant notre réalisation :** release Helm révision 2 `deployed`, Deployments frontend/backend `1/1`, Job `stock-migrate-2` terminé, images GHCR taguées avec le SHA `70f597795dcc17950018178693d0e9ff623edda6`. Ces valeurs sont **des preuves de notre session**, pas des valeurs garanties pour les prochains apprenants.

Depuis le Mac : ouvrez `http://stock.stockage.dsk`, créez un **compte de test**, connectez-vous, ajoutez `Clavier USB`, SKU `CLAV-USB-001`, catégorie `Informatique`, emplacement `Stock A1`, quantité `10`, prix `25`, seuil `3`. Vérifiez le dashboard : `1` produit, `250 €` de valeur, `0` alerte, si la base était vide. Rechargez la page ; pour tester plus solidement la persistance, redémarrez **seulement** le Pod backend, puis rechargez. Ne supprimez ni le PVC ni le namespace.

```bash
kubectl rollout restart deployment/backend -n stock-lab04
kubectl rollout status deployment/backend -n stock-lab04 --timeout=180s
kubectl get pvc -n stock-lab04
```

**Ce qu'il faut expliquer :** la persistance des produits est assurée par PostgreSQL et son volume, pas par le conteneur frontend. Un dashboard correct après un rechargement est un bon test fonctionnel, mais une stratégie de sauvegarde et de restauration doit encore être prévue.

### K. Diagnostic méthodique des incidents rencontrés

| Symptôme | Interprétation possible | Premier contrôle |
|---|---|---|
| `ingresses is forbidden` dans Helm | RBAC du ServiceAccount incomplet | `kubectl auth can-i get ingresses --as=...` |
| `Unauthorized` sur Kubernetes | Token expiré / invalide | service de rotation et kubeconfig du runner |
| `ImagePullBackOff` | Image privée, tag erroné ou problème réseau | `kubectl describe pod`, GHCR et `imagePullSecrets` |
| `curl: Could not resolve host` | DNS ou `/etc/hosts` du poste non configuré | `getent hosts` (Linux), `dscacheutil` (Mac) |
| Page Nginx Proxy Manager | Requête envoyée au mauvais reverse proxy | `curl --resolve`, DNS et IP Traefik |
| `GET /api` retourne 404 | Route `/api` absente | Tester une vraie route telle que `/api/products` |
| `/api/products` retourne 401 | Route accessible, authentification requise | Tester depuis la session authentifiée |
| Page OK mais API cassée | `env.js` ou routage Ingress incorrect | Lire `env.js`, inspecter l'Ingress |
| Migration en erreur | Schéma DB, Secret, réseau ou image | `kubectl logs job/stock-migrate-N` |
| `helm upgrade` échoue | Ressource manquante, RBAC, Pod/Job non prêt | Logs Actions, `helm history`, `kubectl describe` |

### L. Ce qu'un apprenti doit être capable d'expliquer à l'oral

1. Pourquoi avons-nous **deux types de runners** dans le même workflow ?
2. Pourquoi le runner Linux **n'est pas root** et n'utilise pas le kubeconfig administrateur ?
3. Quelle différence entre le compte `github-runner` et le ServiceAccount `stock-lab04-deployer` ?
4. Pourquoi faut-il **renouveler le token** ? Qu'arrive-t-il s'il expire ?
5. Que contient `stock-secrets` ? Pourquoi ne doit-il pas être commité ?
6. Pourquoi la VM et le Mac ont-ils chacun leur propre fichier `/etc/hosts` ?
7. Quelle différence entre **ClusterIP**, **NodePort**, **LoadBalancer**, **Ingress** et **Ingress Controller** ?
8. Pourquoi `GET /api/products` renvoie-t-il `401` avant connexion ?
9. Pourquoi taguer les images avec le SHA Git plutôt qu'uniquement `latest` ?
10. Que fait Helm avec `--atomic --wait --wait-for-jobs` ? Quelle est la limite concernant les migrations SQL ?
11. Pourquoi un Pod `Running` ne suffit-il pas à valider l'application ?
12. Quelle preuve démontre que le produit reste enregistré après un redémarrage du backend ?

**Livrable supplémentaire :** capture du runner `Online`, capture des quatre jobs GitHub Actions réussis, sorties de `helm history` et `kubectl get`, captures du DNS sur VM et Mac, preuve d'une création de produit, et une explication écrite de deux incidents corrigés. **Masquez systématiquement** tokens, JWT, mots de passe, fichiers kubeconfig et identifiants personnels.

## 11. Exercices pour apprendre, pas seulement reproduire

**Exercice 1 — Bloquer une mauvaise livraison.** Dans une branche de test, introduisez volontairement une erreur TypeScript, ouvrez une pull request et constatez que `verify` échoue. Vérifiez qu'aucun déploiement n'a eu lieu. Corrigez l'erreur et relancez.

**Exercice 2 — Retrouver une version.** Comparez `git rev-parse HEAD`, le SHA du run GitHub Actions et les tags d'images affichés par `kubectl get deployments`. Expliquez pourquoi ces valeurs doivent correspondre après le déploiement.

**Exercice 3 — Comprendre les droits.** Vérifiez que le ServiceAccount peut gérer un Ingress dans `stock-lab04` mais ne peut pas créer un `ClusterRoleBinding`. Expliquez le principe du moindre privilège.

**Exercice 4 — Diagnostiquer une erreur DNS.** Retirez temporairement l'entrée locale `hosts` (si elle est la seule source de résolution), observez l'échec DNS, puis utilisez `curl --resolve` pour montrer que Traefik reste accessible. Restaurez l'entrée ensuite.

**Exercice 5 — Préparer la production.** Proposez un plan d'amélioration : DNS centralisé, HTTPS, restriction des droits, surveillance du renouvellement de token, sauvegarde PostgreSQL, migration compatible rollback et contrôle des changements sur `main`.

## 12. Questions de compréhension avec éléments de réponse

**1. Pourquoi les images ne sont-elles pas publiées si `verify` échoue ?**  
Parce que `images` déclare `needs: verify` : GitHub Actions n'exécute ce job que si sa dépendance réussit dans le déroulement normal du workflow.

**2. Pourquoi utiliser un runner self-hosted pour le déploiement ?**  
Pour accéder à l'API privée de K3s sans exposer cette API sur Internet. Cela ajoute une responsabilité de sécurisation du runner.

**3. Quelle différence entre un ServiceAccount et une Role ?**  
Le premier représente une identité ; la seconde décrit des permissions. Le RoleBinding relie les deux.

**4. Pourquoi un Ingress ne suffit-il pas à rendre un site accessible ?**  
Il faut un contrôleur qui applique ses règles, un point d'entrée réseau vers ce contrôleur et une résolution DNS fonctionnelle.

**5. Que prouve un HTTP 401 sur `/api/products` ?**  
Que la requête atteint une route protégée et qu'une authentification est demandée. Ce n'est pas la preuve que toutes les opérations API fonctionnent.

**6. Pourquoi `--atomic` ne garantit-il pas le rollback de PostgreSQL ?**  
Parce que Helm gère les ressources Kubernetes ; les migrations ont déjà pu modifier durablement les données ou le schéma.

**7. Un Pod `Running` signifie-t-il que l'application fonctionne ?**  
Non. Il faut au minimum les probes, les logs et une recette HTTP/métier depuis le poste utilisateur.

## 13. Livrables et critères d'acceptation

- [ ] Fork personnel et `origin` vérifié.
- [ ] Workflow `.github/workflows/stock-labs.yaml` versionné.
- [ ] `verify`, `images (backend)`, `images (frontend)` et `deploy` réussis.
- [ ] Références GHCR avec le SHA du commit documentées.
- [ ] Runner self-hosted identifié et permissions Kubernetes limitées au namespace.
- [ ] Manifeste RBAC conservé dans le dépôt sans token.
- [ ] Helm release `stock` en statut `deployed`.
- [ ] Pods frontend/backend/PostgreSQL prêts et migration terminée.
- [ ] Ingress accessible via le domaine de lab.
- [ ] Création de produit et affichage de la valeur totale validés.
- [ ] Incident rencontré, cause et correction expliqués dans `COMPTE-RENDU.md`.
- [ ] Aucune donnée sensible dans le dépôt, les captures ou le compte rendu.

**Critère de réussite principal :** l'apprenant sait expliquer la chaîne **commit → tests → images → GHCR → runner → Helm → K3s → Ingress → navigateur**, puis démontrer que l'application traite une opération métier.

## 14. Terminer le lab sans détruire les données

Le namespace contient PostgreSQL et un PVC : **ne supprimez pas le namespace ou le PVC** pour simplement arrêter les exercices. Fermez les éventuels `port-forward` avec `Ctrl+C`. Vous pouvez conserver le runner et les workloads pour le lab suivant, mais documentez leur consommation de ressources et leur exposition réseau.

Si vous désactivez le workflow ou le runner, faites-le volontairement et documentez la procédure. Évitez de supprimer les packages GHCR tant qu'une release Kubernetes utilise leurs images.

## 15. Ressources utiles

- [Documentation GitHub Actions](https://docs.github.com/en/actions)
- [Publier des images Docker avec GitHub Actions](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)
- [Documentation des runners self-hosted](https://docs.github.com/en/actions/hosting-your-own-runners)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Helm : upgrade](https://helm.sh/docs/helm/helm_upgrade/)
