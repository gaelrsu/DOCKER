# 🐳 DOCKER — Fiche Mémo

> **Docker** permet de packager et exécuter une application dans un environnement isolé appelé **conteneur**. Les conteneurs sont légers et embarquent tout ce dont l'application a besoin (code, runtime, librairies), indépendamment de l'infrastructure hôte.

---

## Concepts clés

| Concept | Description |
|---|---|
| **Image** | Package exécutable autonome contenant le code, le runtime, les outils et les librairies |
| **Conteneur** | Instance en cours d'exécution d'une image — tourne de façon identique quelle que soit l'infrastructure |
| **Docker Hub** | Registre public pour trouver et partager des images → [hub.docker.com](https://hub.docker.com) |
| **Dockerfile** | Script de construction d'une image |
| **Docker Compose** | Outil pour définir et orchestrer des applications multi-conteneurs |

---

## Commandes générales

| Commande | Description |
|---|---|
| `docker -d` | Démarre le daemon Docker |
| `docker --help` | Affiche l'aide générale (`--help` fonctionne sur toutes les sous-commandes) |
| `docker info` | Affiche les informations système de Docker |
| `docker login -u <username>` | Connexion à Docker Hub |

---

## Images

| Commande | Description |
|---|---|
| `docker build -t <nom> .` | Construit une image depuis le `Dockerfile` du répertoire courant |
| `docker build -t <nom> . --no-cache` | Construit une image **sans utiliser le cache** |
| `docker images` | Liste toutes les images disponibles localement |
| `docker pull <image>:<tag>` | Télécharge une image depuis Docker Hub (ex: `docker pull nginx:latest`) |
| `docker push <username>/<image>` | Publie une image sur Docker Hub |
| `docker search <image>` | Recherche une image sur Docker Hub |
| `docker rmi <image>` | Supprime une image locale |
| `docker image prune` | Supprime toutes les images **non utilisées** |

---

## Conteneurs

| Commande | Description |
|---|---|
| `docker run <image>` | Crée et lance un conteneur depuis une image |
| `docker run --name <nom> <image>` | Lance un conteneur avec un **nom personnalisé** |
| `docker run -p <host>:<container> <image>` | Lance un conteneur en **exposant un port** hôte vers le conteneur |
| `docker run -d <image>` | Lance un conteneur en **arrière-plan** (mode détaché) |
| `docker run -it ubuntu /bin/bash` | Lance un conteneur en mode **interactif** avec un terminal bash |
| `docker start <nom\|id>` | Démarre un conteneur arrêté |
| `docker stop <nom\|id>` | Arrête proprement un conteneur en cours d'exécution |
| `docker rm <nom\|id>` | Supprime un conteneur arrêté |
| `docker ps` | Liste les conteneurs **en cours d'exécution** |
| `docker ps --all` | Liste **tous** les conteneurs (actifs et arrêtés) |
| `docker exec -it <nom> sh` | Ouvre un shell dans un conteneur **déjà en cours d'exécution** |
| `docker logs -f <nom>` | Affiche et suit les logs d'un conteneur en temps réel |
| `docker inspect <nom\|id>` | Affiche les détails complets d'un conteneur (configuration, réseau, volumes…) |
| `docker container stats` | Affiche les statistiques de ressources (CPU, RAM…) des conteneurs actifs |

---

## Dockerfile

Le `Dockerfile` est le **plan de construction** d'une image. Il décrit, étape par étape, l'environnement de l'application.

```dockerfile
FROM python:3.9-slim        # Image de base (OS + runtime)
WORKDIR /app                # Répertoire de travail dans le conteneur
COPY requirements.txt .     # Copie le fichier de dépendances
RUN pip install -r requirements.txt  # Exécute une commande pendant le build
COPY . .                    # Copie tout le code source
CMD ["python", "app.py"]    # Commande lancée au démarrage du conteneur
```

### Instructions clés

| Instruction | Rôle |
|---|---|
| `FROM` | Image de base (obligatoire, toujours en premier) |
| `WORKDIR` | Définit le répertoire courant pour les instructions suivantes |
| `COPY` | Copie des fichiers depuis l'hôte vers l'image |
| `ADD` | Comme `COPY` mais supporte les URLs et l'extraction d'archives |
| `RUN` | Exécute une commande **pendant le build** (ex: installer des packages) |
| `ENV` | Définit une variable d'environnement |
| `EXPOSE` | Documente le port exposé par l'application |
| `CMD` | Commande par défaut au lancement du conteneur (peut être surchargée) |
| `ENTRYPOINT` | Commande principale non surchargeable (souvent combinée avec `CMD`) |

> 💡 **Bonne pratique** : combiner les `RUN` en une seule instruction pour réduire le nombre de layers et la taille de l'image.
> ```dockerfile
> RUN apt-get update && apt-get install -y curl git && rm -rf /var/lib/apt/lists/*
> ```

---

## Docker Compose

Docker Compose permet de **définir et orchestrer plusieurs conteneurs** dans un seul fichier `docker-compose.yml`. Idéal pour les environnements multi-services (app + base de données + cache, etc.).

### Commandes Compose

| Commande | Description |
|---|---|
| `docker compose up` | Démarre tous les services (`-d` pour le mode détaché) |
| `docker compose down` | Arrête et supprime les conteneurs |
| `docker compose build` | Reconstruit les images |
| `docker compose logs -f` | Affiche les logs en temps réel |
| `docker compose ps` | Liste les services en cours |
| `docker compose exec <service> bash` | Ouvre un terminal dans un service |

### Exemple — App web + base de données

```yaml
version: "3.9"

services:

  web:
    build: .                        # Construit l'image depuis le Dockerfile local
    ports:
      - "5000:5000"                 # <port_hôte>:<port_conteneur>
    volumes:
      - .:/app                      # Monte le code source en live (hot reload)
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/mydb
    depends_on:
      - db                          # Attend que le service db soit démarré

  db:
    image: postgres:15              # Image officielle PostgreSQL
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persistance des données

volumes:
  postgres_data:                    # Volume nommé géré par Docker
```

### Exemple — Stack complète (web + cache + reverse proxy)

```yaml
version: "3.9"

services:

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

  app:
    build: .
    expose:
      - "8000"
    environment:
      - REDIS_URL=redis://cache:6379
    depends_on:
      - cache

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

### Concepts clés de Compose

| Concept | Description |
|---|---|
| `services` | Chaque service = un conteneur |
| `image` | Utilise une image existante (Docker Hub) |
| `build` | Construit une image depuis un Dockerfile |
| `ports` | Expose des ports `hôte:conteneur` |
| `volumes` | Monte des fichiers/dossiers ou persiste des données |
| `environment` | Variables d'environnement |
| `depends_on` | Ordre de démarrage des services |
| `networks` | Réseau interne entre services (créé automatiquement par défaut) |

> 💡 Les services communiquent entre eux via leur **nom de service** comme hostname (ex: `db`, `cache`, `redis`).

---

## Ressources

- 📦 [Docker Hub](https://hub.docker.com) — Trouver et partager des images
- 📖 [Documentation officielle](https://docs.docker.com)
- 🧪 [Exemples de projets Docker](https://github.com/docker/awesome-compose)
- 🖥️ [Docker Desktop](https://docs.docker.com/desktop) — Disponible sur Mac, Linux et Windows
