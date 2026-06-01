# DevOps_julien

================================================================================
                           DOCKER DISCOVERY TP – ANSWERS
                           (Cours DevOps – Takima 2025/2026)
================================================================================

Question 1-1 : Pourquoi utiliser l’option -e pour les variables d’environnement
            plutôt que de les écrire en dur dans le Dockerfile ?

Réponse :
Les variables d’environnement définies dans un Dockerfile (via ENV) sont figées
dans l’image. Si vous y placez un mot de passe ou une clé API, toute personne
ayant accès à l’image peut les voir. De plus, il faudrait reconstruire l’image
pour changer une valeur. L’option -e (ou le fichier .env) permet de :

– Ne pas stocker de secrets dans l’image (sécurité).
– Utiliser la même image en développement, recette et production.
– Modifier le comportement d’un conteneur sans le reconstruire.

Exemple : docker run -e "DB_PASSWORD=secret" mon_image


Question 1-2 : Pourquoi attacher un volume au conteneur PostgreSQL ?

Réponse :
Par défaut, tout ce qui est écrit dans un conteneur disparaît à sa suppression.
PostgreSQL stocke ses données dans /var/lib/postgresql/data. Sans volume,
la base de données serait effacée à chaque redémarrage du conteneur.

Le volume rend les données persistantes :
– Les données survivent à l’arrêt / à la suppression du conteneur.
– On peut sauvegarder ou migrer le volume indépendamment.
– On évite les ralentissements liés au layer de stockage du conteneur.


Question 1-3 : Construire une image pour la base de données

a) Dockerfile minimal :
----------------------
FROM postgres:17.2-alpine
COPY ./init-scripts/ /docker-entrypoint-initdb.d/

b) Commandes pour construire l’image et lancer le conteneur :
--------------------------------------------------------------
# Construction
docker build -t ma-base-de-donnees .

# Création d’un réseau dédié
docker network create app-network

# Lancement du conteneur
docker run -d \
  --name ma-base \
  --network app-network \
  -v postgres_data:/var/lib/postgresql/data \
  -e POSTGRES_USER=monuser \
  -e POSTGRES_PASSWORD=monpassword \
  -e POSTGRES_DB=madb \
  ma-base-de-donnees


Question 1-4 : Pourquoi utiliser un multi‑stage build ? Détaillez chaque étape.

Réponse :
Le multi‑stage build permet de séparer l’environnement de compilation
(build) de l’environnement d’exécution. L’image finale est beaucoup plus
petite et ne contient pas les outils de construction (Maven, JDK complet,
code source, etc.).

Exemple pour une application Spring Boot :

┌─────────────────────────────────────────────────────────────────────────────┐
│ STAGE 1 : Build                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ FROM maven:3.9.9-eclipse-temurin-21 AS build                                │
│ WORKDIR /app                                                                │
│ COPY pom.xml .                                                              │
│ RUN mvn dependency:go-offline      → télécharge les dépendances (caching)   │
│ COPY src ./src                                                              │
│ RUN mvn package -DskipTests        → compile et produit le JAR              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ STAGE 2 : Runtime                                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│ FROM openjdk:21-jdk-slim           → image légère (seulement JRE)           │
│ COPY --from=build /app/target/*.jar app.jar                                 │
│ ENTRYPOINT ["java", "-jar", "/app.jar"]                                     │
└─────────────────────────────────────────────────────────────────────────────┘

L’image finale ne contient que le JAR et un JRE minimal – ni Maven, ni JDK,
ni code source.


Question 1-5 : Pourquoi utiliser un reverse proxy (Apache / Nginx) ?

Réponse :
Un reverse proxy se place devant les services internes. Dans l’architecture
3 tiers, il reçoit toutes les requêtes HTTP (port 80/443) et les répartit :

– /api/ → envoyé au backend (conteneur API)
– /*    → sert les fichiers statiques (HTML, CSS, JS)

Avantages dans Docker :
• Point d’entrée unique (pas de ports backend exposés sur la machine hôte).
• Gestion du HTTPS (terminaison SSL).
• Équilibrage de charge si plusieurs instances backend.
• Masque les détails internes (adresses IP, ports).
• Le client n’a qu’une URL à connaître.


Question 1-6 : Pourquoi Docker Compose est‑il si important ?

Réponse :
Docker Compose déclare l’ensemble des services, réseaux et volumes dans un
fichier YAML unique. Il remplace une longue série de commandes docker run
manuellement.

Avantages clés :
– Reproductibilité : toute l’équipe utilise la même configuration.
– Gain de temps : une seule commande (docker compose up) lance toute la stack.
– Gestion multi‑conteneurs simplifiée (dépendances, ordre de démarrage).
– Fichier versionné, facile à intégrer dans une CI/CD.
– Environnements de développement identiques à la production.


Question 1-7 : Commandes Docker Compose les plus importantes

Commande                     Description
-------------------------------------------------------------------------------
docker compose up            Construit, crée et démarre tous les services.
docker compose up -d         Idem, mais en arrière‑plan (détaché).
docker compose down          Arrête et supprime conteneurs, réseaux (et volumes si -v).
docker compose build         Reconstruit les images sans démarrer.
docker compose logs          Affiche les journaux (-f pour suivre en direct).
docker compose ps            Liste les conteneurs gérés par Compose.
docker compose exec <service> <commande>  Exécute une commande dans un conteneur.
docker compose stop / start  Arrête ou redémarre les services sans les supprimer.
docker compose restart       Redémarre un ou plusieurs services.
docker compose config        Valide et affiche la configuration composée.


Question 1-8 : Documentez votre fichier docker-compose.yml

Voici un exemple complet pour l’architecture 3 tiers (HTTP, backend, base de
données). Toutes les valeurs sensibles sont passées par variables d’environnement.

```yaml
version: '3.8'

services:
  database:
    build: ./database
    networks:
      - backend-network
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    restart: unless-stopped

  backend:
    build: ./simpleapi
    networks:
      - backend-network
      - frontend-network
    depends_on:
      - database
    environment:
      - DB_HOST=database
      - DB_USER=${POSTGRES_USER}
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_NAME=${POSTGRES_DB}
    restart: on-failure
    deploy:
      restart_policy:
        condition: on-failure
        max_attempts: 3

  httpd:
    build: ./http_server
    ports:
      - "80:80"
    networks:
      - frontend-network
    depends_on:
      - backend
    environment:
      - BACKEND_URL=http://backend:8080
    restart: unless-stopped

networks:
  frontend-network:   # réseau pour le proxy (exposé à l’hôte)
  backend-network:    # réseau privé pour base + backend

volumes:
  db-data:            # volume persistant pour PostgreSQL