# 



# RAPPORT DE CONTENEURISATION
## Application de Vote Distribuée

---

**Matière** : 3DOKR - Developing and Delivering Software with Docker

**Étudiant** : Pierre-Xavier VELON

---

# Table des matières

1. [Description du projet](#1-description-du-projet)
2. [Modifications apportées](#2-modifications-apportées)
3. [Dockerfiles créés](#3-dockerfiles-créés)
4. [Docker Compose](#4-docker-compose)
5. [Lancement de l'application](#5-lancement-de-lapplication)
6. [Docker Swarm](#6-docker-swarm)

---

## Description

Cette application permet de voter en direct entre deux options. Elle est composée de plusieurs services:

**vote** : Interface web Python permettant aux utilisateurs de voter (port 8080)

**worker** : Service .NET qui récupère les votes depuis Redis et les enregistre dans PostgreSQL

**result** : Interface web Node.js affichant les résultats en temps réel (port 8888)

**redis** : Système de cache et file de messages pour transmettre les votes

**postgres** : Base de données pour stocker les votes de manière permanente

**Fonctionnement** : L'utilisateur vote via l'interface web → Le vote est stocké dans Redis → Le worker récupère le vote et l'enregistre dans PostgreSQL → L'interface de résultats affiche les statistiques en temps réel.



## Modifications

Pour que l'application fonctionne dans des conteneurs Docker, il a fallu remplacer toutes les références à `localhost` par des variables d'environnement. Cela permet aux services de se connecter entre eux en utilisant les noms des conteneurs Docker.

### Module vote (Python)

Le code utilisait `localhost` pour se connecter à Redis. J'ai modifié le code pour utiliser la variable d'environnement `REDIS_HOST` qui prendra la valeur `redis` (nom du service Docker) au lieu de `localhost`.

### Module worker (.NET)

Le code contenait des chaînes de connexion codées en dur pour PostgreSQL et Redis. J'ai remplacé ces valeurs par des variables d'environnement (`POSTGRES_HOST`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `REDIS_HOST`). J'ai également mis à jour la logique de reconnexion pour utiliser ces variables.

### Module result (Node.js)

La chaîne de connexion PostgreSQL était codée en dur. J'ai remplacé cela par des variables d'environnement pour permettre la connexion via le nom du service Docker.

**Résultat** : L'application peut maintenant fonctionner dans Docker en utilisant les noms des services au lieu d'adresses IP fixes.

---

## 3. Dockerfiles créés

J'ai créé un Dockerfile pour chaque module de l'application. Chaque Dockerfile respecte les bonnes pratiques :

### vote/Dockerfile

Utilise une image Python 3.11 spécifique, crée un utilisateur non-root pour la sécurité, installe les dépendances avant de copier le code pour optimiser le cache, et expose le port 8080.

### worker/Dockerfile

Utilise un build multi-stage avec une image pour compiler le code .NET et une autre plus légère pour l'exécution, crée un utilisateur non-root, et réduit la taille finale de l'image.

### result/Dockerfile

Utilise une image Node.js 18 spécifique, crée un utilisateur non-root, utilise `npm ci` pour une installation reproductible, et expose le port 8888.

Tous les Dockerfiles incluent un fichier `.dockerignore` pour exclure les fichiers inutiles du contexte de build.

---

## 4. Docker Compose

Le fichier `docker-compose.yml` permet de lancer tous les services ensemble avec une seule commande. Il configure les healthchecks pour vérifier que les services sont prêts, gère les dépendances pour assurer l'ordre de démarrage, utilise des volumes pour la persistance des données PostgreSQL, sépare les services en deux réseaux (frontend et backend), et configure les variables d'environnement pour les connexions entre services.

---

## 5. Lancement de l'application

### Docker Compose

```bash
docker-compose up -d
```

**Accès aux applications** :
Interface de vote : http://localhost:8080
Interface de résultats : http://localhost:8888

**Commandes utiles** :
```bash
docker-compose ps
docker-compose logs -f
docker-compose down
docker-compose down -v
```

---

## 6. Docker Swarm

Docker Swarm permet de déployer l'application sur plusieurs machines pour assurer la haute disponibilité. J'ai configuré un cluster avec 1 nœud manager et 2 nœuds workers.

### Préparation

J'ai utilisé 3 instances WSL2 (Ubuntu, Debian, Kali Linux) pour simuler 3 machines distinctes. Sur chaque machine, j'ai installé Docker Engine directement.

**Installation de Docker** :
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo service docker start
sudo usermod -aG docker $USER
exit
```

### Initialisation

**Sur le manager** :
```bash
hostname -I
docker swarm init --advertise-addr <IP_MANAGER>
docker swarm join-token worker
```

**Sur chaque worker** :
```bash
docker swarm join --token <TOKEN> <IP_MANAGER>:2377
```

**Vérification** :
```bash
docker node ls
```

### Images

**Sur chaque nœud** :
```bash
cd voting-app
docker build -t voting-app-vote ./vote
docker build -t voting-app-worker ./worker
docker build -t voting-app-result ./result
```

### Réseaux et volumes

**Sur le manager** :
```bash
docker network create --driver overlay --attachable frontend
docker network create --driver overlay --attachable backend
docker volume create postgres_data
```

### Déploiement

**Sur le manager** :
```bash
docker stack deploy -c docker-compose.swarm.yml voting-app
```

### Vérification

```bash
docker stack services voting-app
docker stack ps voting-app
docker service logs voting-app_vote
docker service logs voting-app_worker
docker service logs voting-app_result
```

### Accès

Les applications sont accessibles via l'adresse IP de n'importe quel nœud du cluster :
Vote : http://<IP_ANY_NODE>:8080
Résultats : http://<IP_ANY_NODE>:8888

### Tests

```bash
docker service ls
```

Arrêter un nœud worker et vérifier que les services continuent de fonctionner. Grâce aux 2 replicas de `vote` et `result`, l'application reste accessible même si un nœud quitte le cluster.

---

## Conclusion

L'application a été entièrement conteneurisée avec Docker. Les modifications du code source sont minimales et permettent à l'application de fonctionner dans un environnement conteneurisé. L'utilisation de Docker Compose simplifie le déploiement en local, tandis que Docker Swarm permet de déployer l'application sur plusieurs machines avec haute disponibilité.

Toutes les bonnes pratiques ont été respectées : utilisation d'utilisateurs non-root, healthchecks, volumes pour la persistance, réseaux isolés, et configuration de replicas pour la résilience.

---

**Fin du rapport**
