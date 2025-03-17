---
share: "true"
---

## Objectifs

Ce lab a pour objectif d'améliorer l'organisation et la gestion de votre infrastructure de données géospatiales en utilisant :

- Un fichier `.env` pour centraliser les variables de configuration
- Un `Makefile` pour simplifier l'exécution des commandes courantes

---

## 1. Centralisation des variables dans un fichier `.env`

Nous allons extraire les variables de configuration de `docker-compose.yml` et les placer dans un fichier `.env`.

### Création du fichier `.env`

Créez un fichier `.env` à la racine de votre projet et ajoutez les lignes suivantes :

```env
#
# -> Project name
COMPOSE_PROJECT_NAME=ensg-sdi

# -> Proxy docker networks
#  - APPS_NETWORK will be the network name you use in deployed compose services
APPS_NETWORK=sdi_apps

# -> PostGIS
# kartoza/postgis env variables https://github.com/kartoza/docker-postgis
POSTGIS_VERSION=16-3.4
POSTGRES_DB=ensgdb
POSTGRES_USER=ensgadmin
POSTGRES_PASS=ensgpassword
ALLOW_IP_RANGE=0.0.0.0/0
POSTGRES_PORT=32767
# -> pgAdmin4
PGADMIN_DEFAULT_EMAIL=admin@ensg.eu
PGADMIN_DEFAULT_PASSWORD=ensgpassword

# -> Geoserver
GS_VERSION=2.24.2
GEOSERVER_ADMIN_USER=admin
GEOSERVER_ADMIN_PASSWORD=geoserver
# https://docs.geoserver.org/latest/en/user/datadirectory/setting.html
GEOSERVER_DATA_DIR=/opt/geoserver/data_dir
# https://docs.geoserver.org/latest/en/user/geowebcache/config.html#changing-the-cache-directory
GEOWEBCACHE_CACHE_DIR=/opt/geoserver/data_dir/gwc
# Show the tomcat manager in the browser
TOMCAT_EXTRAS=false
ROOT_WEBAPP_REDIRECT=true
# https://docs.geoserver.org/stable/en/user/production/container.html#optimize-your-jvm
INITIAL_MEMORY=500M
MAXIMUM_MEMORY=1G
# Data and extensions
SAMPLE_DATA=false
# Full compatibility list : https://github.com/kartoza/docker-geoserver/blob/master/build_data/ Look for *_plugins.txt
STABLE_EXTENSIONS=css-plugin,importer-plugin,wmts-multi-dimensional-plugin
COMMUNITY_EXTENSIONS=backup-restore-plugin,ogcapi-plugin,smart-data-loader-plugin,wmts-styles-plugin
```

### Modification de `docker-compose.yml`

Modifiez votre `docker-compose.yml` pour utiliser ces variables :

```yaml
environment:
  POSTGRES_DB: ${POSTGRES_DB}
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Faites de même pour pgAdmin et GeoServer, pour l'ensemble des variables d'environnement

---

## 2. Création d'un `Makefile` pour automatiser les tâches

Nous allons maintenant créer un `Makefile` pour simplifier l'exécution des commandes Docker Compose.

### Création du fichier `Makefile`

Créez un fichier `Makefile` à la racine du projet et ajoutez les commandes suivantes :

```make
# Charge les variables d'environnement
include .env
export $(shell sed 's/=.*//' .env)

# Lancement des services
up:
	docker-compose up -d

down:
	docker-compose down

restart:
	docker-compose restart

logs:
	docker-compose logs -f

ps:
	docker-compose ps

clean:
	docker system prune -f
```

### Utilisation du `Makefile`

Exécutez les commandes suivantes pour tester :

```bash
make up
make logs
make ps
make down
```

---

## 3. Vérification et tests

1. Vérifiez que `docker-compose` charge bien les variables de `.env` :
    
    ```bash
    docker-compose config
    ```
    
2. Assurez-vous que votre infrastructure se lance correctement avec `make up`
3. Testez la connexion à PostgreSQL et pgAdmin

---

## 4. Quizz

1. Où se trouvent maintenant les variables d'environnement de notre infrastructure ?
2. Quelle commande permet de lister les services en cours d'exécution ?
3. Quelle commande du `Makefile` permet de redémarrer les services ?
4. Comment éviter d'avoir des fichiers de configuration sensibles dans un dépôt Git ?
5. Quelle est la commande pour voir la configuration complète après interpolation des variables d’environnement dans `docker-compose.yml` ?

---

## Conclusion

Vous avez maintenant une infrastructure mieux organisée avec un `.env` pour centraliser les configurations et un `Makefile` pour automatiser les tâches courantes ! 🎉