# 🗄️ DOCKER — Persistance des données & Volumes

> Par défaut, toute donnée écrite dans un conteneur **disparaît** à sa suppression. Docker propose trois mécanismes de montage pour persister ou partager des données.

---

## Les 3 types de montage

```
   Hôte (filesystem)
   ┌─────────────────────────────────────────────┐
   │                                             │
   │  /var/lib/docker/volumes/   ←── Volume      │
   │  /home/user/monprojet/      ←── Bind Mount  │
   │  RAM (mémoire)              ←── tmpfs        │
   │                                             │
   └────────────┬────────────────────────────────┘
                │ montage
   ┌────────────▼────────────────────────────────┐
   │           Conteneur                         │
   │   /data        /app        /tmp             │
   └─────────────────────────────────────────────┘
```

| Type | Stockage | Persiste ? | Cas d'usage |
|---|---|---|---|
| **Volume** | Géré par Docker (`/var/lib/docker/volumes/`) | ✅ Oui | Production, bases de données |
| **Bind Mount** | Répertoire choisi sur l'hôte | ✅ Oui | Développement, hot reload |
| **tmpfs** | RAM de l'hôte (mémoire) | ❌ Non | Données temporaires, secrets |

---

## 1. Volumes

Les volumes sont **gérés entièrement par Docker** et constituent la solution recommandée pour la production. Ils n'augmentent pas la taille du conteneur et offrent de meilleures performances que la couche d'écriture interne.

### Commandes de gestion

| Commande | Description |
|---|---|
| `docker volume create <nom>` | Crée un volume nommé |
| `docker volume ls` | Liste tous les volumes |
| `docker volume inspect <nom>` | Affiche les détails d'un volume (chemin, driver…) |
| `docker volume rm <nom>` | Supprime un volume |
| `docker volume prune` | Supprime tous les volumes **non utilisés** |

### Utilisation avec `docker run`

```bash
# Syntaxe --mount (recommandée, plus explicite)
docker run --mount type=volume,source=mon-volume,target=/data nginx

# Syntaxe courte -v
docker run -v mon-volume:/data nginx
```

### Volumes nommés vs anonymes

```bash
# Volume NOMMÉ — persiste même après suppression du conteneur
docker run -v postgres_data:/var/lib/postgresql/data postgres:15

# Volume ANONYME — supprimé avec le conteneur si --rm est utilisé
docker run --rm -v /tmp/cache busybox
```

### Dans Docker Compose

```yaml
services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data  # volume nommé

volumes:
  postgres_data:   # déclaration du volume (géré par Docker)
```

### Sauvegarder et restaurer un volume

```bash
# Sauvegarder dans une archive tar
docker run --rm \
  -v mon-volume:/data \
  -v $(pwd):/backup \
  busybox tar czf /backup/sauvegarde.tar.gz -C /data .

# Restaurer depuis une archive tar
docker run --rm \
  -v mon-volume:/data \
  -v $(pwd):/backup \
  busybox tar xzf /backup/sauvegarde.tar.gz -C /data
```

### Partager un volume entre conteneurs

```bash
# Conteneur 1 : écrit des données
docker run -d --name writer -v shared-data:/data busybox \
  sh -c "echo 'hello' > /data/fichier.txt"

# Conteneur 2 : lit les mêmes données
docker run --rm --name reader -v shared-data:/data busybox \
  cat /data/fichier.txt
```

---

## 2. Bind Mounts

Un bind mount **lie directement un chemin de l'hôte** à un chemin dans le conteneur. Idéal en développement pour voir les modifications de code en temps réel sans rebuild.

### Utilisation avec `docker run`

```bash
# Syntaxe --mount (recommandée)
docker run --mount type=bind,source=$(pwd),target=/app node:20

# Syntaxe courte -v
docker run -v $(pwd):/app node:20
```

### Mode lecture seule

```bash
# Empêche le conteneur de modifier les fichiers de l'hôte
docker run -v $(pwd)/config:/etc/app/config:ro nginx
docker run --mount type=bind,source=$(pwd)/config,target=/etc/app/config,readonly nginx
```

### Dans Docker Compose

```yaml
services:
  app:
    build: .
    volumes:
      # Bind mount du code source (hot reload)
      - type: bind
        source: ./src
        target: /app/src
      # Bind mount d'un fichier de config
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
```

### ⚠️ Limites des bind mounts

- Dépendent de la structure de répertoires de la machine hôte → pas portables
- Les processus du conteneur peuvent modifier n'importe quel fichier monté
- Performances réduites sur **Mac et Windows** (couche VM intermédiaire)
- **Ne pas utiliser en production** — réservé au développement local

---

## 3. tmpfs (Linux uniquement)

Un tmpfs mount stocke les données **en RAM**. Les données sont perdues dès l'arrêt du conteneur. Utile pour les données sensibles (tokens, secrets) ou les caches temporaires hautes performances.

### Utilisation avec `docker run`

```bash
# Syntaxe --mount
docker run --mount type=tmpfs,destination=/tmp nginx

# Syntaxe --tmpfs (plus courte)
docker run --tmpfs /tmp nginx

# Avec options : taille limitée à 100 Mo, permissions 1770
docker run --mount type=tmpfs,destination=/cache,tmpfs-size=100m,tmpfs-mode=1770 nginx
```

### Caractéristiques

- ❌ Non partageable entre plusieurs conteneurs
- ❌ Disponible uniquement sur **Linux** (pas sur Docker Desktop Mac/Windows)
- ✅ Très rapide (accès RAM direct)
- ✅ Aucune trace sur disque après arrêt du conteneur

---

## Comparatif et guide de choix

```
La donnée doit-elle persister ?
│
├── NON (cache, fichiers temporaires, secrets)
│     └──▶ tmpfs
│
└── OUI
      │
      ├── Environnement de PRODUCTION ?
      │     └──▶ Volume (nommé)
      │
      └── Environnement de DÉVELOPPEMENT ?
            │
            ├── Besoin de modifier les fichiers en temps réel ?
            │     └──▶ Bind Mount
            │
            └── Données isolées (ex: node_modules, dépendances)
                  └──▶ Volume
```

| Critère | Volume | Bind Mount | tmpfs |
|---|---|---|---|
| Géré par Docker | ✅ | ❌ | ✅ |
| Portable entre machines | ✅ | ❌ | ✅ |
| Performances (Linux) | ✅ Élevées | ✅ Élevées | ✅✅ Très élevées |
| Performances (Mac/Win) | ✅ | ⚠️ Lentes | ❌ N/A |
| Partage entre conteneurs | ✅ | ✅ | ❌ |
| Données sensibles | ⚠️ | ⚠️ | ✅ Idéal |
| Sauvegarde facile | ✅ | ✅ | ❌ |
| Usage recommandé | Production | Développement | Secrets / cache |

---

## Inspecter les montages d'un conteneur

```bash
# Voir tous les montages (volumes, bind mounts, tmpfs)
docker inspect <conteneur> --format '{{ json .Mounts }}' | python3 -m json.tool

# Exemple de sortie pour un volume
# {
#   "Type": "volume",
#   "Name": "postgres_data",
#   "Source": "/var/lib/docker/volumes/postgres_data/_data",
#   "Destination": "/var/lib/postgresql/data",
#   "RW": true
# }
```

> 💡 **Règle rapide** : si `Type` vaut `volume` → sauvegarder. Si `bind` → la persistance dépend du chemin hôte. Si `tmpfs` → éphémère par conception.
