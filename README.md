# Guide Complet : Docker & Architecture de Conteneurisation

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge)
![Architecture](https://img.shields.io/badge/Architecture-Cloud_Native-0078D4?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-By_Design-2EA44F?style=for-the-badge)

Ce dépôt constitue une base de référence technique sur les fondamentaux de **Docker**, l'architecture de la **conteneurisation**, les stratégies de **sécurisation d'images** (*Security by Design*), la gestion des **données persistantes** et les bonnes pratiques de déploiement en production.

---

## Table des matières

1. [Fondamentaux et Terminologie Docker](#1-fondamentaux-et-terminologie-docker)
2. [Fonctionnement Interne (Sous le capot)](#2-fonctionnement-interne-sous-le-capot)
3. [Conteneurs Docker vs Machines Virtuelles (VM)](#3-conteneurs-docker-vs-machines-virtuelles-vm)
4. [Gestion du Stockage Persistant et des Données (Stateful)](#4-gestion-du-stockage-persistant-et-des-données-stateful)
5. [Sécurisation des Images et de l'Exécution (Security by Design)](#5-sécurisation-des-images-et-de-lexécution-security-by-design)
6. [Impact sur l'Agilité et la Scalabilité](#6-impact-sur-lagilité-et-la-scalabilité)
7. [Évolutions Avancées : Observabilité & Supply Chain](#7-évolutions-avancées--observabilité--supply-chain)
8. [Exemples Pratiques](#8-exemples-pratiques)
9. [Cheatsheet : Commandes Docker & Docker Compose Essentielles](#9-cheatsheet--commandes-docker--docker-compose-essentielles)

---

## 1. Fondamentaux et Terminologie Docker

La conteneurisation élimine la **dérive d'environnement** (*environment drift*) et le syndrome du *« ça marche sur ma machine »* en empaquetant le code applicatif avec ses dépendances exactes dans un artefact immuable.

* **Dockerfile :** Fichier déclaratif définissant étape par étape l'image de base, les dépendances système, les binaires compilés et la commande de démarrage.
* **Image :** Modèle en lecture seule, immuable et versionné (identifié par un digest cryptographique `SHA-256`), composé d'une superposition de couches (*layers*).
* **Registry :** Dépôt centralisé (Docker Hub, Azure Container Registry, GitHub Container Registry) permettant de stocker, versionner et distribuer les images.
* **Conteneur :** Instance d'exécution isolée d'une image. Il ajoute une fine couche inscriptible éphémère au-dessus des couches en lecture seule de l'image.

---

## 2. Fonctionnement Interne (Sous le capot)

Docker repose sur une architecture client-serveur utilisant la **virtualisation au niveau du système d'exploitation** :

1. **Docker CLI (`docker`) & Daemon (`dockerd`) :** Le client envoie des requêtes REST API au démon en arrière-plan, qui délègue l'exécution à `containerd` et `runc`.
2. **OverlayFS (Union File System) :** Les images empilent des couches mises en cache. Lors d'un build ou d'un téléchargement, seules les couches modifiées sont transférées.
3. **Isolation par le noyau Linux (sans OS invité) :**
   * **Namespaces :** Isolent *ce qu'un processus peut voir* (PID, interfaces réseau, points de montage, utilisateurs).
   * **Control Groups (`cgroups`) :** Limitent *ce qu'un processus peut consommer* (quotas CPU, mémoire RAM, I/O disque) pour éviter qu'un conteneur ne sature l'hôte.

---

## 3. Conteneurs Docker vs Machines Virtuelles (VM)

| Critère | Machines Virtuelles (Hyperviseur) | Conteneurs Docker (OS-Level) |
| :--- | :--- | :--- |
| **Couche de virtualisation** | Virtualise le **matériel** (CPU, RAM, disque) via un hyperviseur (Hyper-V, KVM, VMware). | Virtualise le **système d'exploitation** en partageant le noyau (*kernel*) de l'hôte. |
| **Empreinte système** | Lourde : chaque VM embarque un OS invité complet (plusieurs Go) à maintenir et patcher. | Légère : contient uniquement l'application et ses bibliothèques minimales (quelques dizaines de Mo). |
| **Temps de démarrage** | **Minutes** (initialisation matérielle + boot complet de l'OS). | **Millisecondes à quelques secondes** (démarrage comme un processus standard). |
| **Densité & Coûts** | Ressources figées (RAM/CPU pré-alloués), faible densité par serveur. | Allocation dynamique, densité 5 à 10x supérieure sur une même infrastructure. |
| **Frontière d'isolation** | Isolation matérielle stricte au niveau du noyau (idéale pour le multi-tenant non fiable). | Isolation au niveau des processus (sécurisée via durcissement `non-root`, `seccomp`, `AppArmor`). |

> **Coexistence pragmatique VM / Conteneurs :** L'adoption des conteneurs ne signifie pas la disparition totale des machines virtuelles. Nous conservons explicitement l'option VM pour les cas nécessitant une véritable frontière d'OS invité complète — comme les applications *legacy* non conteneurisables à court terme ou les charges de travail exigeant un noyau d'OS non supporté par l'hôte — tout en adoptant les conteneurs par défaut pour tous nos nouveaux services API afin de maximiser la reproductibilité et la vélocité opérationnelle.

---

## 4. Gestion du Stockage Persistant et des Données (Stateful)

Pour lever les verrous liés à la persistance et à la sécurité de manière rigoureuse, nous appliquons le principe de **séparation stricte entre le code (stateless) et les données/secrets (stateful)**, couplé à une approche *Security by Design*.

Un conteneur est une instance d’exécution créée à partir d’une image. Comme un conteneur contient une couche inscriptible, il est **éphémère par nature** : cette couche inscriptible est détruite dès qu'il est supprimé ou mis à jour. Ne jamais stocker de données critiques à l'intérieur du système de fichiers d'un conteneur.

* **Named Volumes pour le développement local :** En local (`docker-compose.yml`), nous découplons le cycle de vie de la base de données de celui du conteneur via des volumes nommés gérés par Docker (ex. `sql_data:/var/opt/mssql`). Le conteneur peut être détruit et recréé sans perte de données.
* **Services managés en production (PaaS) :** En production cloud (ex. Azure), la meilleure pratique consiste à **ne pas héberger la base de données dans un conteneur**, mais à connecter nos API conteneurisées à un service managé externe (Azure SQL Database, PostgreSQL managé). Cela délègue les sauvegardes, la haute disponibilité et les mises à jour au fournisseur cloud tout en gardant nos conteneurs 100 % sans état (*stateless*).
* **Externalisation de la configuration :** Aucune chaîne de connexion n'est codée en dur. Elles sont injectées dynamiquement au démarrage via des variables d'environnement ou montées en mémoire via des gestionnaires de secrets (Docker Secrets, Azure Key Vault).

---

## 5. Sécurisation des Images et de l'Exécution (Security by Design)

La sécurité d’un conteneur dépend surtout de l’image dont il est issu : nous durcissons l’image dès la construction du `Dockerfile` :

* **Builds multi-étapes (*Multi-stage builds*) :** Nous séparons l'environnement de compilation (qui contient le SDK, les outils de build et le code source) de l'image finale d'exécution. L'image déployée ne contient que les binaires compilés et le runtime strictement nécessaire.
* **Images de base minimales (*Distroless* / *Chiseled*) :** Au lieu d'une distribution Linux complète embarquant `bash`, `curl` ou des gestionnaires de paquets exploitables par un attaquant, nous utilisons des images ultra-allégées (ex. images Ubuntu Chiseled de Microsoft pour .NET ou Alpine).
* **Exécution non-root obligatoire :** Par défaut, un processus dans un conteneur tourne en tant que `root` (UID 0). Nous imposons systématiquement l'instruction `USER` dans le `Dockerfile` (ex. `USER $APP_UID`) afin qu'une faille applicative ne donne jamais de privilèges administrateur sur le conteneur ou l'hôte.
* **Scan de vulnérabilités (CVE) en CI/CD et système de fichiers en lecture seule :** Chaque image est scannée automatiquement avant déploiement (via **Trivy** ou **Docker Scout**) pour bloquer toute faille critique, et exécutée avec l'option `--read-only` pour empêcher toute modification malveillante de fichiers à chaud.

---

## 6. Impact sur l'Agilité et la Scalabilité

* **Flux de développement harmonisé :** Les développeurs démarrent en local des piles multi-services strictement identiques via une simple commande `docker compose up`, éliminant la configuration manuelle, les conflits de dépendances et l'effet *« ça marche sur ma machine »*.
* **Vélocité CI/CD & *Zero-Fear Releases* :** Le pipeline passe de la maintenance de VM avec état à la promotion d'images immuables vérifiées par empreinte `SHA-256` de la préproduction vers la production, permettant des déploiements sans interruption (*zero-downtime*) et des retours arrière (*rollbacks*) instantanés.
* **Mise à l'échelle élastique (*Auto-scaling*) :** Le démarrage quasi instantané d'images légères permet aux orchestrateurs (Kubernetes, Azure Container Apps) d'absorber les pics de charge en quelques secondes.
* **Portabilité Multi-Cloud :** Le format standardisé OCI garantit l'exécution identique de l'application sur poste local, serveur on-premise ou n'importe quel fournisseur Cloud sans modification de code.

---

## 7. Évolutions Avancées : Observabilité & Supply Chain

1. **Observabilité Standardisée (OpenTelemetry) :**
   * Écriture des logs au format **JSON structuré** vers `stdout`/`stderr` pour collecte centralisée.
   * Intégration d'**OpenTelemetry** (traces distribuées et métriques) et exposition de sondes de santé (`/health/live`, `/health/ready`).
2. **Sécurité de la *Supply Chain* :**
   * **SBOM (*Software Bill of Materials*) :** Génération automatique de l'inventaire complet des paquets et dépendances embarqués lors du build CI.
   * **Signature cryptographique (Cosign / Notation) :** Validation de l'authenticité et de l'intégrité des images avant exécution en production.
3. **Orchestration Serverless & Expérience Développeur (DX) :**
   * Déploiement sur **Azure Container Apps** (mise à l'échelle événementielle KEDA, *scale-to-zero*).
   * Standardisation des environnements de travail via **Dev Containers** (`devcontainer.json`) et accélération des builds CI via le cache distant **BuildKit**.

---

## 8. Exemples Pratiques

Un exemple complet d'architecture conteneurisée pour une **API Web .NET et SQL Server** (incluant un `Dockerfile` multi-stage durci avec image Chiseled, exécution `non-root`, `docker-compose.yml` avec volume nommé et isolation réseau) est disponible dans [`examples/dotnet-sqlserver`](./examples/dotnet-sqlserver).

---

## 9. Cheatsheet : Commandes Docker & Docker Compose Essentielles

### Cycle de vie des Images (Build & Registry)

| Action | Commande | Description |
| :--- | :--- | :--- |
| **Construire une image** | `docker build -t monapi:1.0 .` | Construit une image à partir du `Dockerfile` courant avec un tag explicite. |
| **Build sans cache** | `docker build --no-cache -t monapi:1.0 .` | Force la reconstruction complète de toutes les couches (*layers*). |
| **Lister les images** | `docker images` | Affiche les images locales, leurs tags, identifiants et tailles. |
| **Inspecter les couches** | `docker history monapi:1.0` | Affiche la taille et l'instruction de chaque couche de l'image. |
| **Pousser vers un Registry** | `docker push monregistry.azurecr.io/monapi:1.0` | Publie l'image vers un registre distant (Docker Hub, ACR, GHCR). |

### Exécution et Gestion des Conteneurs

| Action | Commande | Description |
| :--- | :--- | :--- |
| **Lancer un conteneur** | `docker run -d -p 5000:8080 --name api monapi:1.0` | Démarre un conteneur en arrière-plan (`-d`) avec mappage de port. |
| **Conteneurs actifs** | `docker ps` | Liste les conteneurs en cours d'exécution (`-a` pour inclure les arrêtés). |
| **Arrêter / Démarrer** | `docker stop api` / `docker start api` | Arrête proprement (signal `SIGTERM`) ou redémarre un conteneur existant. |
| **Supprimer un conteneur** | `docker rm -f api` | Force l'arrêt et la suppression d'un conteneur. |

### Orchestration Multi-Services (Docker Compose)

| Action | Commande | Description |
| :--- | :--- | :--- |
| **Démarrer la stack** | `docker compose up -d --build` | Reconstruit les images modifiées et lance tous les services en arrière-plan. |
| **État des services** | `docker compose ps` | Affiche l'état, les ports et le statut `healthcheck` de chaque service. |
| **Arrêter la stack** | `docker compose down` | Arrête et supprime les conteneurs et réseaux (conserve les volumes nommés). |
| **Reset complet (Données)** | `docker compose down -v` | Supprime les conteneurs, réseaux **et les volumes nommés** (réinitialise la BDD). |

### Débogage, Logs et Inspection

| Action | Commande | Description |
| :--- | :--- | :--- |
| **Suivre les logs en direct** | `docker compose logs -f api` | Affiche les sorties `stdout`/`stderr` en temps réel (`--tail 100` pour limiter). |
| **Ouvrir un shell interactif** | `docker exec -it api /bin/sh` | Ouvre un terminal dans le conteneur (indisponible sur les images *Chiseled*/*Distroless*). |
| **Inspecter la configuration** | `docker inspect api` | Retourne toutes les métadonnées JSON (IP interne, volumes montés, variables, état). |
| **Consommation ressources** | `docker stats` | Monitore en temps réel l'utilisation CPU, RAM et réseau de chaque conteneur. |
| **Lister les volumes / réseaux** | `docker volume ls` / `docker network ls` | Vérifie la présence des volumes persistants et des réseaux isolés. |

### Nettoyage et Maintenance du Disque

| Action | Commande | Description |
| :--- | :--- | :--- |
| **Espace disque utilisé** | `docker system df` | Affiche l'espace occupé par les images, conteneurs, volumes et cache de build. |
| **Nettoyer les images orphelines** | `docker image prune -a` | Supprime toutes les images non associées à un conteneur actif. |
| **Purger le cache de build** | `docker builder prune` | Libère l'espace disque consommé par le cache interne de BuildKit. |
| **Nettoyage global sécurisé** | `docker system prune` | Supprime conteneurs arrêtés, réseaux inutilisés et images *dangling* (épargne les volumes). |