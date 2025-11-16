# Rapport Technique Détaillé : Déploiement d'une Application de Gestion de Flotte sur Kubernetes

## Table des Matières

1. [Guide de Démarrage Rapide](#guide-de-démarrage-rapide-)
2. [Introduction](#1-introduction)
3. [Architecture Globale de l'Application](#2-architecture-globale-de-lapplication)
4. [Explication Détaillée des Fichiers YAML](#3-explication-détaillée-des-fichiers-yaml)
5. [Problèmes Rencontrés et Solutions](#4-problèmes-rencontrés-et-solutions)
6. [Guide de Déploiement Étape par Étape](#5-guide-de-déploiement-étape-par-étape)
7. [Accès à l'Application](#6-accès-à-lapplication)
8. [Conclusion](#7-conclusion)

---

## Guide de Démarrage Rapide ⚡

**Vous voulez lancer l'application immédiatement ?** Suivez ces étapes :

### Prérequis

1. **Docker Desktop** installé et démarré
2. **Kubernetes activé** dans Docker Desktop (Settings → Kubernetes → Enable Kubernetes)
3. **kubectl** installé (inclus avec Docker Desktop)
4. **Mémoire allouée** : Au moins 6 Go dans Docker Desktop (Settings → Resources)

### Étapes de Déploiement (5 minutes)

**Étape 1 : Naviguer vers le répertoire du projet**
```powershell
cd "C:\Users\harry\Documents\kubernetes project"
```

**Étape 2 : Vérifier que Kubernetes fonctionne**
```powershell
kubectl cluster-info
# Vous devriez voir : "Kubernetes control plane is running at..."
```

**Étape 3 : Déployer toutes les ressources**
```powershell
# Déployer tous les fichiers YAML dans l'ordre
kubectl apply -f namespace.yaml
kubectl apply -f mongodb-pvc.yaml
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f queue-deployment.yaml
kubectl apply -f position-simulator-deployment.yaml
kubectl apply -f position-tracker-deployment.yaml
kubectl apply -f api-gateway-deployment.yaml
kubectl apply -f webapp-deployment.yaml

# OU en une seule commande (déploie tout le répertoire)
kubectl apply -f .
```

**Étape 4 : Vérifier que tous les Pods sont Running**
```powershell
kubectl get pods

# Attendez 2-3 minutes que tous les Pods passent à "Running"
# Résultat attendu : 10 Pods avec STATUS "Running" et READY "1/1"
```

**Si les Pods prennent du temps :** C'est normal, Docker télécharge les images. Attendez que tous affichent `1/1 Running`.

### Accès à l'Application

**Ouvrez 3 terminaux PowerShell séparés :**

**Terminal 1 : Interface Web (Principal)**
```powershell
kubectl port-forward service/fleetman-webapp 3000:80
```
- Laissez ce terminal ouvert
- Ouvrez votre navigateur : **http://localhost:3000**
- Vous verrez une carte avec des véhicules se déplaçant en temps réel

**Terminal 2 : API Backend**
```powershell
kubectl port-forward service/fleetman-api-gateway 8080:8080
```
- Laissez ce terminal ouvert
- Testez dans un navigateur : **http://localhost:8080/api/vehicles**
- Vous verrez des données JSON des véhicules

**Terminal 3 : Monitoring (Optionnel)**
```powershell
# Voir les logs en temps réel du simulateur
kubectl logs -f deployment/fleetman-position-simulator
```

### Vérification Rapide du Déploiement

```powershell
# 1. Vérifier que tous les Pods sont Running
kubectl get pods
# Résultat attendu : 10 Pods avec "1/1 Running"

# 2. Vérifier que tous les Services sont créés
kubectl get svc
# Résultat attendu : 7 services listés

# 3. Tester l'API Gateway
curl http://localhost:8080/api/vehicles
# Résultat attendu : JSON avec liste de véhicules
```

### Dépannage Rapide

**Problème 1 : Pod en "CrashLoopBackOff" ou "Error"**
```powershell
# Voir les logs du Pod
kubectl logs <nom-du-pod>

# Voir les détails du Pod
kubectl describe pod <nom-du-pod>

# Causes fréquentes :
# - Manque de mémoire : Augmentez la RAM de Docker Desktop à 6-8 Go
# - Image non téléchargée : Attendez le téléchargement
```

**Problème 2 : "No resources found in default namespace"**
```powershell
# Vérifier dans quel namespace sont les ressources
kubectl get pods --all-namespaces

# Si elles sont dans "fleetman", il faut les voir avec :
kubectl get pods -n fleetman

# OU modifier les fichiers YAML pour utiliser "default" namespace
```

**Problème 3 : Port-forward ne fonctionne pas**
```powershell
# Vérifier que le Service existe
kubectl get svc fleetman-webapp

# Vérifier que des Pods sont prêts
kubectl get pods -l app=fleetman-webapp

# Réessayer le port-forward avec un port différent
kubectl port-forward service/fleetman-webapp 8888:80
```

**Problème 4 : La webapp affiche une page blanche**
```powershell
# Vérifier les logs de la webapp
kubectl logs -l app=fleetman-webapp

# Vérifier que l'API Gateway est accessible
kubectl get endpoints fleetman-api-gateway

# Redémarrer la webapp
kubectl rollout restart deployment fleetman-webapp
```

### Commandes Essentielles

```powershell
# Voir l'état de tous les Pods
kubectl get pods

# Voir les logs d'un Pod en temps réel
kubectl logs -f <nom-pod>

# Voir les détails d'un Pod (pour debugging)
kubectl describe pod <nom-pod>

# Redémarrer un Deployment
kubectl rollout restart deployment fleetman-webapp

# Supprimer toutes les ressources (nettoyer)
kubectl delete -f .

# Voir l'utilisation des ressources
kubectl top pods
```

### Architecture Rapide

L'application fonctionne comme ceci :
1. **Position Simulator** génère des positions GPS fictives
2. **ActiveMQ Queue** stocke ces positions dans une file de messages
3. **Position Tracker** consomme les messages et les stocke dans **MongoDB**
4. **API Gateway** expose les données via une API REST
5. **WebApp** affiche les véhicules sur une carte interactive

**Pour comprendre en détail chaque composant et commande, continuez la lecture ci-dessous.**

---

## 1. Introduction

### 1.1 Qu'est-ce que Kubernetes ?

Imaginez que vous avez une grande entreprise avec plusieurs départements : comptabilité, marketing, production, etc. Chaque département a besoin de bureaux, d'ordinateurs, d'électricité, et doit pouvoir communiquer avec les autres départements. 

**Kubernetes, c'est comme un gestionnaire d'immeuble intelligent** qui :
- Alloue automatiquement des bureaux (ressources serveur) aux départements (applications)
- S'assure que chaque département a ce dont il a besoin (mémoire, CPU)
- Remplace automatiquement les équipements défectueux (redémarre les conteneurs)
- Gère la communication entre départements (réseau interne)
- Permet aux clients externes de visiter certains départements (exposition des services)

Dans notre cas, nous déployons une **application de gestion de flotte de véhicules** qui simule le suivi en temps réel de véhicules, comme une application de livraison ou de taxis.

### 1.2 Objectif du Projet

Ce projet déploie une architecture microservices complète comprenant :
- Une **base de données MongoDB** pour stocker les données
- Une **file de messages ActiveMQ** pour la communication asynchrone
- Un **simulateur de positions** qui génère des positions GPS fictives
- Un **tracker de positions** qui consomme et traite ces positions
- Une **API Gateway** qui expose les données via une API REST
- Une **application web** (interface utilisateur) pour visualiser la flotte

### 1.3 Pourquoi utiliser Kubernetes ?

Sans Kubernetes, vous devriez :
1. Installer manuellement chaque application sur un serveur
2. Configurer manuellement les communications réseau
3. Surveiller constamment les applications et les redémarrer manuellement si elles plantent
4. Gérer manuellement l'ajout de ressources si la charge augmente

**Avec Kubernetes**, tout cela est automatisé. Vous décrivez simplement ce que vous voulez (dans des fichiers YAML), et Kubernetes s'occupe du reste.

---

## 2. Architecture Globale de l'Application

### 2.1 Vue d'Ensemble des Composants

Notre application est composée de **6 microservices** qui travaillent ensemble :

```
┌─────────────────────────────────────────────────────────────┐
│                    UTILISATEUR                               │
│                    (Navigateur Web)                          │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP (port 30080)
                         ▼
┌────────────────────────────────────────────────────────────┐
│                   FLEETMAN-WEBAPP                           │
│            (Interface utilisateur Angular)                  │
│                  Port interne: 80                           │
└────────────────────────┬───────────────────────────────────┘
                         │
                         │ Appels HTTP
                         ▼
┌────────────────────────────────────────────────────────────┐
│                FLEETMAN-API-GATEWAY                         │
│              (Spring Boot - API REST)                       │
│                  Port interne: 8080                         │
│                  Port externe: 30020                        │
└─────────────┬──────────────────────────┬───────────────────┘
              │                          │
              │                          │
              ▼                          ▼
┌─────────────────────────┐   ┌──────────────────────────────┐
│ FLEETMAN-POSITION-      │   │  FLEETMAN-POSITION-TRACKER   │
│      SIMULATOR          │   │     (Spring Boot)            │
│   (Génère positions)    │   │   Consomme & Stocke données  │
│   Port interne: 8080    │   │   Port interne: 8080         │
└──────────┬──────────────┘   └────────┬─────────────────────┘
           │                           │
           │ Publie                    │ Consomme
           │ messages                  │ messages
           ▼                           ▼
┌──────────────────────────────────────────────────────────────┐
│              FLEETMAN-QUEUE (ActiveMQ)                       │
│            File de messages asynchrone                       │
│         Ports: 8161 (admin), 61616 (messaging)              │
└──────────────────────────────────────────────────────────────┘

                         │
                         │ Stockage persistant
                         ▼
┌──────────────────────────────────────────────────────────────┐
│              FLEETMAN-MONGODB                                │
│            Base de données NoSQL                             │
│                  Port: 27017                                 │
│            Volume persistant: /data/db                       │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Rôle de Chaque Composant

#### MongoDB (Base de données)
**Rôle** : Stocker toutes les positions des véhicules de manière persistante.

**Analogie** : C'est comme un coffre-fort bancaire qui conserve toutes les données même si l'électricité est coupée. Les données restent là, intactes.

**Caractéristiques** :
- **1 seule instance** (replica: 1) car c'est une base de données stateful
- **Volume persistant** pour ne pas perdre les données si le conteneur redémarre
- Communication **uniquement interne** (ClusterIP) - personne de l'extérieur ne peut y accéder directement

#### ActiveMQ Queue (File de messages)
**Rôle** : Jouer l'intermédiaire entre le simulateur qui génère des positions et le tracker qui les consomme.

**Analogie** : Imaginez une boîte aux lettres. Le simulateur dépose des lettres (positions GPS), et le tracker vient les récupérer quand il est prêt. Si le tracker est occupé, les lettres s'accumulent, elles ne sont pas perdues.

**Caractéristiques** :
- **1 seule instance** (pour garantir l'ordre des messages)
- **Port 61616** : pour les messages (communication interne)
- **Port 8161** : interface web d'administration (optionnel)

#### Position Simulator (Générateur de données)
**Rôle** : Générer des positions GPS fictives de véhicules et les publier dans la queue ActiveMQ.

**Analogie** : C'est comme un générateur de trafic dans un simulateur de ville. Il crée des véhicules virtuels qui se déplacent et envoie leurs coordonnées.

**Caractéristiques** :
- **2 réplicas** pour la haute disponibilité (si l'un tombe, l'autre continue)
- Application Spring Boot
- Lit la variable d'environnement `SPRING_PROFILES_ACTIVE=production-microservice`

#### Position Tracker (Consommateur et stockeur)
**Rôle** : Récupérer les positions depuis la queue, les traiter, et les stocker dans MongoDB.

**Analogie** : C'est comme un employé qui prend les lettres de la boîte aux lettres et les archive dans le coffre-fort (MongoDB).

**Caractéristiques** :
- **2 réplicas** pour la résilience
- Exposé en **NodePort** (30010) pour accès externe (tests ou monitoring)
- Se connecte à MongoDB pour persister les données

#### API Gateway (Passerelle API)
**Rôle** : Exposer les données via une API REST. C'est le point d'entrée pour les requêtes HTTP externes.

**Analogie** : C'est comme un guichet d'accueil dans un hôpital. Tous les visiteurs passent par là, et le guichet les redirige vers le bon service.

**Caractéristiques** :
- **2 réplicas** pour supporter la charge
- Exposé en **NodePort** (30020)
- Communique avec le Position Tracker pour récupérer les données

#### WebApp (Interface utilisateur)
**Rôle** : Application web (HTML/CSS/JavaScript) qui affiche une carte avec les véhicules en temps réel.

**Analogie** : C'est la vitrine du magasin, ce que l'utilisateur final voit et avec quoi il interagit.

**Caractéristiques** :
- **2 réplicas** pour la disponibilité
- Serveur web Nginx
- Exposé en **NodePort** (30080) - port principal d'accès utilisateur
- Communique avec l'API Gateway

### 2.3 Concepts Kubernetes Utilisés

#### Namespace
**Qu'est-ce que c'est ?** Un espace de noms logique pour grouper des ressources.

**Analogie** : Comme des dossiers sur votre ordinateur. Au lieu de mettre tous vos fichiers à la racine, vous créez un dossier "Fleetman" et mettez tout dedans.

**Dans notre projet** : Nous avons déployé dans le namespace `default` (pour des raisons de compatibilité avec le WebApp, expliquées plus tard).

#### Deployment
**Qu'est-ce que c'est ?** Une ressource Kubernetes qui gère des Pods (conteneurs) répliqués.

**Analogie** : Imaginez un chef d'équipe qui s'assure que vous avez toujours 2 employés disponibles. Si un employé tombe malade, il en embauche un autre automatiquement.

**Dans notre projet** : Tous nos services utilisent des Deployments (sauf rien car tout est en Deployment ici).

#### Service
**Qu'est-ce que c'est ?** Un point d'accès réseau stable pour un ensemble de Pods.

**Analogie** : Les employés (Pods) changent, mais le numéro de téléphone du service (Service) reste le même. Peu importe qui répond, vous composez toujours le même numéro.

**Types utilisés** :
- **ClusterIP** : Accès uniquement interne (MongoDB, Queue, Simulator)
- **NodePort** : Accès externe via un port sur le nœud (WebApp, API Gateway, Tracker)

#### PersistentVolumeClaim (PVC)
**Qu'est-ce que c'est ?** Une demande de stockage persistant.

**Analogie** : C'est comme louer un box de stockage. Vous demandez un espace de 5Go, et Kubernetes vous en fournit un. Même si vous déménagez (conteneur redémarre), vos affaires restent dans le box.

**Dans notre projet** : Utilisé pour MongoDB pour stocker les données de manière persistante.

---

## 3. Explication Détaillée des Fichiers YAML

### 3.1 namespace.yaml - Définition de l'Espace de Noms

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: default
  labels:
    name: default
```

**Explication ligne par ligne** :

- `apiVersion: v1` : Version de l'API Kubernetes utilisée. La v1 est la version stable pour les Namespaces.
- `kind: Namespace` : Type de ressource que nous créons. Ici, un Namespace (espace de noms).
- `metadata:` : Section contenant les métadonnées de la ressource.
  - `name: default` : Le nom du namespace. **Important** : Nous utilisons "default" au lieu de "fleetman" car l'application webapp a son fichier de configuration nginx qui cherche l'API Gateway dans le namespace "default". C'était un choix imposé par la configuration de l'image Docker.
  - `labels:` : Étiquettes pour identifier la ressource.
    - `name: default` : Label simple pour identifier ce namespace.

**Pourquoi utiliser des namespaces ?**
- **Isolation** : Séparer les environnements (dev, test, prod)
- **Organisation** : Grouper les ressources liées
- **Sécurité** : Appliquer des politiques d'accès par namespace
- **Quotas** : Limiter les ressources CPU/mémoire par namespace

**Note importante** : Dans un projet réel, nous aurions créé un namespace "fleetman", mais nous avons dû utiliser "default" à cause d'une configuration hardcodée dans le webapp.

---

### 3.2 mongodb-pvc.yaml - Volume Persistant pour MongoDB

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: default
  labels:
    app: fleetman-mongodb
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**Explication ligne par ligne** :

- `apiVersion: v1` : Version stable de l'API pour les PersistentVolumeClaims.
- `kind: PersistentVolumeClaim` : Type de ressource - une demande de volume persistant.
- `metadata:` : Métadonnées de la ressource.
  - `name: mongodb-pvc` : Nom unique du PVC. Ce nom sera utilisé dans le Deployment MongoDB.
  - `namespace: default` : Namespace où créer ce PVC.
  - `labels:` : Étiquettes pour identifier ce PVC.
    - `app: fleetman-mongodb` : Indique que ce PVC est lié à MongoDB.

- `spec:` : Spécification du volume demandé.
  - `accessModes:` : Comment le volume peut être accédé.
    - `- ReadWriteOnce` : Un seul nœud peut monter ce volume en lecture/écriture. Parfait pour MongoDB qui ne peut avoir qu'une seule instance écrivant dans la base.
  
  **Autres modes possibles** :
  - `ReadOnlyMany` : Plusieurs nœuds peuvent lire, personne ne peut écrire.
  - `ReadWriteMany` : Plusieurs nœuds peuvent lire et écrire (nécessite un système de fichiers réseau).

  - `resources:` : Ressources demandées pour le volume.
    - `requests:` : Demande minimum.
      - `storage: 5Gi` : Demande 5 Giga-octets d'espace disque.

**Analogie** : C'est comme réserver un box de stockage de 5 mètres cubes. Kubernetes trouve un espace de stockage disponible et vous le réserve. Même si MongoDB redémarre 100 fois, les données restent dans ce box.

**Pourquoi MongoDB a besoin d'un PVC ?**
Sans PVC, si le conteneur MongoDB redémarre, **toutes les données sont perdues**. Avec un PVC, les données survivent aux redémarrages, mises à jour, et même aux migrations vers d'autres nœuds.

---

### 3.3 mongodb-deployment.yaml - Déploiement de MongoDB

Ce fichier contient **deux ressources** : un Deployment et un Service.

#### 3.3.1 Deployment MongoDB

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleetman-mongodb
  namespace: default
  labels:
    app: fleetman-mongodb
    component: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fleetman-mongodb
  template:
    metadata:
      labels:
        app: fleetman-mongodb
        component: database
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.23
        ports:
        - containerPort: 27017
          name: mongodb
          protocol: TCP
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: mongodb-persistent-storage
          mountPath: /data/db
      volumes:
      - name: mongodb-persistent-storage
        persistentVolumeClaim:
          claimName: mongodb-pvc
```

**Explication détaillée** :

- `apiVersion: apps/v1` : Version de l'API pour les Deployments.
- `kind: Deployment` : Type de ressource - un Deployment gère des Pods répliqués.
- `metadata:` : Métadonnées du Deployment.
  - `name: fleetman-mongodb` : Nom unique du Deployment.
  - `namespace: default` : Namespace de déploiement.
  - `labels:` : Étiquettes pour identifier ce Deployment.
    - `app: fleetman-mongodb` : Application principale.
    - `component: database` : Rôle dans l'architecture (base de données).

- `spec:` : Spécification du Deployment.
  - `replicas: 1` : **Nombre de copies (Pods) à maintenir en permanence.**
    
    **Pourquoi 1 seul ?** MongoDB est une base de données stateful (avec état). Avoir plusieurs instances nécessiterait de configurer un ReplicaSet MongoDB, ce qui est complexe. Pour ce projet, 1 instance suffit.

  - `selector:` : Comment le Deployment identifie les Pods qu'il gère.
    - `matchLabels:` : Sélectionne les Pods ayant ces labels.
      - `app: fleetman-mongodb` : Le Deployment gère tous les Pods avec ce label.

  - `template:` : **Modèle** pour créer les Pods. C'est le plan de construction.
    - `metadata:` : Métadonnées des Pods créés.
      - `labels:` : Labels appliqués à chaque Pod créé.
        - `app: fleetman-mongodb` : Doit correspondre au selector ci-dessus.
        - `component: database` : Label supplémentaire pour l'organisation.

    - `spec:` : Spécification du Pod (ce qui tourne dedans).
      - `containers:` : Liste des conteneurs dans le Pod. Ici, un seul : MongoDB.
        - `- name: mongodb` : Nom du conteneur (pour les logs, debug).
        - `image: mongo:3.6.23` : **Image Docker à utiliser**. 
          - `mongo` : Nom de l'image officielle MongoDB sur Docker Hub.
          - `3.6.23` : Version spécifique (important pour la stabilité).
        
        - `ports:` : Ports exposés par le conteneur.
          - `- containerPort: 27017` : MongoDB écoute sur le port 27017 (port standard).
          - `name: mongodb` : Nom du port (pour référence).
          - `protocol: TCP` : Protocole réseau utilisé.

        - `resources:` : **Ressources CPU et mémoire allouées**.
          - `requests:` : **Minimum garanti** par Kubernetes.
            - `memory: "256Mi"` : 256 Méga-octets de RAM garantis.
            - `cpu: "250m"` : 250 milli-CPU = 0,25 CPU garanti.
          - `limits:` : **Maximum autorisé**. Si dépassé, le Pod est tué (OOMKilled).
            - `memory: "512Mi"` : Maximum 512 Mo de RAM.
            - `cpu: "500m"` : Maximum 0,5 CPU.

        **Analogie des ressources** : Imaginez un employé. Les `requests` sont son salaire minimum garanti. Les `limits` sont le maximum qu'il peut gagner avec les heures supplémentaires. S'il dépasse les limites, il est licencié (Pod tué).

        - `volumeMounts:` : **Montage de volumes** dans le conteneur.
          - `- name: mongodb-persistent-storage` : Nom du volume (défini plus bas).
          - `mountPath: /data/db` : **Chemin dans le conteneur** où monter le volume. C'est là que MongoDB stocke ses données par défaut.

      - `volumes:` : **Définition des volumes** disponibles pour le Pod.
        - `- name: mongodb-persistent-storage` : Nom du volume (référencé dans volumeMounts).
        - `persistentVolumeClaim:` : Type de volume - un PVC.
          - `claimName: mongodb-pvc` : **Nom du PVC créé précédemment**. Kubernetes lie ce volume au PVC.

**Flux complet** :
1. Vous créez un PVC demandant 5 Gi de stockage.
2. Kubernetes trouve un volume disponible et le réserve.
3. Le Deployment crée un Pod MongoDB.
4. Kubernetes monte le volume PVC dans `/data/db` du conteneur.
5. MongoDB écrit ses données dans `/data/db`, qui est en réalité le volume persistant.
6. Si le Pod redémarre, le volume reste, et MongoDB retrouve ses données.

#### 3.3.2 Service MongoDB

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-mongodb
  namespace: default
  labels:
    app: fleetman-mongodb
    component: database
spec:
  type: ClusterIP
  selector:
    app: fleetman-mongodb
  ports:
  - port: 27017
    targetPort: 27017
    protocol: TCP
    name: mongodb
```

**Explication détaillée** :

- `apiVersion: v1` : Version de l'API pour les Services.
- `kind: Service` : Type de ressource - un Service réseau.
- `metadata:` : Métadonnées du Service.
  - `name: fleetman-mongodb` : **Nom DNS du service**. Les autres Pods peuvent accéder à MongoDB via `fleetman-mongodb.default.svc.cluster.local` ou simplement `fleetman-mongodb`.
  - `namespace: default` : Namespace du Service.
  - `labels:` : Étiquettes du Service.

- `spec:` : Spécification du Service.
  - `type: ClusterIP` : **Type de service - accès interne uniquement**.
    
    **Types de Service Kubernetes** :
    - `ClusterIP` : IP interne, accessible uniquement depuis le cluster (défaut).
    - `NodePort` : Expose le service sur un port de chaque nœud (30000-32767).
    - `LoadBalancer` : Crée un load balancer externe (nécessite un cloud provider).

    **Pourquoi ClusterIP pour MongoDB ?** Pour des raisons de sécurité. MongoDB ne doit être accessible que depuis les autres services internes, jamais depuis Internet.

  - `selector:` : **Sélectionne les Pods** que ce Service cible.
    - `app: fleetman-mongodb` : Le Service redirige le trafic vers tous les Pods ayant ce label.

  - `ports:` : Ports exposés par le Service.
    - `- port: 27017` : **Port du Service** - les autres Pods se connectent sur ce port.
    - `targetPort: 27017` : **Port du conteneur** - le Service redirige vers ce port du Pod.
    - `protocol: TCP` : Protocole utilisé.
    - `name: mongodb` : Nom du port.

**Analogie** : Le Service est comme un standard téléphonique. Peu importe quel employé (Pod) répond, vous composez toujours le même numéro (fleetman-mongodb:27017). Le standard redirige vers l'employé disponible.

---

### 3.4 queue-deployment.yaml - File de Messages ActiveMQ

Ce fichier contient le Deployment et le Service pour ActiveMQ, la file de messages.

#### 3.4.1 Deployment ActiveMQ Queue

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleetman-queue
  namespace: default
  labels:
    app: fleetman-queue
    component: messaging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fleetman-queue
  template:
    metadata:
      labels:
        app: fleetman-queue
        component: messaging
    spec:
      containers:
      - name: queue
        image: supinfo4kube/queue:1.0.1
        ports:
        - containerPort: 8161
          name: admin-ui
          protocol: TCP
        - containerPort: 61616
          name: message-port
          protocol: TCP
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "250m"
```

**Explication détaillée** :

- `replicas: 1` : **Une seule instance**. 
  
  **Pourquoi ?** ActiveMQ est une file de messages stateful. Avoir plusieurs instances nécessiterait une configuration de cluster complexe (master/slave). Pour cette application, une instance suffit.

- `image: supinfo4kube/queue:1.0.1` : Image Docker personnalisée contenant ActiveMQ préconfigurée.
  - `supinfo4kube` : Organisation/utilisateur Docker Hub.
  - `queue` : Nom de l'image.
  - `1.0.1` : Version spécifique.

- `ports:` : ActiveMQ expose **deux ports** :
  - `containerPort: 8161` : **Port de l'interface d'administration web**.
    - `name: admin-ui` : Nom descriptif (Interface graphique pour monitorer la queue).
    - Accès : `http://fleetman-queue:8161/admin`
    - Identifiants par défaut : admin/admin
  
  - `containerPort: 61616` : **Port de messagerie**.
    - `name: message-port` : Port utilisé par les applications (simulateur, tracker) pour envoyer/recevoir des messages.
    - Protocole : OpenWire (protocole de ActiveMQ).

**Pourquoi deux ports ?**
- **8161** : Pour les humains (monitoring, debug)
- **61616** : Pour les machines (applications qui communiquent)

- `resources:` : Ressources allouées.
  - `requests: memory: 256Mi, cpu: 100m` : Minimum garanti.
  - `limits: memory: 512Mi, cpu: 250m` : Maximum autorisé.

**Note** : Ces valeurs ont été ajustées après des problèmes de mémoire (OOMKilled). Initialement, nous avions des limites plus basses qui causaient des crashs.

#### 3.4.2 Service ActiveMQ Queue

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue
  namespace: default
  labels:
    app: fleetman-queue
    component: messaging
spec:
  type: ClusterIP
  selector:
    app: fleetman-queue
  ports:
  - port: 8161
    targetPort: 8161
    protocol: TCP
    name: admin-ui
  - port: 61616
    targetPort: 61616
    protocol: TCP
    name: message-port
```

**Explication** :

- `type: ClusterIP` : Accès interne uniquement. ActiveMQ ne doit pas être accessible depuis l'extérieur pour des raisons de sécurité.

- `ports:` : **Deux ports exposés** par le Service.
  - **Port 8161** : Interface d'administration.
    - Les développeurs peuvent y accéder via port-forward pour debugging.
  - **Port 61616** : Port de messagerie.
    - Utilisé par position-simulator (publier) et position-tracker (consommer).

**Comment les applications trouvent la queue ?**
Les applications Spring Boot utilisent la configuration :
```
spring.activemq.broker-url=tcp://fleetman-queue:61616
```

Kubernetes résout `fleetman-queue` en l'adresse IP du Service, qui redirige vers le Pod ActiveMQ.

---

### 3.5 position-simulator-deployment.yaml - Générateur de Positions

#### 3.5.1 Deployment Position Simulator

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleetman-position-simulator
  namespace: default
  labels:
    app: fleetman-position-simulator
    component: simulator
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fleetman-position-simulator
  template:
    metadata:
      labels:
        app: fleetman-position-simulator
        component: simulator
    spec:
      containers:
      - name: position-simulator
        image: supinfo4kube/position-simulator:1.0.1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production-microservice"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "250m"
```

**Explication détaillée** :

- `replicas: 2` : **Deux instances** du simulateur.
  
  **Pourquoi 2 ?** 
  - **Haute disponibilité** : Si une instance crash, l'autre continue.
  - **Performance** : Deux instances génèrent plus de données de test.
  - C'est une application **stateless** (sans état), donc on peut facilement en avoir plusieurs.

**Analogie** : C'est comme avoir 2 employés qui génèrent des rapports. Si l'un est malade, l'autre continue. Ils travaillent indépendamment.

- `image: supinfo4kube/position-simulator:1.0.1` : Image Docker du simulateur.

- `env:` : **Variables d'environnement** passées au conteneur.
  - `- name: SPRING_PROFILES_ACTIVE` : Nom de la variable.
  - `value: "production-microservice"` : Valeur de la variable.

**Qu'est-ce qu'un Spring Profile ?**
Spring Boot peut avoir différentes configurations selon l'environnement :
- `development` : Configuration pour développement local
- `production-microservice` : Configuration pour production en mode microservices

Cette configuration indique au simulateur :
- De se connecter à `fleetman-queue` (pas à `localhost`)
- D'utiliser les paramètres optimisés pour la production

**Comment Spring Boot lit cette variable ?**
```java
// Dans le code Spring Boot
@Profile("production-microservice")
public class ProductionConfig {
    // Configuration spécifique à la production
}
```

- `resources:` : Allocation de ressources.
  - `requests: 256Mi / 100m` : Garanti.
  - `limits: 512Mi / 250m` : Maximum.

**Pourquoi ces valeurs ?** Spring Boot (framework Java) consomme beaucoup de mémoire au démarrage (JVM). 256 Mi est le minimum raisonnable. Sans ces valeurs, nous avions des erreurs OOMKilled (Out Of Memory).

#### 3.5.2 Service Position Simulator

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-position-simulator
  namespace: default
  labels:
    app: fleetman-position-simulator
    component: simulator
spec:
  type: ClusterIP
  selector:
    app: fleetman-position-simulator
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
    name: http
```

**Explication** :

- `type: ClusterIP` : Accès interne uniquement.

**Pourquoi un Service pour le simulateur ?**
Même si personne n'appelle directement le simulateur, le Service est utile pour :
- **Monitoring** : Accéder aux endpoints de health check.
- **Logs** : Via des outils de monitoring (Prometheus, Grafana).
- **Debug** : Se connecter pour inspecter l'état.

- `port: 8080` : Spring Boot expose par défaut un serveur web sur le port 8080 avec des endpoints comme :
  - `/actuator/health` : État de santé de l'application.
  - `/actuator/metrics` : Métriques de performance.

---

### 3.6 position-tracker-deployment.yaml - Traitement et Stockage des Positions

#### 3.6.1 Deployment Position Tracker

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleetman-position-tracker
  namespace: default
  labels:
    app: fleetman-position-tracker
    component: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fleetman-position-tracker
  template:
    metadata:
      labels:
        app: fleetman-position-tracker
        component: backend
    spec:
      containers:
      - name: position-tracker
        image: supinfo4kube/position-tracker:1.0.1
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production-microservice"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "250m"
```

**Explication** :

- `replicas: 2` : Deux instances pour la haute disponibilité.

**Rôle du Position Tracker** :
1. **Consommer** les messages de la queue ActiveMQ (positions GPS).
2. **Traiter** ces positions (validation, transformation).
3. **Stocker** dans MongoDB.
4. **Exposer** une API REST pour lire les positions.

**Pourquoi 2 réplicas ?**
- **Haute disponibilité** : Si une instance crash, l'autre continue à traiter les messages.
- **Performance** : Deux instances traitent plus de messages par seconde.

**Note importante** : ActiveMQ garantit qu'un message n'est consommé que par un seul consumer. Donc pas de doublon même avec 2 instances.

- `env: SPRING_PROFILES_ACTIVE: production-microservice` : Configure le tracker pour :
  - Se connecter à `fleetman-queue:61616` (queue)
  - Se connecter à `fleetman-mongodb:27017` (database)

**Comment ça marche en interne ?**
```
Position Simulator → Publie message → ActiveMQ Queue
                                           ↓
                                  Position Tracker consume
                                           ↓
                                    Stocke dans MongoDB
```

#### 3.6.2 Service Position Tracker (avec NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-position-tracker
  namespace: default
  labels:
    app: fleetman-position-tracker
    component: backend
spec:
  type: NodePort
  selector:
    app: fleetman-position-tracker
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30010
    protocol: TCP
    name: http
```

**Explication** :

- `type: NodePort` : **Expose le service à l'extérieur du cluster**.

**Différence avec ClusterIP** :
- **ClusterIP** : IP interne, accessible uniquement depuis le cluster.
- **NodePort** : Accessible depuis l'extérieur via `<IP_du_noeud>:<NodePort>`.

- `port: 8080` : Port du Service (interne).
- `targetPort: 8080` : Port du conteneur.
- `nodePort: 30010` : **Port externe** sur le nœud Kubernetes.

**Plage de NodePort** : 30000-32767 (réservée par Kubernetes).

**Comment y accéder ?**
- En théorie : `http://localhost:30010`
- En pratique avec Docker Desktop : Nécessite `kubectl port-forward` (limitation Windows).

**Pourquoi exposer le Position Tracker ?**
- **Debugging** : Accéder directement à l'API pour tester.
- **Monitoring** : Vérifier l'état, les métriques.
- **Alternative** : L'API Gateway est le point d'entrée principal, mais avoir un accès direct est utile pour le développement.

---

### 3.7 api-gateway-deployment.yaml - Passerelle API REST

#### 3.7.1 Deployment API Gateway

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleetman-api-gateway
  namespace: default
  labels:
    app: fleetman-api-gateway
    component: gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fleetman-api-gateway
  template:
    metadata:
      labels:
        app: fleetman-api-gateway
        component: gateway
    spec:
      containers:
      - name: api-gateway
        image: supinfo4kube/api-gateway:1.0.1
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production-microservice"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "250m"
```

**Explication détaillée** :

- `replicas: 2` : Deux instances pour haute disponibilité.

**Rôle de l'API Gateway** :
L'API Gateway est le **point d'entrée unique** pour toutes les requêtes vers les services backend. C'est un pattern architectural appelé **API Gateway Pattern**.

**Analogie** : Imaginez un hôtel. Au lieu que les clients appellent directement la cuisine, la blanchisserie, le ménage, ils appellent la **réception** (API Gateway). La réception redirige vers le bon service.

**Avantages** :
1. **Abstraction** : Le client ne connaît qu'une seule URL.
2. **Sécurité** : L'API Gateway peut gérer l'authentification.
3. **Routage** : Redirige vers les bons microservices.
4. **Agrégation** : Peut combiner plusieurs appels en un seul.

**Qu'est-ce que l'API Gateway fait dans notre application ?**
- **Reçoit** les requêtes HTTP du frontend (webapp).
- **Appelle** le Position Tracker pour récupérer les données.
- **Retourne** les résultats au format JSON.

Exemple de flux :
```
Navigateur → API Gateway (/api/vehicles) → Position Tracker → MongoDB
                                                  ↓
Navigateur ← API Gateway ← Position Tracker ← Données
```

- `image: supinfo4kube/api-gateway:1.0.1` : Image Docker contenant le code Spring Boot de l'API Gateway.

- `env: SPRING_PROFILES_ACTIVE: production-microservice` : Configure l'API Gateway pour :
  - Se connecter à `fleetman-position-tracker:8080` (pas à localhost).
  - Utiliser les timeouts et retry policy appropriés pour la production.

**Configuration Spring Boot en interne** :
```yaml
# application-production-microservice.yml
position-tracker:
  url: http://fleetman-position-tracker:8080
```

Spring Boot remplace automatiquement le hostname par l'adresse du Service Kubernetes.

- `resources: requests: 256Mi/100m, limits: 512Mi/250m` : Même allocation que les autres services Spring Boot.

#### 3.7.2 Service API Gateway (avec NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-api-gateway
  namespace: default
  labels:
    app: fleetman-api-gateway
    component: gateway
spec:
  type: NodePort
  selector:
    app: fleetman-api-gateway
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30020
    protocol: TCP
    name: http
```

**Explication** :

- `type: NodePort` : Expose le service à l'extérieur.

**Pourquoi NodePort pour l'API Gateway ?**
- Le frontend (webapp) doit pouvoir appeler l'API Gateway.
- En production, on utiliserait un **Ingress** ou **LoadBalancer**, mais NodePort est plus simple pour le développement.

- `nodePort: 30020` : Port externe sur le nœud.

**Accès** :
- En théorie : `http://localhost:30020/api/vehicles`
- En pratique (Docker Desktop Windows) : `kubectl port-forward service/fleetman-api-gateway 8080:8080`

**Endpoints disponibles** :
- `/api/vehicles` : Liste tous les véhicules.
- `/api/vehicles/{vehicleId}` : Détails d'un véhicule spécifique.
- `/api/vehicles/{vehicleId}/history` : Historique des positions.

**Test manuel** :
```powershell
kubectl port-forward service/fleetman-api-gateway 8080:8080
curl http://localhost:8080/api/vehicles
```

Résultat attendu : JSON avec la liste des véhicules et leurs positions.

---

### 3.8 webapp-deployment.yaml - Interface Utilisateur Web

#### 3.8.1 Deployment WebApp

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleetman-webapp
  namespace: default
  labels:
    app: fleetman-webapp
    component: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fleetman-webapp
  template:
    metadata:
      labels:
        app: fleetman-webapp
        component: frontend
    spec:
      containers:
      - name: webapp
        image: supinfo4kube/web-app:1.0.0
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

**Explication détaillée** :

- `replicas: 2` : Deux instances pour haute disponibilité.

**Rôle de la WebApp** :
L'application web est l'**interface utilisateur** visible par les utilisateurs finaux. C'est une Single Page Application (SPA) qui affiche une carte avec les véhicules en temps réel.

**Technologies** :
- **Nginx** : Serveur web léger qui sert les fichiers statiques (HTML, CSS, JavaScript).
- **Angular/React/Vue** (selon l'implémentation) : Framework JavaScript pour l'interface interactive.

**Analogie** : La webapp est comme la **vitrine d'un magasin**. Les clients voient les produits (véhicules sur la carte), mais tout le travail (traitement, stockage) se fait en arrière-boutique (backend).

- `image: supinfo4kube/web-app:1.0.0` : Image Docker contenant :
  - Nginx configuré.
  - Fichiers HTML/CSS/JavaScript de l'application.
  - Configuration de proxy vers l'API Gateway.

- `containerPort: 80` : Nginx écoute sur le port 80 (port HTTP standard).

**Différence avec les autres services** :
- Les autres services utilisent le port 8080 (Spring Boot).
- La webapp utilise le port 80 (Nginx).

- `resources: requests: 64Mi/50m, limits: 128Mi/100m` : **Beaucoup moins de ressources** que les services Spring Boot.

**Pourquoi ?**
- Nginx est ultra-léger (écrit en C, pas en Java).
- Il ne fait que servir des fichiers statiques (pas de traitement lourd).
- Spring Boot (JVM) nécessite 512Mi, Nginx fonctionne avec 64Mi.

**Analogie** : C'est comme comparer un camion (Spring Boot/Java) et un vélo (Nginx/C). Le vélo consomme beaucoup moins de carburant.

#### 3.8.2 Service WebApp (avec NodePort)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp
  namespace: default
  labels:
    app: fleetman-webapp
    component: frontend
spec:
  type: NodePort
  selector:
    app: fleetman-webapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
    name: http
```

**Explication** :

- `type: NodePort` : Expose l'application web à l'extérieur.

**Pourquoi NodePort ?**
- Les utilisateurs doivent accéder à l'application via leur navigateur.
- C'est le **point d'entrée principal** de l'application.

- `port: 80` : Port du Service (interne).
- `targetPort: 80` : Port du conteneur Nginx.
- `nodePort: 30080` : Port externe.

**Accès** :
- En théorie : `http://localhost:30080`
- En pratique (Docker Desktop) : `kubectl port-forward service/fleetman-webapp 3000:80`
- Ouvrir : `http://localhost:3000` dans le navigateur.

#### 3.8.3 Le Problème DNS Hardcodé (Cause du Bug Namespace)

**⚠️ Point Critique** : Cette webapp a causé notre problème majeur de déploiement !

**Configuration Nginx interne** :
Dans l'image Docker `supinfo4kube/web-app:1.0.0`, le fichier de configuration Nginx contient :

```nginx
upstream api {
    server fleetman-api-gateway.default.svc.cluster.local:8080;
}

location /api/ {
    proxy_pass http://api/;
}
```

**Explication** :
- `upstream api` : Définit un backend nommé "api".
- `server fleetman-api-gateway.default.svc.cluster.local:8080` : **Adresse DNS hardcodée**.

**Le problème** :
- `fleetman-api-gateway` : Nom du Service (✅ OK).
- `.default` : **Namespace hardcodé en dur** (❌ PROBLÈME).
- `.svc.cluster.local` : Suffixe DNS Kubernetes standard (✅ OK).
- `:8080` : Port (✅ OK).

**Qu'est-ce que ça signifie ?**
La webapp cherche l'API Gateway **uniquement dans le namespace "default"**.

**Notre erreur initiale** :
Nous avions créé un namespace "fleetman" pour tous les services. Résultat :
- L'API Gateway était dans `fleetman-api-gateway.fleetman.svc.cluster.local`.
- Mais la webapp cherchait `fleetman-api-gateway.default.svc.cluster.local`.
- **DNS ne trouvait rien** → Webapp crashait avec "host not found".

**Les logs de la webapp** :
```
2025/01/30 10:15:23 [emerg] 1#1: host not found in upstream "fleetman-api-gateway.default.svc.cluster.local" in /etc/nginx/nginx.conf:12
nginx: [emerg] host not found in upstream "fleetman-api-gateway.default.svc.cluster.local" in /etc/nginx/nginx.conf:12
```

**Solution** :
Nous avons **changé tous les manifests** pour utiliser le namespace "default" au lieu de "fleetman".

**Leçon apprise** :
- **Toujours vérifier les configurations hardcodées** dans les images Docker.
- En production, on utiliserait des **variables d'environnement** pour le namespace :
  ```nginx
  server ${API_GATEWAY_HOST}:8080;
  ```

**Format DNS Kubernetes complet** :
```
<service-name>.<namespace>.svc.cluster.local:<port>
```

Exemples :
- `fleetman-mongodb.default.svc.cluster.local:27017`
- `fleetman-queue.default.svc.cluster.local:61616`
- `fleetman-api-gateway.default.svc.cluster.local:8080`

**Raccourcis DNS** :
- Depuis le même namespace : `fleetman-mongodb` (suffixe automatique).
- Depuis un autre namespace : `fleetman-mongodb.other-namespace` (nécessite le namespace).

---

## 4. Problèmes Rencontrés et Solutions Appliquées

Durant le déploiement, nous avons rencontré plusieurs problèmes techniques. Voici chaque problème, ses symptômes, sa cause profonde, et la solution appliquée.

### 4.1 Problème n°1 : OOMKilled - Manque de Mémoire

#### 4.1.1 Symptômes

Les Pods redémarraient en boucle. En vérifiant l'état :

```powershell
kubectl get pods
```

Résultat :
```
NAME                                      READY   STATUS      RESTARTS   AGE
fleetman-api-gateway-xxxxx                0/1     OOMKilled   5          2m
fleetman-position-tracker-xxxxx           0/1     OOMKilled   3          2m
```

En inspectant un Pod :
```powershell
kubectl describe pod fleetman-api-gateway-xxxxx
```

Dans la section **Events** :
```
Reason: OOMKilled
Message: Container was killed due to out-of-memory
```

#### 4.1.2 Cause Profonde

**OOMKilled** signifie **Out Of Memory Killed** (tué pour manque de mémoire).

**Qu'est-ce qui s'est passé ?**
1. Nous avions initialement configuré `limits.memory: 256Mi` pour les services Spring Boot.
2. Spring Boot (basé sur Java/JVM) consomme beaucoup de mémoire au démarrage :
   - JVM (Java Virtual Machine) : ~150 Mi
   - Spring Framework : ~50 Mi
   - Application logic : ~50 Mi
   - **Total** : ~250 Mi minimum
3. Avec une limite de 256 Mi, le conteneur dépassait la limite → Linux OOM Killer (mécanisme du kernel) tuait le processus.

**Analogie** : Imaginez un employé avec un bureau de 2m² (256Mi). Si son travail nécessite 3m² de documents étalés (Spring Boot), il ne peut pas travailler. Le manager (Linux) le renvoie chez lui (kill le processus).

#### 4.1.3 Solution Appliquée

**Augmentation des limites de mémoire** dans tous les fichiers de déploiement Spring Boot :

**Avant** :
```yaml
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "256Mi"
```

**Après** :
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

**Explication** :
- `requests: 256Mi` : Kubernetes **garantit** 256 Mi disponibles.
- `limits: 512Mi` : Le Pod peut utiliser **jusqu'à** 512 Mi si disponible.

**Résultat** :
- Tous les Pods Spring Boot démarrent correctement.
- Pas de restart depuis l'ajustement.
- Utilisation réelle : ~300-350 Mi par Pod.

**Commande de vérification** :
```powershell
kubectl top pod
```

Résultat :
```
NAME                                      CPU(cores)   MEMORY(bytes)
fleetman-api-gateway-xxx                  50m          320Mi
fleetman-position-tracker-xxx             45m          350Mi
```

---

### 4.2 Problème n°2 : Health Probes Échecs Répétés

#### 4.2.1 Symptômes

Les Pods affichaient `0/1 Ready` avec des restarts constants :

```powershell
kubectl get pods
```

Résultat :
```
NAME                                      READY   STATUS    RESTARTS   AGE
fleetman-position-simulator-xxxxx         0/1     Running   8          5m
```

En vérifiant les événements :
```powershell
kubectl describe pod fleetman-position-simulator-xxxxx
```

Dans **Events** :
```
Readiness probe failed: HTTP probe failed with statuscode: 404
Liveness probe failed: Get "http://10.1.0.5:8080/actuator/health": dial tcp 10.1.0.5:8080: connect: connection refused
```

#### 4.2.2 Cause Profonde

Nous avions configuré des **health probes** (sondes de santé) dans les Deployments :

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

**Qu'est-ce qu'une health probe ?**
- **Liveness Probe** : Kubernetes vérifie si le Pod est en vie. Si échec → redémarre le Pod.
- **Readiness Probe** : Kubernetes vérifie si le Pod est prêt à recevoir du trafic. Si échec → retire le Pod du Service.

**Le problème** :
Les images Docker `supinfo4kube/*` **n'exposent pas** l'endpoint `/actuator/health`.

**Pourquoi ?**
`/actuator/health` est un endpoint Spring Boot Actuator. Il faut :
1. Ajouter la dépendance Spring Boot Actuator dans le code.
2. Configurer l'exposition dans `application.yml`.

Ces images Docker n'ont pas cette configuration → endpoint inexistant → probe échoue → restart en boucle.

**Analogie** : C'est comme appeler un numéro de téléphone (endpoint) qui n'existe pas. Le système pense que le service est mort, alors qu'il fonctionne mais n'a simplement pas ce numéro.

#### 4.2.3 Solution Appliquée

**Suppression complète des health probes** de tous les Deployments.

**Modification** : Retrait des sections `livenessProbe` et `readinessProbe`.

**Pourquoi c'est acceptable ?**
- En **développement/test** : Pas critique. Docker Desktop redémarre les Pods en cas de crash réel.
- En **production** : Il faudrait soit :
  - Modifier les images Docker pour exposer `/actuator/health`.
  - Ou créer un endpoint custom (`/health`) dans l'application.

**Résultat** :
- Tous les Pods démarrent et restent dans l'état `Running`.
- `READY` passe à `1/1`.
- Pas de restarts intempestifs.

**Vérification** :
```powershell
kubectl get pods
```

Résultat :
```
NAME                                      READY   STATUS    RESTARTS   AGE
fleetman-position-simulator-xxx           1/1     Running   0          10m
```

---

### 4.3 Problème n°3 : WebApp CrashLoopBackOff - Erreur DNS Namespace

#### 4.3.1 Symptômes

Le Pod webapp crashait immédiatement au démarrage :

```powershell
kubectl get pods
```

Résultat :
```
NAME                                READY   STATUS             RESTARTS   AGE
fleetman-webapp-xxxxx               0/1     CrashLoopBackOff   5          3m
```

**CrashLoopBackOff** signifie : Le Pod crash → Kubernetes attend avant de redémarrer → crash à nouveau → attend plus longtemps → etc.

En vérifiant les logs :
```powershell
kubectl logs fleetman-webapp-xxxxx
```

Résultat :
```
2025/01/30 10:15:23 [emerg] 1#1: host not found in upstream "fleetman-api-gateway.default.svc.cluster.local" in /etc/nginx/nginx.conf:12
nginx: [emerg] host not found in upstream "fleetman-api-gateway.default.svc.cluster.local" in /etc/nginx/nginx.conf:12
```

#### 4.3.2 Cause Profonde

**Le problème** : La webapp cherche `fleetman-api-gateway.default.svc.cluster.local`, mais nous avions déployé tous les services dans le namespace **"fleetman"**.

**Configuration Nginx dans l'image Docker** :
```nginx
upstream api {
    server fleetman-api-gateway.default.svc.cluster.local:8080;
}
```

Le mot-clé `.default` est **hardcodé** (écrit en dur) dans la configuration Nginx.

**Pourquoi ça pose problème ?**
- L'API Gateway était déployé dans : `fleetman-api-gateway.fleetman.svc.cluster.local`
- La webapp cherchait dans : `fleetman-api-gateway.default.svc.cluster.local`
- DNS Kubernetes ne trouvait pas → erreur "host not found" → Nginx refusait de démarrer.

**Format DNS Kubernetes** :
```
<service-name>.<namespace>.svc.cluster.local
```

**Notre situation initiale** :
- Tous les manifests utilisaient `namespace: fleetman`.
- DNS résolvait vers `*.fleetman.svc.cluster.local`.
- Mais webapp cherchait `*.default.svc.cluster.local`.

**Analogie** : C'est comme chercher un appartement au 5ème étage (namespace default) alors qu'il est au 3ème étage (namespace fleetman). L'adresse est incorrecte, donc impossible de le trouver.

#### 4.3.3 Solution Appliquée

**Changement de tous les manifests vers le namespace "default".**

**Modification dans TOUS les fichiers YAML** :

**Avant** :
```yaml
metadata:
  namespace: fleetman
```

**Après** :
```yaml
metadata:
  namespace: default
```

**Fichiers modifiés** :
- namespace.yaml
- mongodb-deployment.yaml, mongodb-pvc.yaml
- queue-deployment.yaml
- position-simulator-deployment.yaml
- position-tracker-deployment.yaml
- api-gateway-deployment.yaml
- webapp-deployment.yaml
- Tous les fichiers Service correspondants

**Commandes appliquées** :
```powershell
# Supprimer les anciens déploiements
kubectl delete namespace fleetman

# Redéployer dans default
kubectl apply -f namespace.yaml
kubectl apply -f mongodb-pvc.yaml
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f queue-deployment.yaml
kubectl apply -f position-simulator-deployment.yaml
kubectl apply -f position-tracker-deployment.yaml
kubectl apply -f api-gateway-deployment.yaml
kubectl apply -f webapp-deployment.yaml
```

**Résultat** :
- DNS résout correctement : `fleetman-api-gateway.default.svc.cluster.local` → IP du Service.
- Webapp démarre sans erreur.
- `kubectl get pods` : Tous les Pods `Running 1/1`.

**Vérification finale** :
```powershell
kubectl logs fleetman-webapp-xxxxx
```

Résultat (logs Nginx sains) :
```
2025/01/30 11:20:15 [notice] 1#1: start worker process 30
```

**Leçon apprise** :
- **Toujours vérifier les configurations hardcodées** dans les images Docker tierces.
- En production, utiliser des **variables d'environnement** pour les noms de services et namespaces.
- **Tester le DNS** : `kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup fleetman-api-gateway.default.svc.cluster.local`

---

### 4.4 Problème n°4 : NodePort Non Accessible sur Docker Desktop Windows

#### 4.4.1 Symptômes

Les Services de type `NodePort` étaient créés avec succès :

```powershell
kubectl get svc
```

Résultat :
```
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
fleetman-webapp             NodePort    10.96.150.23    <none>        80:30080/TCP     5m
fleetman-api-gateway        NodePort    10.96.140.50    <none>        8080:30020/TCP   5m
```

Mais impossible d'y accéder via le navigateur :

- `http://localhost:30080` → "Empty reply from server"
- `http://127.0.0.1:30080` → "Connection refused"

Test avec curl :
```powershell
curl http://localhost:30080
```

Résultat :
```
curl: (52) Empty reply from server
```

#### 4.4.2 Cause Profonde

**Limitation de Docker Desktop sur Windows** :

Docker Desktop for Windows utilise une machine virtuelle (WSL2 ou Hyper-V) pour exécuter Kubernetes. Le réseau entre Windows et cette VM a des limitations :

1. **NodePort sur localhost ne fonctionne pas directement** contrairement à Linux.
2. Les ports NodePort (30000-32767) ne sont pas automatiquement exposés sur l'hôte Windows.

**Pourquoi ?**
- Sur **Linux natif** : Kubernetes tourne directement → `localhost:30080` fonctionne.
- Sur **Docker Desktop Windows** : Kubernetes tourne dans une VM → `localhost:30080` pointe vers Windows, pas vers la VM.

**Analogie** : C'est comme appeler un numéro de téléphone interne d'une entreprise (NodePort dans la VM) depuis l'extérieur (Windows). Le numéro n'est pas accessible directement, il faut passer par le standard (port-forward).

#### 4.4.3 Solution Appliquée

**Utilisation de `kubectl port-forward`** au lieu de NodePort direct.

**Commande** :
```powershell
kubectl port-forward service/fleetman-webapp 3000:80
```

**Explication** :
- `kubectl port-forward` : Commande qui crée un tunnel entre votre machine et le cluster Kubernetes.
- `service/fleetman-webapp` : Cible le Service nommé "fleetman-webapp".
- `3000:80` : 
  - `3000` : Port sur votre machine Windows (localhost).
  - `80` : Port du Service dans Kubernetes.

**Flux** :
```
Navigateur (localhost:3000) → kubectl port-forward → Service (port 80) → Pod webapp
```

**Avantage** :
- Fonctionne sur Docker Desktop Windows/Mac.
- Ne nécessite pas de configuration réseau complexe.
- Idéal pour le développement.

**Inconvénient** :
- Il faut garder la fenêtre PowerShell ouverte (Ctrl+C arrête le tunnel).
- En production, on utiliserait un **Ingress Controller** ou **LoadBalancer**.

**Configuration finale pour l'accès complet** :

**Fenêtre PowerShell 1** : WebApp
```powershell
kubectl port-forward service/fleetman-webapp 3000:80
```
Accès : `http://localhost:3000`

**Fenêtre PowerShell 2** : API Gateway
```powershell
kubectl port-forward service/fleetman-api-gateway 8080:8080
```
Accès : `http://localhost:8080/api/vehicles`

**Fenêtre PowerShell 3** : Position Tracker (optionnel pour debug)
```powershell
kubectl port-forward service/fleetman-position-tracker 8081:8080
```
Accès : `http://localhost:8081`

**Résultat** :
- La webapp s'affiche correctement dans le navigateur.
- La carte avec les véhicules apparaît.
- Les positions se mettent à jour en temps réel.

**Test de validation** :
```powershell
curl http://localhost:3000
```

Résultat (extrait HTML) :
```html
<!DOCTYPE html>
<html>
<head>
    <title>Fleet Management</title>
...
</head>
```

**Statut HTTP 200** → Success !

---

### 4.5 Problème n°5 : Ressources Insuffisantes Docker Desktop

#### 4.5.1 Symptômes

Certains Pods restaient bloqués en état `Pending` :

```powershell
kubectl get pods
```

Résultat :
```
NAME                                      READY   STATUS    RESTARTS   AGE
fleetman-position-tracker-xxxxx           0/1     Pending   0          2m
```

En inspectant :
```powershell
kubectl describe pod fleetman-position-tracker-xxxxx
```

Dans **Events** :
```
Warning  FailedScheduling  2m  default-scheduler  0/1 nodes are available: 1 Insufficient memory.
```

#### 4.5.2 Cause Profonde

**Docker Desktop alloue par défaut 2 Go de RAM** à la VM Kubernetes.

**Calcul de nos besoins** :
- MongoDB : 512 Mi
- Queue : 512 Mi
- Position Simulator (x2) : 2 × 512 Mi = 1024 Mi
- Position Tracker (x2) : 2 × 512 Mi = 1024 Mi
- API Gateway (x2) : 2 × 512 Mi = 1024 Mi
- WebApp (x2) : 2 × 128 Mi = 256 Mi

**Total** : ~4.3 Go minimum.

Avec seulement 2 Go alloués → **Insuffisant** → Scheduler Kubernetes ne peut pas placer tous les Pods.

**Analogie** : C'est comme essayer de loger 10 personnes dans un appartement de 2 chambres. Impossible, il faut un plus grand appartement.

#### 4.5.3 Solution Appliquée

**Augmentation de la mémoire Docker Desktop** :

**Étapes** :
1. Ouvrir **Docker Desktop**.
2. Cliquer sur l'icône **Settings** (engrenage).
3. Aller dans **Resources → Advanced**.
4. Augmenter **Memory** de **2 Go** à **6 Go** (ou plus).
5. Cliquer sur **Apply & Restart**.

**Docker Desktop redémarre** avec les nouvelles ressources.

**Vérification** :
```powershell
kubectl get nodes
kubectl describe node docker-desktop
```

Dans la section **Allocatable** :
```
Allocatable:
  cpu:                8
  memory:             6Gi
```

**Résultat** :
- Tous les Pods sont schedulés avec succès.
- `kubectl get pods` : Tous `Running 1/1`.

**Recommandation** :
- **Développement** : 4-6 Go minimum.
- **Production** : Cluster multi-nœuds avec des ressources dédiées.

---

### 4.6 Erreurs Additionnelles et Leçons Apprises

Au cours du déploiement, nous avons également rencontré d'autres erreurs courantes. Voici un récapitulatif complet.

#### 4.6.1 Erreur : "No resources found in default namespace"

**Symptôme** :
```powershell
kubectl get pods
# No resources found in default namespace.
```

**Cause** :
Les ressources ont été déployées dans un namespace spécifique (ex: "fleetman"), mais la commande `kubectl get pods` cherche dans le namespace "default" par défaut.

**Solution** :
```powershell
# Option 1 : Spécifier le namespace
kubectl get pods -n fleetman

# Option 2 : Changer le namespace par défaut
kubectl config set-context --current --namespace=fleetman

# Option 3 : Voir tous les namespaces
kubectl get pods --all-namespaces
```

#### 4.6.2 Erreur : "ImagePullBackOff"

**Symptôme** :
```powershell
kubectl get pods
# NAME                                READY   STATUS             RESTARTS   AGE
# fleetman-webapp-xxx                 0/1     ImagePullBackOff   0          2m
```

**Cause** :
Kubernetes ne peut pas télécharger l'image Docker. Raisons possibles :
- Image inexistante ou mal orthographiée.
- Registre Docker privé nécessitant une authentification.
- Problèmes de connexion réseau.

**Diagnostic** :
```powershell
kubectl describe pod fleetman-webapp-xxx
# Events:
# Failed to pull image "supinfo4kube/web-app:1.0.0": rpc error: code = Unknown desc = Error response from daemon: pull access denied
```

**Solutions** :
- Vérifier le nom de l'image : `image: supinfo4kube/web-app:1.0.0`
- Pour un registre privé, créer un Secret :
  ```powershell
  kubectl create secret docker-registry regcred \
    --docker-server=<registry> \
    --docker-username=<username> \
    --docker-password=<password> \
    --docker-email=<email>
  ```
- Ajouter dans le Deployment :
  ```yaml
  spec:
    imagePullSecrets:
    - name: regcred
  ```

#### 4.6.3 Erreur : "Endpoints Not Ready"

**Symptôme** :
```powershell
kubectl get endpoints
# NAME                        ENDPOINTS         AGE
# fleetman-api-gateway        <none>            5m
```

**Cause** :
Aucun Pod n'est prêt à recevoir du trafic. Le Service ne peut pas router les requêtes.

**Diagnostic** :
```powershell
# Vérifier l'état des Pods
kubectl get pods -l app=fleetman-api-gateway

# Vérifier les labels
kubectl describe svc fleetman-api-gateway
# Selector: app=fleetman-api-gateway

# Vérifier que les Pods ont bien ce label
kubectl get pods --show-labels
```

**Causes fréquentes** :
- Les Pods sont en état `CrashLoopBackOff` ou `Pending`.
- Les **labels** du Service ne correspondent pas aux labels des Pods.
- Les **health probes** échouent (Pods `0/1 Ready`).

**Solution** :
- Corriger les labels dans le Deployment pour qu'ils correspondent au Service.
- Résoudre les problèmes de démarrage des Pods (logs, describe).

#### 4.6.4 Erreur : "Connection Refused" ou "Timeout"

**Symptôme** :
```powershell
curl http://localhost:30080
# curl: (7) Failed to connect to localhost port 30080: Connection refused
```

**Cause** :
- Sur Docker Desktop Windows/Mac : NodePort ne fonctionne pas directement.
- Le Service n'a pas d'endpoints (Pods non prêts).
- Le port est incorrect.

**Solution** :
```powershell
# Vérifier que les Pods sont Running
kubectl get pods

# Vérifier que le Service a des endpoints
kubectl get endpoints fleetman-webapp

# Sur Docker Desktop : Utiliser port-forward
kubectl port-forward service/fleetman-webapp 3000:80
curl http://localhost:3000
```

#### 4.6.5 Erreur : "PersistentVolumeClaim is Pending"

**Symptôme** :
```powershell
kubectl get pvc
# NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   AGE
# mongodb-pvc   Pending                                      5m
```

**Cause** :
Aucun PersistentVolume (PV) disponible ne correspond aux exigences du PVC (taille, access mode, storage class).

**Diagnostic** :
```powershell
kubectl describe pvc mongodb-pvc
# Events:
# Warning  ProvisioningFailed  2m  persistentvolume-controller  no persistent volumes available
```

**Solutions** :
- **Docker Desktop** : Utilise le provisioning automatique avec `storageClassName: hostpath`. Vérifier que Kubernetes est bien activé.
- **Cluster cloud** : Vérifier que le StorageClass existe :
  ```powershell
  kubectl get storageclass
  ```
- Créer manuellement un PersistentVolume (si pas de provisioning automatique) :
  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: mongodb-pv
  spec:
    capacity:
      storage: 5Gi
    accessModes:
      - ReadWriteOnce
    hostPath:
      path: /data/mongodb
  ```

#### 4.6.6 Erreur : "Forbidden" ou "Error from server"

**Symptôme** :
```powershell
kubectl apply -f deployment.yaml
# Error from server (Forbidden): error when creating "deployment.yaml": deployments.apps is forbidden: User "system:anonymous" cannot create resource "deployments"
```

**Cause** :
Problème d'authentification ou de permissions RBAC (Role-Based Access Control).

**Solutions** :
- Vérifier que kubectl est bien configuré :
  ```powershell
  kubectl config view
  kubectl config current-context
  ```
- Se reconnecter au cluster :
  ```powershell
  kubectl config use-context docker-desktop
  ```
- Sur Docker Desktop, redémarrer Kubernetes dans Settings.

#### 4.6.7 Erreur : "Context Deadline Exceeded"

**Symptôme** :
```powershell
kubectl get pods
# Error from server: context deadline exceeded
```

**Cause** :
- Kubernetes API server ne répond pas (cluster en panne ou lent).
- Docker Desktop Kubernetes est arrêté.
- Problèmes réseau.

**Solutions** :
- Vérifier que Docker Desktop est lancé.
- Redémarrer Kubernetes : Settings → Kubernetes → Restart.
- Vérifier l'état de Docker Desktop :
  ```powershell
  docker version
  kubectl cluster-info
  ```

#### 4.6.8 Erreur : "Error: UPGRADE FAILED: another operation is in progress"

**Symptôme** (si vous utilisez Helm) :
```powershell
helm install myapp ./chart
# Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```

**Cause** :
Une opération Helm précédente est bloquée.

**Solution** :
```powershell
# Lister les releases
helm list --all

# Supprimer la release bloquée
helm uninstall myapp

# Ou forcer le rollback
helm rollback myapp 0
```

#### 4.6.9 Erreur : "Pod has unbound immediate PersistentVolumeClaims"

**Symptôme** :
```powershell
kubectl describe pod mongodb-xxx
# Events:
# Warning  FailedScheduling  2m  default-scheduler  pod has unbound immediate PersistentVolumeClaims
```

**Cause** :
Le PVC référencé par le Pod n'est pas encore "Bound" (lié à un PV).

**Solution** :
- Vérifier l'état du PVC :
  ```powershell
  kubectl get pvc
  ```
- Attendre que le PVC soit provisionné (peut prendre quelques secondes).
- Si le PVC reste "Pending", voir la section 4.6.5 ci-dessus.

#### 4.6.10 Erreur : "Too many open files"

**Symptôme** (dans les logs des Pods) :
```
2025-01-30 12:00:00 ERROR Cannot open database: too many open files
```

**Cause** :
Le système a atteint la limite de fichiers ouverts (file descriptors).

**Solution (Docker Desktop)** :
- Augmenter les limites du système :
  - **Windows** : Redémarrer Docker Desktop.
  - **Linux** : Modifier `/etc/security/limits.conf` :
    ```
    * soft nofile 65536
    * hard nofile 65536
    ```
- Ajouter des limites dans le Deployment :
  ```yaml
  securityContext:
    sysctls:
    - name: fs.file-max
      value: "65536"
  ```

#### 4.6.11 Récapitulatif des Commandes de Debug

**Checklist de debugging Kubernetes** :

```powershell
# 1. Vérifier l'état général
kubectl get all
kubectl get pods --all-namespaces

# 2. Vérifier un Pod spécifique
kubectl describe pod <nom-pod>
kubectl logs <nom-pod>
kubectl logs <nom-pod> --previous  # Logs du conteneur précédent

# 3. Vérifier les Services et Endpoints
kubectl get svc
kubectl get endpoints

# 4. Vérifier les ressources de stockage
kubectl get pvc
kubectl get pv

# 5. Tester la connectivité réseau
kubectl run curl-test --image=curlimages/curl:latest --rm -it --restart=Never -- curl http://<service-name>:<port>

# 6. Vérifier les événements du cluster
kubectl get events --sort-by='.lastTimestamp'

# 7. Vérifier les ressources disponibles
kubectl top nodes
kubectl top pods

# 8. Accéder à un Pod pour debugging
kubectl exec -it <nom-pod> -- /bin/sh

# 9. Vérifier la configuration du contexte
kubectl config current-context
kubectl config view

# 10. Vérifier les labels et selectors
kubectl get pods --show-labels
kubectl describe svc <nom-service>
```

---

## 5. Guide de Déploiement : Commandes Détaillées

Cette section explique toutes les commandes Kubernetes utilisées pour déployer et gérer l'application.

### 5.1 kubectl apply - Déployer les Ressources

**Commande** :
```powershell
kubectl apply -f <fichier.yaml>
```

**Qu'est-ce que ça fait ?**
- `kubectl` : Outil en ligne de commande pour interagir avec Kubernetes.
- `apply` : Crée ou met à jour une ressource à partir d'un fichier YAML.
- `-f` : Flag signifiant "file" (fichier).
- `<fichier.yaml>` : Chemin vers le fichier de configuration.

**Différence entre `create` et `apply`** :
- `kubectl create` : Crée une ressource. Échoue si elle existe déjà.
- `kubectl apply` : Crée ou met à jour. **Idempotent** (peut être exécuté plusieurs fois sans erreur).

**Analogie** : `apply` est comme "enregistrer" dans Word. Si le document existe, il est mis à jour. Sinon, il est créé.

**Déploiement complet de notre application** :
```powershell
# 1. Créer le namespace (bien que "default" existe déjà)
kubectl apply -f namespace.yaml

# 2. Créer le PersistentVolumeClaim pour MongoDB
kubectl apply -f mongodb-pvc.yaml

# 3. Déployer MongoDB avec son Service
kubectl apply -f mongodb-deployment.yaml

# 4. Déployer la Queue ActiveMQ avec son Service
kubectl apply -f queue-deployment.yaml

# 5. Déployer le Position Simulator avec son Service
kubectl apply -f position-simulator-deployment.yaml

# 6. Déployer le Position Tracker avec son Service
kubectl apply -f position-tracker-deployment.yaml

# 7. Déployer l'API Gateway avec son Service
kubectl apply -f api-gateway-deployment.yaml

# 8. Déployer la WebApp avec son Service
kubectl apply -f webapp-deployment.yaml
```

**Ordre important ?**
En théorie, Kubernetes gère les dépendances. En pratique, il est préférable de :
1. Créer les ressources de stockage (PVC) d'abord.
2. Déployer les dépendances (MongoDB, Queue) avant les applications qui les utilisent.

**Commande pour déployer tout d'un coup** :
```powershell
kubectl apply -f .
```
Cette commande applique **tous** les fichiers YAML du répertoire courant.

**Vérification** :
```powershell
kubectl get all
```

Résultat attendu : Liste de tous les Pods, Services, Deployments créés.

---

### 5.2 kubectl get - Lister les Ressources

#### 5.2.1 kubectl get pods - Voir les Pods

**Commande** :
```powershell
kubectl get pods
```

**Résultat** :
```
NAME                                      READY   STATUS    RESTARTS   AGE
fleetman-api-gateway-6f8c9d7b5c-abc12     1/1     Running   0          10m
fleetman-api-gateway-6f8c9d7b5c-def34     1/1     Running   0          10m
fleetman-mongodb-5d7c8b9f4d-ghi56         1/1     Running   0          15m
fleetman-position-simulator-7d8e9c-jkl78  1/1     Running   0          12m
fleetman-position-simulator-7d8e9c-mno90  1/1     Running   0          12m
fleetman-position-tracker-8e9f0a-pqr12    1/1     Running   0          11m
fleetman-position-tracker-8e9f0a-stu34    1/1     Running   0          11m
fleetman-queue-9f0g1b-vwx56               1/1     Running   0          14m
fleetman-webapp-0g1h2c-yza78              1/1     Running   0          9m
fleetman-webapp-0g1h2c-bcd90              1/1     Running   0          9m
```

**Colonnes** :
- `NAME` : Nom unique du Pod (généré automatiquement avec un suffixe aléatoire).
- `READY` : Nombre de conteneurs prêts / nombre total de conteneurs.
  - `1/1` : Un conteneur, prêt.
  - `0/1` : Un conteneur, pas encore prêt (en cours de démarrage ou erreur).
- `STATUS` : État du Pod.
  - `Running` : Fonctionne normalement.
  - `Pending` : En attente de ressources ou en cours de téléchargement de l'image.
  - `CrashLoopBackOff` : Crash répété.
  - `OOMKilled` : Tué pour manque de mémoire.
  - `Error` : Erreur.
- `RESTARTS` : Nombre de fois que le conteneur a redémarré.
  - `0` : Aucun restart (bon signe).
  - `5+` : Problème récurrent (nécessite investigation).
- `AGE` : Temps écoulé depuis la création du Pod.

**Options utiles** :
```powershell
# Voir plus de détails (IP, nœud, etc.)
kubectl get pods -o wide

# Suivre en temps réel (actualisation automatique)
kubectl get pods --watch

# Filtrer par label
kubectl get pods -l app=fleetman-webapp
```

#### 5.2.2 kubectl get services - Voir les Services

**Commande** :
```powershell
kubectl get svc
```
(`svc` est l'abréviation de `services`)

**Résultat** :
```
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
fleetman-api-gateway        NodePort    10.96.140.50     <none>        8080:30020/TCP   10m
fleetman-mongodb            ClusterIP   10.96.200.100    <none>        27017/TCP        15m
fleetman-position-simulator ClusterIP   10.96.180.80     <none>        8080/TCP         12m
fleetman-position-tracker   NodePort    10.96.160.60     <none>        8080:30010/TCP   11m
fleetman-queue              ClusterIP   10.96.170.70     <none>        8161/TCP,61616/TCP  14m
fleetman-webapp             NodePort    10.96.150.40     <none>        80:30080/TCP     9m
kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP          30d
```

**Colonnes** :
- `NAME` : Nom du Service.
- `TYPE` : Type de Service.
  - `ClusterIP` : Accès interne uniquement.
  - `NodePort` : Accès externe via un port sur le nœud.
  - `LoadBalancer` : Crée un load balancer cloud (AWS ELB, Azure Load Balancer).
- `CLUSTER-IP` : IP interne du Service dans le cluster.
- `EXTERNAL-IP` : IP externe (pour LoadBalancer uniquement, sinon `<none>`).
- `PORT(S)` : Ports exposés.
  - Format : `<port-service>:<nodePort>/TCP`
  - Exemple : `8080:30020/TCP` signifie :
    - `8080` : Port du Service (interne).
    - `30020` : NodePort (externe).
- `AGE` : Temps écoulé depuis la création.

**Comment utiliser ces informations ?**
- **ClusterIP** : Accessible depuis un Pod via `http://<service-name>:<port>`
  - Exemple : `http://fleetman-mongodb:27017`
- **NodePort** : Accessible (en théorie) via `http://localhost:<nodePort>`
  - Exemple : `http://localhost:30080`
  - Sur Docker Desktop : Nécessite `kubectl port-forward`.

#### 5.2.3 kubectl get deployments - Voir les Deployments

**Commande** :
```powershell
kubectl get deployments
```

**Résultat** :
```
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
fleetman-api-gateway        2/2     2            2           10m
fleetman-mongodb            1/1     1            1           15m
fleetman-position-simulator 2/2     2            2           12m
fleetman-position-tracker   2/2     2            2           11m
fleetman-queue              1/1     1            1           14m
fleetman-webapp             2/2     2            2           9m
```

**Colonnes** :
- `NAME` : Nom du Deployment.
- `READY` : Nombre de réplicas prêts / nombre de réplicas souhaités.
  - `2/2` : Les 2 réplicas fonctionnent.
  - `1/2` : Seulement 1 réplica prêt (problème potentiel).
- `UP-TO-DATE` : Nombre de réplicas à jour avec la dernière configuration.
- `AVAILABLE` : Nombre de réplicas disponibles pour servir du trafic.
- `AGE` : Temps écoulé depuis la création.

**Différence entre Deployment et Pod** :
- **Pod** : Instance en cours d'exécution.
- **Deployment** : Contrôleur qui gère les Pods (garantit qu'ils existent).

**Analogie** : Le Deployment est le contrat de travail (2 employés requis). Les Pods sont les employés réels.

#### 5.2.4 kubectl get pvc - Voir les PersistentVolumeClaims

**Commande** :
```powershell
kubectl get pvc
```

**Résultat** :
```
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-pvc   Bound    pvc-abc123-def4-5678-90ab-cdef12345678    5Gi        RWO            hostpath       15m
```

**Colonnes** :
- `NAME` : Nom du PVC.
- `STATUS` : État.
  - `Bound` : Lié à un PersistentVolume (stockage alloué).
  - `Pending` : En attente d'un PersistentVolume disponible.
- `VOLUME` : Nom du PersistentVolume lié.
- `CAPACITY` : Capacité de stockage.
- `ACCESS MODES` :
  - `RWO` : ReadWriteOnce (lecture/écriture par un seul nœud).
  - `RWX` : ReadWriteMany (lecture/écriture par plusieurs nœuds).
- `STORAGECLASS` : Classe de stockage utilisée.
  - `hostpath` : Stockage local sur Docker Desktop.
- `AGE` : Temps écoulé depuis la création.

---

### 5.3 kubectl describe - Détails d'une Ressource

**Commande** :
```powershell
kubectl describe <type> <nom>
```

**Exemples** :
```powershell
# Détails d'un Pod spécifique
kubectl describe pod fleetman-api-gateway-6f8c9d7b5c-abc12

# Détails d'un Service
kubectl describe svc fleetman-webapp

# Détails d'un Deployment
kubectl describe deployment fleetman-position-tracker
```

**Qu'est-ce que ça affiche ?**
- **Informations générales** : Nom, namespace, labels, annotations.
- **Spécifications** : Configuration complète (image, ports, variables d'env, volumes).
- **État actuel** : IP, nœud, QoS class.
- **Events** : Historique des événements (création, démarrage, erreurs).

**Exemple de sortie pour un Pod** :
```
Name:         fleetman-api-gateway-6f8c9d7b5c-abc12
Namespace:    default
Node:         docker-desktop/192.168.65.3
Start Time:   Thu, 30 Jan 2025 11:20:15 +0100
Labels:       app=fleetman-api-gateway
Status:       Running
IP:           10.1.0.25

Containers:
  api-gateway:
    Image:          supinfo4kube/api-gateway:1.0.1
    Port:           8080/TCP
    State:          Running
      Started:      Thu, 30 Jan 2025 11:20:30 +0100
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     250m
      memory:  512Mi
    Requests:
      cpu:        100m
      memory:     256Mi
    Environment:
      SPRING_PROFILES_ACTIVE:  production-microservice

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10m   default-scheduler  Successfully assigned default/fleetman-api-gateway-6f8c9d7b5c-abc12 to docker-desktop
  Normal  Pulling    10m   kubelet            Pulling image "supinfo4kube/api-gateway:1.0.1"
  Normal  Pulled     10m   kubelet            Successfully pulled image "supinfo4kube/api-gateway:1.0.1"
  Normal  Created    10m   kubelet            Created container api-gateway
  Normal  Started    10m   kubelet            Started container api-gateway
```

**Utilité** :
- **Debugging** : La section **Events** montre les erreurs (OOMKilled, ImagePullBackOff, CrashLoopBackOff).
- **Vérification** : Confirmer que la configuration (image, variables d'env, ressources) est correcte.

---

### 5.4 kubectl logs - Voir les Logs d'un Pod

**Commande** :
```powershell
kubectl logs <nom-pod>
```

**Exemple** :
```powershell
kubectl logs fleetman-api-gateway-6f8c9d7b5c-abc12
```

**Résultat** : Affiche tous les logs du conteneur (stdout + stderr).

**Options utiles** :
```powershell
# Suivre en temps réel (comme tail -f)
kubectl logs -f fleetman-api-gateway-6f8c9d7b5c-abc12

# Afficher les 50 dernières lignes
kubectl logs --tail=50 fleetman-api-gateway-6f8c9d7b5c-abc12

# Logs du conteneur précédent (après un restart)
kubectl logs --previous fleetman-api-gateway-6f8c9d7b5c-abc12
```

**Cas d'usage** :
- **Vérifier le démarrage** : Voir si l'application démarre correctement.
- **Debugging** : Identifier les erreurs de connexion (MongoDB, Queue).
- **Monitoring** : Suivre les requêtes HTTP entrantes.

**Exemple de logs Spring Boot sains** :
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.5.0)

2025-01-30 11:20:30.123  INFO 1 --- [           main] c.v.fleetman.ApiGatewayApplication       : Starting ApiGatewayApplication
2025-01-30 11:20:35.456  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http)
2025-01-30 11:20:35.789  INFO 1 --- [           main] c.v.fleetman.ApiGatewayApplication       : Started ApiGatewayApplication in 5.666 seconds
```

**Exemple de logs avec erreur** :
```
2025-01-30 11:15:23.456 ERROR 1 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   : 

***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.

Reason: Failed to determine a suitable driver class
```

**Diagnostic** : Configuration de base de données manquante.

---

### 5.5 kubectl port-forward - Accéder aux Services

**Commande** :
```powershell
kubectl port-forward <ressource> <port-local>:<port-distant>
```

**Exemples** :
```powershell
# Port-forward vers un Service
kubectl port-forward service/fleetman-webapp 3000:80

# Port-forward vers un Pod spécifique
kubectl port-forward pod/fleetman-webapp-0g1h2c-yza78 3000:80

# Port-forward vers un Deployment (cible un Pod aléatoire)
kubectl port-forward deployment/fleetman-webapp 3000:80
```

**Différence** :
- **Service** : Kubernetes choisit automatiquement un Pod sain.
- **Pod** : Cible un Pod spécifique (utile pour debugging).
- **Deployment** : Équivalent au Service (cible un Pod du Deployment).

**Recommandation** : Utiliser `service/` pour la stabilité.

**Flux** :
```
Navigateur (localhost:3000) → kubectl CLI → API Kubernetes → Pod (port 80)
```

**Multi-port-forward** :
```powershell
# Terminal 1
kubectl port-forward service/fleetman-webapp 3000:80

# Terminal 2
kubectl port-forward service/fleetman-api-gateway 8080:8080

# Terminal 3
kubectl port-forward service/fleetman-position-tracker 8081:8080
```

**Stopper le port-forward** : `Ctrl + C` dans le terminal.

---

### 5.6 kubectl delete - Supprimer des Ressources

**Commande** :
```powershell
kubectl delete <type> <nom>
```

**Exemples** :
```powershell
# Supprimer un Pod spécifique
kubectl delete pod fleetman-webapp-0g1h2c-yza78

# Supprimer un Service
kubectl delete svc fleetman-webapp

# Supprimer un Deployment (supprime aussi les Pods associés)
kubectl delete deployment fleetman-webapp

# Supprimer toutes les ressources d'un fichier YAML
kubectl delete -f webapp-deployment.yaml

# Supprimer toutes les ressources du répertoire
kubectl delete -f .
```

**⚠️ Attention** :
- Supprimer un **Deployment** supprime aussi tous les **Pods** associés.
- Supprimer un **PVC** peut supprimer les données persistantes (selon la politique de récupération).

**Supprimer tout et redéployer** :
```powershell
# Supprimer toutes les ressources
kubectl delete -f .

# Redéployer
kubectl apply -f .
```

---

### 5.7 kubectl exec - Exécuter une Commande dans un Pod

**Commande** :
```powershell
kubectl exec <nom-pod> -- <commande>
```

**Exemples** :
```powershell
# Lister les fichiers dans le Pod
kubectl exec fleetman-mongodb-5d7c8b9f4d-ghi56 -- ls /data/db

# Vérifier la connexion à MongoDB
kubectl exec fleetman-mongodb-5d7c8b9f4d-ghi56 -- mongo --eval "db.version()"

# Ouvrir un shell interactif
kubectl exec -it fleetman-mongodb-5d7c8b9f4d-ghi56 -- /bin/bash
```

**Options** :
- `-it` : Mode interactif (permet de taper des commandes).
- `--` : Sépare les options de kubectl des arguments de la commande.

**Cas d'usage** :
- **Debugging** : Vérifier la configuration interne.
- **Tests de connexion** : `curl` depuis un Pod vers un Service.
- **Inspection de données** : Requêtes manuelles sur MongoDB.

**Exemple : Test de connectivité** :
```powershell
# Créer un Pod temporaire pour tester le réseau
kubectl run debug --image=busybox --rm -it --restart=Never -- /bin/sh

# Dans le shell du Pod
nslookup fleetman-mongodb
curl http://fleetman-api-gateway:8080/api/vehicles
```

---

## 6. Instructions d'Accès à l'Application

Après le déploiement réussi, voici comment accéder à l'application Fleet Management.

### 6.1 Vérification de l'État du Déploiement

Avant d'accéder à l'application, vérifiez que tous les services sont opérationnels.

**Commandes de vérification** :
```powershell
# Vérifier que tous les Pods sont en état Running
kubectl get pods

# Résultat attendu : Tous les Pods doivent afficher 1/1 READY et STATUS Running
# NAME                                      READY   STATUS    RESTARTS   AGE
# fleetman-api-gateway-xxx                  1/1     Running   0          10m
# fleetman-mongodb-xxx                      1/1     Running   0          15m
# fleetman-position-simulator-xxx           1/1     Running   0          12m
# fleetman-position-tracker-xxx             1/1     Running   0          11m
# fleetman-queue-xxx                        1/1     Running   0          14m
# fleetman-webapp-xxx                       1/1     Running   0          9m

# Vérifier que tous les Services sont créés
kubectl get svc

# Résultat attendu : Tous les Services avec leurs ports
# NAME                        TYPE        CLUSTER-IP       PORT(S)
# fleetman-api-gateway        NodePort    10.96.xxx.xxx    8080:30020/TCP
# fleetman-mongodb            ClusterIP   10.96.xxx.xxx    27017/TCP
# fleetman-webapp             NodePort    10.96.xxx.xxx    80:30080/TCP
```

**Si un Pod n'est pas Running** :
```powershell
# Voir les détails du Pod
kubectl describe pod <nom-pod>

# Voir les logs
kubectl logs <nom-pod>
```

---

### 6.2 Configuration Port-Forward pour Docker Desktop

Sur Docker Desktop (Windows/Mac), les ports NodePort ne fonctionnent pas directement sur `localhost`. Il faut utiliser `kubectl port-forward`.

**Configuration recommandée : 3 terminaux PowerShell**

#### Terminal 1 : WebApp (Interface Utilisateur)
```powershell
kubectl port-forward service/fleetman-webapp 3000:80
```

**Explication** :
- `service/fleetman-webapp` : Cible le Service webapp.
- `3000:80` : Mappe le port 80 du Service vers le port 3000 de votre machine.

**Résultat dans le terminal** :
```
Forwarding from 127.0.0.1:3000 -> 80
Forwarding from [::1]:3000 -> 80
```

**Accès** : Ouvrir le navigateur sur `http://localhost:3000`

#### Terminal 2 : API Gateway (Backend REST)
```powershell
kubectl port-forward service/fleetman-api-gateway 8080:8080
```

**Explication** :
- Mappe le port 8080 du Service API Gateway vers le port 8080 local.

**Résultat** :
```
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

**Accès** : `http://localhost:8080/api/vehicles` (retourne du JSON)

#### Terminal 3 (Optionnel) : Position Tracker (Debugging)
```powershell
kubectl port-forward service/fleetman-position-tracker 8081:8080
```

**Explication** :
- Mappe le port 8080 du Position Tracker vers le port 8081 local (pour éviter le conflit avec l'API Gateway).

**Accès** : `http://localhost:8081` (endpoints internes)

---

### 6.3 Accès à l'Interface Web

Une fois le port-forward configuré pour la webapp (Terminal 1), ouvrez votre navigateur.

**URL** : `http://localhost:3000`

**Ce que vous devriez voir** :
- Une **carte interactive** (Google Maps ou OpenStreetMap).
- Des **marqueurs de véhicules** avec leurs positions en temps réel.
- Un **panneau latéral** avec la liste des véhicules.
- Les positions se mettent à jour **automatiquement** (toutes les quelques secondes).

**Fonctionnalités** :
- **Cliquer sur un véhicule** : Affiche son historique de positions.
- **Zoom/Dézoom** : Navigation sur la carte.
- **Suivi en temps réel** : Les véhicules se déplacent sur la carte.

**Si la page ne se charge pas** :
1. Vérifier que le terminal PowerShell avec `kubectl port-forward` est toujours actif.
2. Vérifier les logs de la webapp :
   ```powershell
   kubectl logs -l app=fleetman-webapp
   ```
3. Vérifier que l'API Gateway est accessible :
   ```powershell
   curl http://localhost:8080/api/vehicles
   ```

---

### 6.4 Accès à l'API Gateway (Backend)

**URL** : `http://localhost:8080`

**Endpoints disponibles** :

#### GET /api/vehicles
**Description** : Liste tous les véhicules avec leurs dernières positions connues.

**Exemple** :
```powershell
curl http://localhost:8080/api/vehicles
```

**Réponse (JSON)** :
```json
[
  {
    "name": "City Truck",
    "lat": 48.8566,
    "lng": 2.3522,
    "timestamp": "2025-01-30T11:30:00Z",
    "speed": 45
  },
  {
    "name": "Village Truck",
    "lat": 48.8606,
    "lng": 2.3376,
    "timestamp": "2025-01-30T11:30:05Z",
    "speed": 60
  }
]
```

#### GET /api/vehicles/{vehicleName}
**Description** : Détails d'un véhicule spécifique.

**Exemple** :
```powershell
curl http://localhost:8080/api/vehicles/City%20Truck
```

**Réponse** : Objet JSON avec les détails du véhicule.

#### GET /api/vehicles/{vehicleName}/history
**Description** : Historique complet des positions d'un véhicule.

**Exemple** :
```powershell
curl http://localhost:8080/api/vehicles/City%20Truck/history
```

**Réponse** : Tableau JSON avec toutes les positions enregistrées.

---

### 6.5 Accès aux Services Internes (Debugging)

Pour accéder aux services internes (MongoDB, ActiveMQ) qui utilisent ClusterIP, il faut également utiliser `kubectl port-forward`.

#### Accès à MongoDB
```powershell
kubectl port-forward service/fleetman-mongodb 27017:27017
```

**Connexion avec MongoDB Compass** (outil GUI) :
- URL : `mongodb://localhost:27017`
- Base de données : `fleetman` (créée automatiquement)
- Collection : `positions`

**Connexion en ligne de commande** :
```powershell
# Installer MongoDB client (si pas déjà fait)
# Puis :
mongo mongodb://localhost:27017

# Dans le shell Mongo
use fleetman
db.positions.find().limit(10)
```

#### Accès à ActiveMQ Admin UI
```powershell
kubectl port-forward service/fleetman-queue 8161:8161
```

**URL** : `http://localhost:8161/admin`

**Identifiants** :
- Utilisateur : `admin`
- Mot de passe : `admin`

**Ce que vous pouvez voir** :
- Nombre de messages dans la queue.
- Nombre de producteurs (position-simulator).
- Nombre de consommateurs (position-tracker).
- Statistiques de débit de messages.

---

### 6.6 Arrêter les Port-Forwards

Pour arrêter un port-forward :
1. Aller dans le terminal PowerShell concerné.
2. Appuyer sur `Ctrl + C`.

**Message affiché** :
```
Handling connection for 3000
^C
```

Pour arrêter tous les port-forwards :
- Fermer toutes les fenêtres PowerShell avec `Ctrl + C`.

---

### 6.7 Résumé de la Configuration d'Accès

| Service | Port-Forward | URL Locale | Description |
|---------|--------------|------------|-------------|
| **WebApp** | `kubectl port-forward service/fleetman-webapp 3000:80` | `http://localhost:3000` | Interface utilisateur principale |
| **API Gateway** | `kubectl port-forward service/fleetman-api-gateway 8080:8080` | `http://localhost:8080/api/vehicles` | API REST pour les données de véhicules |
| **Position Tracker** | `kubectl port-forward service/fleetman-position-tracker 8081:8080` | `http://localhost:8081` | Service backend (optionnel, pour debug) |
| **MongoDB** | `kubectl port-forward service/fleetman-mongodb 27017:27017` | `mongodb://localhost:27017` | Base de données (optionnel, pour debug) |
| **ActiveMQ Admin** | `kubectl port-forward service/fleetman-queue 8161:8161` | `http://localhost:8161/admin` | Interface d'administration de la queue |

---

## 7. Conclusion et Récapitulatif

### 7.1 Ce que Nous Avons Accompli

Nous avons déployé avec succès une **application de gestion de flotte distribuée** composée de **6 microservices** sur un cluster Kubernetes local (Docker Desktop).

**Architecture finale** :
- **MongoDB** : Base de données persistante (5 Gi de stockage).
- **ActiveMQ Queue** : File de messages pour communication asynchrone.
- **Position Simulator** : Générateur de positions GPS simulées (2 réplicas).
- **Position Tracker** : Consommateur de messages et stockage dans MongoDB (2 réplicas).
- **API Gateway** : Passerelle REST pour le frontend (2 réplicas).
- **WebApp** : Interface utilisateur avec carte interactive (2 réplicas).

**Total** : **10 Pods** en cours d'exécution.

---

### 7.2 Concepts Kubernetes Appliqués

Durant ce projet, nous avons mis en pratique les concepts Kubernetes suivants :

1. **Namespace** : Organisation logique des ressources.
2. **PersistentVolumeClaim (PVC)** : Demande de stockage persistant pour MongoDB.
3. **Deployment** : Gestion déclarative des Pods avec réplicas.
4. **Service (ClusterIP)** : Communication interne entre microservices.
5. **Service (NodePort)** : Exposition de services vers l'extérieur.
6. **Labels et Selectors** : Identification et routage des Pods.
7. **Resources (Requests/Limits)** : Gestion de la mémoire et du CPU.
8. **Environment Variables** : Configuration des applications (Spring Profiles).
9. **Volume Mounts** : Liaison des PVC aux conteneurs.
10. **DNS Kubernetes** : Résolution automatique des noms de services.

---

### 7.3 Problèmes Résolus

| Problème | Cause | Solution |
|----------|-------|----------|
| **OOMKilled** | Manque de mémoire (256 Mi insuffisant pour Spring Boot) | Augmentation des limites à 512 Mi |
| **Health Probes Échecs** | Endpoint `/actuator/health` inexistant | Suppression des probes |
| **WebApp CrashLoopBackOff** | DNS hardcodé vers namespace "default" | Migration de tous les services vers "default" |
| **NodePort Non Accessible** | Limitation Docker Desktop Windows | Utilisation de `kubectl port-forward` |
| **Pods Pending** | Mémoire Docker Desktop insuffisante (2 Go) | Augmentation à 6 Go dans les paramètres |

---

### 7.4 Architecture de Communication

**Flux de données** :

```
1. Position Simulator → Publie positions GPS → ActiveMQ Queue
2. ActiveMQ Queue → Stocke messages → En attente de consommateur
3. Position Tracker → Consomme messages → Traite positions
4. Position Tracker → Stocke données → MongoDB
5. API Gateway → Requête données → Position Tracker
6. Position Tracker → Requête base de données → MongoDB
7. MongoDB → Retourne données → Position Tracker
8. Position Tracker → Retourne JSON → API Gateway
9. API Gateway → Retourne JSON → WebApp
10. WebApp → Affiche carte → Navigateur utilisateur
```

**Communication DNS interne** :
- `fleetman-mongodb.default.svc.cluster.local:27017`
- `fleetman-queue.default.svc.cluster.local:61616`
- `fleetman-position-tracker.default.svc.cluster.local:8080`
- `fleetman-api-gateway.default.svc.cluster.local:8080`

---

### 7.5 Commandes Essentielles à Retenir

**Déploiement** :
```powershell
kubectl apply -f .
```

**Vérification** :
```powershell
kubectl get pods
kubectl get svc
kubectl get deployments
```

**Debugging** :
```powershell
kubectl describe pod <nom>
kubectl logs <nom>
kubectl logs -f <nom>  # Suivi en temps réel
```

**Accès** :
```powershell
kubectl port-forward service/fleetman-webapp 3000:80
kubectl port-forward service/fleetman-api-gateway 8080:8080
```

**Nettoyage** :
```powershell
kubectl delete -f .
```

---

### 7.6 Différences entre Développement et Production

| Aspect | Développement (Docker Desktop) | Production (Cluster réel) |
|--------|-------------------------------|---------------------------|
| **Accès** | `kubectl port-forward` | **Ingress Controller** ou **LoadBalancer** |
| **Stockage** | `hostpath` (local) | **Persistent Volumes cloud** (AWS EBS, Azure Disk) |
| **Haute Disponibilité** | Single-node cluster | **Multi-node cluster** avec anti-affinity |
| **Sécurité** | Pas d'authentification | **RBAC**, **Network Policies**, **Secrets** |
| **Monitoring** | `kubectl logs` | **Prometheus**, **Grafana**, **ELK Stack** |
| **DNS** | `.default.svc.cluster.local` | **Ingress DNS** (`fleetman.example.com`) |
| **Ressources** | Limites basses (6 Go total) | **Nœuds dédiés** avec ressources importantes |

---

### 7.7 Améliorations Possibles

Pour passer à une architecture production-ready :

1. **Sécurité** :
   - Ajouter des **Secrets** pour les mots de passe MongoDB/ActiveMQ.
   - Configurer **RBAC** (Role-Based Access Control).
   - Utiliser des **Network Policies** pour isoler les Pods.

2. **Monitoring** :
   - Déployer **Prometheus** pour collecter les métriques.
   - Installer **Grafana** pour visualiser les dashboards.
   - Ajouter des **health probes** fonctionnels (avec endpoints custom).

3. **Scalabilité** :
   - Configurer **Horizontal Pod Autoscaler** (HPA) pour ajuster les réplicas selon la charge.
   - Utiliser un **Cluster Autoscaler** pour ajouter des nœuds si nécessaire.

4. **CI/CD** :
   - Automatiser le déploiement avec **GitHub Actions**, **GitLab CI**, ou **Jenkins**.
   - Utiliser **Helm Charts** pour gérer les configurations multi-environnements.

5. **Ingress** :
   - Installer un **Ingress Controller** (Nginx, Traefik).
   - Configurer des **Ingress Rules** pour exposer les services avec des noms de domaine.
   - Ajouter **TLS/SSL** avec **cert-manager**.

6. **Base de Données** :
   - Utiliser **MongoDB Atlas** (cloud) ou un **StatefulSet MongoDB** avec réplication.
   - Configurer des **backups automatiques**.

7. **Logging** :
   - Centraliser les logs avec **ELK Stack** (Elasticsearch, Logstash, Kibana).
   - Ou utiliser **Loki** avec **Grafana**.

---

### 7.8 Ressources pour Approfondir

**Documentation officielle** :
- Kubernetes : [https://kubernetes.io/docs/](https://kubernetes.io/docs/)
- Docker Desktop : [https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)
- Spring Boot : [https://spring.io/projects/spring-boot](https://spring.io/projects/spring-boot)

**Tutoriels recommandés** :
- **Kubernetes for Beginners** : [https://kubernetes.io/docs/tutorials/](https://kubernetes.io/docs/tutorials/)
- **Play with Kubernetes** : [https://labs.play-with-k8s.com/](https://labs.play-with-k8s.com/) (lab interactif gratuit)

**Outils utiles** :
- **K9s** : Interface terminal interactive pour Kubernetes ([https://k9scli.io/](https://k9scli.io/))
- **Lens** : IDE graphique pour Kubernetes ([https://k8slens.dev/](https://k8slens.dev/))
- **Kubectl Plugins** : `kubectl krew` pour installer des plugins ([https://krew.sigs.k8s.io/](https://krew.sigs.k8s.io/))

---

### 7.9 Mot de la Fin

Félicitations ! Vous avez déployé une application microservices complexe sur Kubernetes. Ce projet vous a permis de :

- Comprendre les **concepts fondamentaux** de Kubernetes.
- Pratiquer la **création de manifests YAML**.
- Résoudre des **problèmes réels** (OOMKilled, DNS, ressources).
- Utiliser les **commandes kubectl** essentielles.
- Configurer des **services de communication** entre microservices.

**Kubernetes est puissant mais complexe**. Ce projet n'est que le début. Continuez à pratiquer, expérimenter et apprendre !

**Prochaines étapes** :
1. Expérimenter avec des modifications (changer le nombre de réplicas, ajouter un nouveau service).
2. Essayer de déployer sur un cluster cloud (Azure AKS, AWS EKS, Google GKE).
3. Implémenter une pipeline CI/CD pour automatiser les déploiements.
4. Explorer Helm pour gérer des déploiements complexes.

**Bonne continuation dans votre voyage Kubernetes ! 🚀**

---

## Annexes

### Annexe A : Structure Complète des Fichiers YAML

Tous les fichiers YAML du projet sont disponibles dans le répertoire de travail :

```
kubernetes project/
├── namespace.yaml
├── mongodb-pvc.yaml
├── mongodb-deployment.yaml
├── queue-deployment.yaml
├── position-simulator-deployment.yaml
├── position-tracker-deployment.yaml
├── api-gateway-deployment.yaml
├── webapp-deployment.yaml
├── README.md
└── RAPPORT_DETAILLE.md (ce fichier)
```

---

### Annexe B : Commandes de Troubleshooting Avancées

**Voir l'utilisation des ressources** :
```powershell
kubectl top nodes
kubectl top pods
```

**Voir les événements du cluster** :
```powershell
kubectl get events --sort-by='.lastTimestamp'
```

**Voir les logs de tous les Pods d'un Deployment** :
```powershell
kubectl logs -l app=fleetman-api-gateway --tail=50
```

**Redémarrer un Deployment** :
```powershell
kubectl rollout restart deployment fleetman-api-gateway
```

**Voir l'historique des rollouts** :
```powershell
kubectl rollout history deployment fleetman-api-gateway
```

**Tester la résolution DNS** :
```powershell
kubectl run debug --image=busybox --rm -it --restart=Never -- nslookup fleetman-mongodb
```

---

### Annexe C : Configuration Docker Desktop Recommandée

**Settings → Resources → Advanced** :
- **CPUs** : 4 (minimum 2)
- **Memory** : 6 GB (minimum 4 GB)
- **Swap** : 1 GB
- **Disk image size** : 60 GB

**Settings → Kubernetes** :
- ✅ Enable Kubernetes
- ✅ Show system containers (advanced)

**Settings → General** :
- ✅ Use the WSL 2 based engine (Windows uniquement)

---

**Fin du Rapport Détaillé**

**Date de création** : 30 janvier 2025  
**Version** : 1.0  
**Auteur** : Assistance de Déploiement Kubernetes  
**Projet** : Fleet Management Application - Kubernetes Deployment

---
