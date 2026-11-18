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
4. [Gestion du Stockage Persistant (Stateful vs Stateless)](#4-gestion-du-stockage-persistant-stateful-vs-stateless)
5. [Sécurisation des Images et de l'Exécution](#5-sécurisation-des-images-et-de-lexécution)
6. [Impact sur l'Agilité et la Scalabilité](#6-impact-sur-lagilité-et-la-scalabilité)
7. [Évolutions Avancées : Observabilité & Supply Chain](#7-évolutions-avancées--observabilité--supply-chain)

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

> **Architecture hybride en production :** En environnement Cloud, les conteneurs s'exécutent généralement **à l'intérieur** de VM Linux durcies — combinant la frontière d'isolation d'infrastructure des VM avec la vélocité et la densité des conteneurs.

---

## 4. Gestion du Stockage Persistant (Stateful vs Stateless)

Un conteneur est **éphémère par conception** : toute donnée écrite dans son système de fichiers interne est perdue lors de sa suppression ou mise à jour.

* **Développement local (`Named Volumes`) :** Utilisation de volumes nommés gérés par Docker dans `docker-compose.yml` (ex. `sql_data:/var/opt/mssql`) pour découpler le cycle de vie des données de celui du conteneur.
* **Production Cloud (`Services Managés / PaaS`) :** Les conteneurs applicatifs restent **100 % sans état (*stateless*)** et se connectent à des bases de données managées externes (ex. Azure SQL, PostgreSQL managé) qui gèrent nativement les sauvegardes et la haute disponibilité.
* **Configuration externalisée :** Aucune chaîne de connexion ou clé API n'est intégrée à l'image. Elles sont injectées dynamiquement au démarrage via variables d'environnement ou gestionnaires de secrets (*Docker Secrets*, *Azure Key Vault*).

---

## 5. Sécurisation des Images et de l'Exécution

L'application du principe *Security by Design* réduit la surface d'attaque dès la construction :

* **Builds multi-étapes (*Multi-stage builds*) :** Séparation stricte entre l'image de compilation (contenant le SDK et le code source) et l'image finale d'exécution (ne contenant que les binaires compilés).
* **Images de base minimales (*Chiseled* / *Distroless*) :** Suppression des outils système superflus (`bash`, `curl`, gestionnaires de paquets) exploitables en cas d'intrusion.
* **Exécution `non-root` obligatoire :** Définition explicite d'un utilisateur sans privilèges (`USER $APP_UID`) dans le `Dockerfile` afin d'éviter l'exécution sous `root` (UID 0).
* **Système de fichiers en lecture seule & Scans CVE :** Exécution avec `--read-only` et analyse automatisée des vulnérabilités dans le pipeline CI/CD (via **Trivy** ou **Docker Scout**) avant tout déploiement.

---

## 6. Impact sur l'Agilité et la Scalabilité

* **Vélocité CI/CD & *Zero-Fear Releases* :** Grâce au cache des couches Docker et à l'immuabilité des artefacts, les builds sont rapides, reproductibles de bout en bout et les retours arrière (*rollbacks*) sont immédiats.
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

## Contribution & Références

Ce guide est évolutif et sert de support d'architecture et de bonnes pratiques de développement Cloud-Native.
