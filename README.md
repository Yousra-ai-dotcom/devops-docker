# Yousra AMEUR 

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
