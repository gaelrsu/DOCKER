# 🐳 DOCKER — Fiche Mémo

---

## Commandes essentielles

| Commande | Description |
|---|---|
| `docker ps` | Liste les conteneurs **en cours d'exécution** (`-a` pour tous, y compris arrêtés) |
| `docker pull <name>:<tag>` | Télécharge une image depuis Docker Hub (ex: `docker pull nginx:latest`) |
| `docker images` | Liste toutes les images disponibles localement |
| `docker build -t nom:latest .` | Construit une image à partir du `Dockerfile` du répertoire courant (`.`) |
| `docker run -it ubuntu /bin/bash` | Lance un conteneur Ubuntu en mode **interactif** avec un terminal bash |
| `docker stop <id>` | Arrête proprement un conteneur |
| `docker rm <id>` | Supprime un conteneur arrêté |
| `docker rmi <image>` | Supprime une image locale |
| `docker logs <id>` | Affiche les logs d'un conteneur |
| `docker exec -it <id> bash` | Ouvre un terminal dans un conteneur **déjà en cours d'exécution** |

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
