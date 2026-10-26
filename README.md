# Yousra AMEUR 
# TP Part 01 et 02
---

# TP Part 01 – Docker

## Introduction

L'objectif de ce TP est de déployer une application à trois niveaux (*3-tier application*) en utilisant Docker :

- **Database :** PostgreSQL 17.2 Alpine
- **Backend :** API REST Spring Boot avec Java 21
- **HTTP Server :** Apache configuré comme reverse proxy

Nous avons d'abord exécuté les conteneurs séparément, puis utilisé Docker Compose pour orchestrer l'application et Docker Hub pour publier les images.

## 1. Database – PostgreSQL

### Mise en place

Nous avons créé une image PostgreSQL à partir de `postgres:17.2-alpine`, puis démarré un conteneur en utilisant des variables d'environnement pour configurer la base.

```bash
docker build -t my-database ./database
docker network create app-network
```

Nous avons utilisé Adminer sur `http://localhost:8090` pour accéder à la base de données.

Deux scripts SQL permettent d'initialiser la base :

- `01-CreateScheme.sql` : création des tables `departments` et `students`.
- `02-InsertData.sql` : insertion des départements et des étudiants.

Ces scripts sont copiés dans `/docker-entrypoint-initdb.d/` afin d'être exécutés automatiquement lors de la première initialisation de PostgreSQL.

### Question 1-1 — Why use `-e` instead of environment variables in the Dockerfile?

L'option `-e` permet de séparer la configuration de l'image Docker. Nous pouvons ainsi modifier les paramètres sans reconstruire l'image et utiliser la même image dans plusieurs environnements.

Cela évite également d'inscrire directement les identifiants dans le Dockerfile.

### Question 1-2 — Why do we need a volume for PostgreSQL?

Un volume permet de conserver les données même si le conteneur est supprimé ou recréé.

Nous avons utilisé un volume nommé `postgres-data`, monté sur `/var/lib/postgresql/data`, afin d'assurer la persistance des données PostgreSQL.

### Question 1-3 — Document the database Dockerfile and essential commands

Notre Dockerfile utilise PostgreSQL 17.2 Alpine et copie les scripts SQL d'initialisation dans `/docker-entrypoint-initdb.d/`.

Les commandes essentielles sont :

```bash
docker build -t my-database ./database
docker ps
docker logs my-database-container
docker volume ls
```

**Résultat :** la base PostgreSQL contient les départements IRC, ETI et CGP ainsi que les étudiants insérés par les scripts SQL.

---

## 2. Backend API – Spring Boot

### Mise en place

Nous avons commencé par exécuter une application Java Hello World, puis créé une API Spring Boot avec Java 21 et Maven.

Pour construire le backend, nous avons utilisé un Dockerfile multistage avec les images :

- `eclipse-temurin:21-jdk-alpine` pour la compilation.
- `eclipse-temurin:21-jre-alpine` pour l'exécution.

Nous avons également utilisé `mvn dependency:go-offline` afin d'améliorer la gestion du cache des dépendances Maven.

### Question 1-4 — Why do we need a multistage build? Explain the Dockerfile.

Un multistage build permet de séparer la compilation de l'exécution afin de réduire la taille de l'image finale.

Notre Dockerfile comporte deux étapes :

**Build stage :** installation de Maven, copie de `pom.xml` et du code source, téléchargement des dépendances et compilation avec `mvn package -DskipTests`.

**Run stage :** utilisation d'une image JRE plus légère, copie du fichier JAR généré et lancement avec `java -jar myapp.jar`.

Cette méthode évite de conserver les outils de compilation dans l'image finale.

### Connexion à PostgreSQL

Le backend communique avec PostgreSQL grâce au réseau Docker et à l'URL JDBC :

```text
jdbc:postgresql://database:5432/db
```

Nous avons testé l'endpoint :

```text
GET /departments/IRC/students
```

**Résultat obtenu :**

```json
[
  {
    "id": 1,
    "firstname": "Eli",
    "lastname": "Copter",
    "department": {
      "id": 1,
      "name": "IRC"
    }
  }
]
```

Ce résultat confirme que Spring Boot récupère correctement les données PostgreSQL.

---

## 3. HTTP Server – Apache

### Mise en place

Nous avons créé une image Apache HTTP Server avec une page `index.html` et un fichier de configuration personnalisé `httpd.conf`.

Apache a ensuite été configuré comme reverse proxy pour rediriger les requêtes vers le backend Spring Boot.

```apache
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://backend:8080/
    ProxyPassReverse / http://backend:8080/
</VirtualHost>
```

Les modules `proxy_module` et `proxy_http_module` ont été activés.

### Question 1-5 — Why do we need a reverse proxy?

Un reverse proxy sert d'intermédiaire entre le client et le backend.

Il permet de centraliser les requêtes, de masquer les services internes, de gérer HTTPS et de faciliter le load balancing.

Dans notre application, Apache reçoit les requêtes HTTP et les transmet à Spring Boot.

**Résultat :** l'API est accessible à travers Apache, sans avoir besoin d'accéder directement au backend.

---

## 4. Docker Compose

### Mise en place

Nous avons créé un fichier `docker-compose.yml` pour orchestrer les trois services :

- `database`
- `backend`
- `httpd`

Nous avons également configuré un réseau `app-network`, un volume `postgres-data`, un healthcheck PostgreSQL et une politique de redémarrage `unless-stopped`.

Seul Apache expose son port 80 sur la machine hôte. Le backend et PostgreSQL communiquent uniquement à travers le réseau Docker.

### Question 1-6 — Why is Docker Compose so important?

Docker Compose permet de gérer plusieurs conteneurs à partir d'un seul fichier YAML.

Il simplifie leur démarrage, leur arrêt, leur configuration et leur communication.

Dans notre TP, il permet de lancer toute l'application avec une seule commande.

### Question 1-7 — Document the most important Docker Compose commands

| Commande | Description |
|---|---|
| `docker compose up -d --build` | Construire et démarrer les services |
| `docker compose ps` | Vérifier l'état des conteneurs |
| `docker compose logs` | Consulter les logs |
| `docker compose stop` | Arrêter les services |
| `docker compose down` | Supprimer les conteneurs et réseaux du projet |
| `docker compose config` | Vérifier la configuration Compose |

### Question 1-8 — Document your Docker Compose file

Notre fichier définit :

- **`backend` :** construit l'API Spring Boot et attend que PostgreSQL soit healthy.
- **`database` :** construit PostgreSQL, initialise la base et utilise un volume persistant.
- **`httpd` :** construit Apache et expose le port 80.
- **`app-network` :** permet la communication entre les services.
- **`postgres-data` :** conserve les données PostgreSQL.
- **`healthcheck` :** vérifie la disponibilité de PostgreSQL avec `pg_isready`.
- **`restart: unless-stopped` :** permet le redémarrage automatique des conteneurs après un arrêt inattendu.

Nous avons également créé un fichier `.env` pour externaliser les variables `POSTGRES_USER`, `POSTGRES_PASSWORD` et `POSTGRES_DB`.

Ce fichier est exclu du dépôt Git grâce à `.gitignore`.

### Résultat final

Après avoir lancé :

```bash
docker compose up -d --build
```

Nous avons vérifié le fonctionnement de l'application avec :

```text
http://localhost/departments/IRC/students
```

L'API a retourné les informations de l'étudiant Eli Copter.

**Checkpoint validé :** application à trois niveaux fonctionnelle avec Docker Compose.

---

## 5. Publish – Docker Hub

### Publication des images

Nous avons utilisé Docker Hub pour publier les images de nos trois services.

Après authentification avec `docker login`, nous avons attribué des tags de version aux images.

### Question 1-9 — Document your publication commands and images

```bash
docker login

docker tag tp-devops-database ussi77/my-database:1.0
docker tag tp-devops-backend ussi77/my-backend:1.0
docker tag tp-devops-httpd ussi77/my-httpd:1.0

docker push ussi77/my-database:1.0
docker push ussi77/my-backend:1.0
docker push ussi77/my-httpd:1.0
```

Les trois repositories créés sur Docker Hub sont :

- [ussi77/my-database](https://hub.docker.com/r/ussi77/my-database)
- [ussi77/my-backend](https://hub.docker.com/r/ussi77/my-backend)
- [ussi77/my-httpd](https://hub.docker.com/r/ussi77/my-httpd)

Les repositories ont été vérifiés. Les tags et leur contenu doivent être revérifiés avant la remise finale.

### Question 1-10 — Why do we put our images into an online repository?

Un registre en ligne comme Docker Hub permet de stocker, versionner et partager les images Docker.

Il facilite le déploiement sur différentes machines, le travail en équipe et l'intégration dans des pipelines CI/CD.

Les images peuvent être téléchargées avec `docker pull` sans devoir reconstruire l'application.

---

## Conclusion

Ce TP nous a permis de découvrir la création d'images Docker, la gestion des conteneurs, les réseaux, les volumes, les Dockerfiles multistage et l'orchestration avec Docker Compose.

Nous avons obtenu une application à trois niveaux fonctionnelle, composée de PostgreSQL, Spring Boot et Apache.

L'API est accessible depuis `http://localhost/departments/IRC/students` et retourne correctement les données de la base PostgreSQL.

Enfin, nous avons créé les trois repositories Docker Hub destinés à partager nos images.


---

# TP Part 02 – GitHub Actions / CI-CD

## Introduction

L'objectif de ce TP est d'automatiser les tests, la construction et la publication des images Docker grâce à GitHub Actions.

Nous avons également intégré SonarQube Cloud afin d'analyser la qualité et la sécurité du code Java.

Le workflow est défini dans le fichier `.github/workflows/main.yml`.

## 1. Continuous Integration – Backend Tests

Nous avons configuré GitHub Actions pour exécuter automatiquement les tests de notre backend Spring Boot.

Le workflow utilise :
- Ubuntu comme environnement d'exécution.
- Java JDK 21.
- Maven pour compiler et tester l'application.
- Testcontainers pour les tests d'intégration avec PostgreSQL.

Commande utilisée :

```bash
mvn clean verify --file backend/simpleapi/pom.xml
```

### Question 2-1 — What are Testcontainers?

Testcontainers est une bibliothèque Java permettant de démarrer automatiquement des conteneurs Docker temporaires pendant les tests d'intégration.

Dans notre projet, elle permet de tester le backend avec une véritable base PostgreSQL, sans devoir configurer manuellement une base de test.

Les conteneurs sont supprimés après les tests.

## 2. GitHub Actions – Secured Variables

Nous avons configuré des secrets dans GitHub :

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `SONAR_TOKEN`

Ces secrets sont utilisés par le workflow sans exposer leurs valeurs dans le code source.

### Question 2-2 — For what purpose do we need to use secured variables?

Les variables sécurisées permettent de protéger les informations sensibles, notamment les tokens d'authentification et les identifiants.

Elles évitent de stocker ces informations directement dans les fichiers du dépôt GitHub.

GitHub Actions peut ainsi s'authentifier auprès de Docker Hub et SonarQube Cloud de manière sécurisée.

## 3. Continuous Delivery – Docker Images

Nous avons ajouté un deuxième job GitHub Actions nommé `build-and-push-docker-image`.

Ce job construit et publie les images Docker de notre application sur Docker Hub.

Il dépend du job `test-backend` grâce à l'instruction :

```yaml
needs: test-backend
```

### Question 2-3 — Why did we put `needs` on the Docker job?

L'instruction `needs` permet de définir une dépendance entre deux jobs GitHub Actions.

Dans notre projet, le job Docker attend que les tests du backend réussissent avant de commencer.

Cela évite de publier des images Docker si les tests échouent.

### Question 2-4 — For what purpose do we need to push Docker images?

Publier les images Docker sur Docker Hub permet de les stocker, les partager et les réutiliser pour déployer l'application sur d'autres machines.

Cela automatise la distribution des nouvelles versions et facilite le processus de Continuous Delivery.

## 4. SonarQube Cloud – Code Quality Analysis

Nous avons intégré SonarQube Cloud dans notre pipeline GitHub Actions.

L'analyse est exécutée automatiquement avec Maven et utilise le secret `SONAR_TOKEN`.

Commande d'analyse :

```bash
mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
  -Dsonar.projectKey=yousra-ai-dotcom_devops-docker \
  -Dsonar.organization=yousra-ai-dotcom \
  -Dsonar.host.url=https://sonarcloud.io \
  --file backend/simpleapi/pom.xml
```

### Résultats SonarQube Cloud

Lors de l'analyse du backend `simple-api`, nous avons obtenu :

| Indicateur | Résultat |
|---|---|
| Quality Gate | Passed |
| Security | D |
| Reliability | A |
| Maintainability | A |
| Code Coverage | 53,6 % |
| Duplications | 0,0 % |
| Security Issues | 5 |
| Dependency Risks | 130 |

Le Quality Gate est validé, mais les problèmes de sécurité et les risques liés aux dépendances restent des points à améliorer.

## 5. Résultats GitHub Actions

Le workflow `CI devops 2025` a été exécuté avec succès sur la branche `master`.

Les deux jobs ont réussi :

- `test-backend` : compilation, tests Maven et analyse SonarQube.
- `build-and-push-docker-image` : construction et publication des images Docker.

Le workflow permet ainsi d'automatiser les principales étapes de CI/CD.

## Conclusion

Ce TP nous a permis de mettre en place un pipeline CI/CD avec GitHub Actions.

Nous avons automatisé les tests du backend Java, la construction et la publication des images Docker, ainsi que l'analyse de qualité du code avec SonarQube Cloud.

Le pipeline GitHub Actions fonctionne correctement. Des améliorations restent possibles concernant la sécurité et la couverture des tests.

---
# TP3 — Ansible : Inventaire et commandes de base

### 1. Configuration de l'inventaire Ansible

L'inventaire Ansible permet de définir les serveurs distants à administrer ainsi que les paramètres nécessaires à leur connexion.

Nous avons créé le fichier `ansible/inventories/setup.yml` :

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: /path/to/private/key
  children:
    prod:
      hosts:
        IP:
```

**Explication :**

- `all` : représente l'ensemble des serveurs.
- `vars` : définit les variables de connexion.
- `ansible_user` : précise l'utilisateur SSH (`admin`).
- `ansible_ssh_private_key_file` : indique le chemin de la clé privée SSH.
- `prod` : groupe des serveurs de production.
- `hosts` : liste les serveurs à administrer.

### 2. Vérification de la connexion Ansible

Commande exécutée depuis le dossier `ansible` :

```bash
ansible all -i inventories/setup.yml -m ping
```

Cette commande utilise le module `ping` pour vérifier la connexion et l'authentification SSH avec le serveur distant.

**Résultat obtenu :**
- `SUCCESS` : connexion réussie.
- `ping: pong` : Ansible peut communiquer avec le serveur et exécuter son module.
- `changed: false` : aucune modification effectuée.

### 3. Récupération des informations du serveur (Facts)

Commande :

```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Le module `setup` permet de récupérer automatiquement des informations sur le serveur distant, appelées *facts*.

**Résultats obtenus :**
- Système d'exploitation : Debian
- Version : 12.7
- Nom de version : Bookworm

Cette commande permet de connaître les caractéristiques du serveur avant de le configurer.

### 3-2 — Documentation du playbook Ansible

Le playbook `ansible/playbook.yml` permet d'automatiser la configuration du serveur distant.

Il utilise les paramètres suivants :

- `hosts: all` : cible les serveurs définis dans l'inventaire.
- `gather_facts: true` : récupère les informations système nécessaires, notamment la distribution Linux.
- `become: true` : permet d'exécuter les tâches avec les privilèges administrateur.
- `roles: - docker` : appelle le rôle chargé de l'installation de Docker.

Le rôle `docker`, situé dans `ansible/roles/docker/tasks/main.yml`, regroupe les tâches d'installation des dépendances, de configuration du dépôt Docker, d'installation de Docker Engine et du SDK Python, ainsi que le démarrage du service Docker.

Cette organisation sépare la configuration générale du playbook des tâches techniques. Elle facilite la maintenance, la réutilisation et l'évolution de l'automatisation.

La commande `ansible-playbook -i inventories/setup.yml playbook.yml` permet d'exécuter le playbook sur le serveur distant.

### 3-3 — Documentation de la configuration des conteneurs Docker

#### Objectif

L'objectif de cette étape est de déployer automatiquement notre application sur le serveur distant Takima à l'aide d'Ansible et de Docker.

L'application est composée de trois services : PostgreSQL pour la base de données, Spring Boot pour le backend et Apache HTTPD pour le reverse proxy.

Le déploiement est organisé en cinq rôles Ansible : `docker`, `network`, `database`, `app` et `proxy`.

#### 1. Création du réseau Docker

Le rôle `network` utilise le module `community.docker.docker_network` pour créer le réseau `app-network` avec le driver `bridge`.

Ce réseau permet aux conteneurs de communiquer entre eux en utilisant leurs noms de services, sans exposer les ports internes de PostgreSQL et Spring Boot sur le serveur.

#### 2. Déploiement de PostgreSQL

Le rôle `database` utilise le module `community.docker.docker_container` pour démarrer le conteneur PostgreSQL.

Configuration :

- **Nom du conteneur :** `database`
- **Image Docker Hub :** `ussi77/tp-devops-database:latest`
- **Réseau :** `app-network`
- **Port interne :** `5432`
- **Variables d'environnement :** `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
- **Volume :** stockage persistant des données PostgreSQL
- **Politique de redémarrage :** `unless-stopped`

Les variables d'environnement permettent de configurer les identifiants et le nom de la base de données. Le volume permet de conserver les données lorsque le conteneur est recréé.

Un healthcheck permet également de vérifier que PostgreSQL est prêt à accepter des connexions.

#### 3. Déploiement du backend Spring Boot

Le rôle `app` utilise le module `community.docker.docker_container` pour démarrer l'application Java Spring Boot.

Configuration :

- **Nom du conteneur :** `backend`
- **Image Docker Hub :** `ussi77/tp-devops-simple-api:latest`
- **Réseau :** `app-network`
- **Port interne :** `8080`
- **Variables d'environnement :**
  - `SPRING_DATASOURCE_URL`
  - `SPRING_DATASOURCE_USERNAME`
  - `SPRING_DATASOURCE_PASSWORD`
- **Politique de redémarrage :** `unless-stopped`

L'URL JDBC utilise le nom du conteneur `database` pour communiquer avec PostgreSQL sur le port `5432`.

Les logs Spring Boot confirment le démarrage de Tomcat sur le port `8080` ainsi que l'établissement de la connexion à PostgreSQL.

#### 4. Déploiement du reverse proxy Apache

Le rôle `proxy` utilise le module `community.docker.docker_container` pour démarrer Apache HTTPD.

Configuration :

- **Nom du conteneur :** `httpd`
- **Image Docker Hub :** `ussi77/tp-devops-httpd:latest`
- **Réseau :** `app-network`
- **Port publié :** `80:80`
- **Politique de redémarrage :** `unless-stopped`

Apache reçoit les requêtes HTTP des utilisateurs sur le port `80` et les transmet au backend Spring Boot sur le port `8080`.

#### 5. Interpréteur Python et SDK Docker

Le playbook définit l'interpréteur Python suivant :

`ansible_python_interpreter: /opt/docker_venv/bin/python`

Cet environnement virtuel contient le SDK Docker pour Python, nécessaire aux modules `community.docker.docker_network` et `community.docker.docker_container`.

Cela permet à Ansible de gérer les réseaux et les conteneurs Docker sur le serveur distant.

#### 6. Exécution et vérification du déploiement

Le déploiement est exécuté avec la commande :

```bash
ansible-playbook -i inventories/setup.yml playbook.yml
```

Résultat obtenu :

```text
PLAY RECAP
serveur_takima : ok=13 changed=2 unreachable=0 failed=0
```

La commande suivante permet de vérifier les conteneurs :

```bash
ansible all -i inventories/setup.yml -m command -a "docker ps" --become
```

Les trois conteneurs `database`, `backend` et `httpd` sont en cours d'exécution. PostgreSQL présente également le statut `healthy`.

Enfin, l'API a été testée depuis Safari en utilisant l'adresse publique du serveur Takima :

`http://3.250.55.22`

Réponse JSON obtenue :

```json
{"id":1,"content":"Hello, World!"}
```

Ce résultat confirme que le reverse proxy Apache transmet correctement les requêtes HTTP à l'application Spring Boot, qui fonctionne avec PostgreSQL.

#### Conclusion

Le déploiement automatique de l'application a été réalisé avec succès grâce à Ansible. Les cinq rôles permettent de gérer séparément l'installation de Docker, la création du réseau et le déploiement des trois services.

L'application est accessible depuis le navigateur via l'adresse publique du serveur, ce qui valide le checkpoint « Deploy your App » du TP3.

### 3-4 — Is it really safe to deploy automatically every new image on the hub? What can I do to make it more secure?
Non, déployer automatiquement toutes les nouvelles images Docker Hub n'est pas totalement sécurisé. Une image peut contenir des bugs, des vulnérabilités ou du code non validé.
Pour sécuriser le déploiement, il faut exécuter les tests CI avant la publication, analyser les vulnérabilités, utiliser des versions d'images identifiables plutôt que de dépendre uniquement de latest, protéger les secrets avec GitHub Actions Secrets et éventuellement demander une approbation avant le déploiement en production.