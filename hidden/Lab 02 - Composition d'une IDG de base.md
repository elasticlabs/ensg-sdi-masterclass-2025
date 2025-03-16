---
share: "True"
---

## Objectifs

Ce laboratoire a pour objectif de vous apprendre à déployer une infrastructure de données géospatiales en utilisant Docker Compose. À la fin de ce lab, vous serez capables de :

- Comprendre et utiliser Docker Compose pour déployer plusieurs services
- Installer et configurer un serveur PostgreSQL avec PostGIS
- Administrer PostgreSQL avec pgAdmin 4
- Déployer et configurer GeoServer
- Importer des données OpenStreetMap dans PostGIS

## Prérequis

- Connaissances de base en Docker
- Connaissances de base en bases de données relationnelles
- Notions en systèmes d'information géographique (SIG)
- Docker et Docker Compose installés sur votre machine

---

## 1. Rappels sur Docker Compose

Docker Compose permet d'orchestrer plusieurs conteneurs Docker en définissant leurs configurations dans un fichier `docker-compose.yml`.

Les avantages de Docker Compose :

- Facilité de déploiement de plusieurs services
- Gestion des dépendances entre services
- Centralisation des configurations

Dans ce lab, nous allons utiliser Docker Compose pour déployer une infrastructure de données géospatiales composée de plusieurs services :

- **Portainer** : Interface web pour gérer les conteneurs Docker
- **PostgreSQL/PostGIS** : Base de données géospatiale
- **pgAdmin 4** : Interface web pour administrer PostgreSQL
- **GeoServer** : Serveur de diffusion de données géospatiales

---

## 2. Déploiement de Portainer

Portainer permet d'administrer Docker via une interface web. Nous allons d'abord le déployer et créer un compte administrateur.

### Créez le réseau `sdi_apps` 

Ce réseau supportera l'ensemble de nos services par la suite. Pour des raisons de simplification du développement (mais pas de la configuration 😇), les adresses IP sont fixées pour chaque service. 

```shell
docker network create --subnet=172.24.0.0/16 --driver bridge sdi_apps
```
### Ajout du service Portainer

Ajoutez le service suivant dans `docker-compose.yml` (attention! en soi, ce n'est pas du tout suffisant pour faire un fichier fonctionnel!)

```yaml
  portainer:
    image: portainer/portainer-ce
    container_name: portainer
    restart: always
    expose:
      - "9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      sdi_apps:
        ipv4_address: 172.24.0.22
```

<u>Question</u> : que devez-vous ajouter pour compléter cette configuration ? 
- Indice : pensez aux volumes, au réseau, et à la structure générale d'un fichier docker compose!

### Déploiement

Lancez Portainer avec :

```bash
docker-compose up -d portainer
```

Accédez à l'interface de Portainer : `http://172.24.0.22:9000`

1. Créez un compte administrateur
2. Connectez Portainer à l'environnement Docker local

<u>Quizz</u> : qu'elle est la différence entre les directive `expose` et `ports` ?

---

## 3. Installation de PostgreSQL avec PostGIS

Nous allons maintenant ajouter un service PostgreSQL avec l'extension PostGIS.

### Ajout du service PostgreSQL/PostGIS

Ajoutez le bloc suivant dans `docker-compose.yml` :

```yaml
  postgis:
    image: postgis/postgis:16-3.4
    container_name: postgis
    restart: always
    expose:
      - "5432"
    environment:
      POSTGRES_DB: ensgdb
      POSTGRES_USER: ensgadmin
      POSTGRES_PASSWORD: ensgpassword
      ALLOW_IP_RANGE: 0.0.0.0/0
      FORCE_SSL: FALSE
    volumes:
      - postgis_data:/var/lib/postgresql/data
    networks:
      sdi_apps:
        ipv4_address: 172.24.0.10
```

Lancez le service :

```bash
docker-compose up -d postgis
```

Vérifiez qu'il fonctionne :

```bash
docker ps
```

Testez la connexion :

```bash
docker exec -it postgis psql -U ensgadmin -d ensgdb
```

### Test de connexion depuis VS Code

1. Installez l'extension **PostgreSQL** dans VS Code.
2. Ouvrez le panneau `PostgreSQL Explorer`.
3. Cliquez sur `Add Connection` et renseignez :
    - **Host** : `172.24.0.10`
    - **Port** : `5432`
    - **User** : `ensgadmin`
    - **Password** : `ensgpassword`
    - **Database** : `ensgdb`
4. Testez la connexion.

---

## 4. Installation et configuration de pgAdmin 4

pgAdmin est une interface web pour gérer PostgreSQL.

### Ajout du service pgAdmin

Ajoutez le bloc suivant dans `docker-compose.yml` :

```yaml
  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@ensg.eu
      PGADMIN_DEFAULT_PASSWORD: ensgpassword
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    expose:
      - "5050"
    networks:
      sdi_apps:
        ipv4_address: 172.24.0.20
```

### Déploiement

```bash
docker-compose up -d pgadmin
```

Accédez à `http://172.24.0.20:5050` et connectez-vous avec :

- **Email** : `admin@ensg.eu`
- **Mot de passe** : `ensgpassword`

### Installation des extensions PostgreSQL

Dans l'interface pgAdmin, exécutez les commandes suivantes :

```sql
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
CREATE EXTENSION hstore;
CREATE EXTENSION pg_stat_statements;
```


---

## 5. Déploiement et configuration de GeoServer

GeoServer permet de diffuser des données géospatiales via des services OGC.

### Ajout du service GeoServer

Ajoutez le bloc suivant dans `docker-compose.yml` :

```yaml
  geoserver:
    image: kartoza/geoserver
    container_name: geoserver
    restart: always
    environment:
      - "CORS_ENABLED=true"
      - GEOSERVER_ADMIN_PASSWORD=geoserver
      - GEOSERVER_ADMIN_USER=admin
      - INITIAL_MEMORY=500M
      - MAXIMUM_MEMORY=1G
      - GEOSERVER_DATA_DIR=/opt/geoserver/data_dir
      - GEOWEBCACHE_CACHE_DIR=/opt/geoserver/data_dir/gwc
      - ROOT_WEBAPP_REDIRECT=true
      - TOMCAT_EXTRAS=false
      - SAMPLE_DATA=false
      # Extensions set to be installed
      - "INSTALL_EXTENSIONS=true"
      - STABLE_EXTENSIONS=css-plugin,importer-plugin,wmts-multi-dimensional-plugin
      - COMMUNITY_EXTENSIONS=backup-restore-plugin,ogcapi-plugin,smart-data-loader-plugin,wmts-styles-plugin
    expose:
      - "8080"
    volumes:
      - geoserver_data:/opt/geoserver/data_dir
      - geoserver_bal:/opt/geoserver/data_dir/_BAL_
      - geoserver_settings:/settings
    networks:
      sdi_apps:
        ipv4_address: 172.24.0.11
```

### Déploiement

```bash
docker-compose up -d geoserver
```

Accédez à `http://172.24.0.11:8080/geoserver` et connectez-vous avec :

- **Utilisateur** : `admin`
- **Mot de passe** : `geoserver`

### Connexion à PostgreSQL

1. Dans GeoServer, allez dans `Stores > Add new Store`
2. Choisissez `PostGIS`
3. Remplissez les champs :
    - **Database** : `ensgdb`
    - **User** : `ensgadmin`
    - **Password** : `ensgpassword`
    - **Host** : `postgis`
    - **Port** : `5432`
4. Cliquez sur `Save`

---

## 6. Import de données OpenStreetMap dans PostGIS

Nous allons importer des données OSM de la région Bretagne dans PostGIS.

### Téléchargement des données

```bash
wget https://download.geofabrik.de/europe/france/bretagne-latest.osm.pbf
```

### Installation de osm2pgsql

```bash
sudo apt install osm2pgsql
```

### Importation des données dans PostGIS

```bash
osm2pgsql -d ensgdb -U ensgadmin -H 172.24.0.10 -P 5432 -W --create --slim --hstore bretagne-latest.osm.pbf
```

### Vérification

```sql
SELECT * FROM planet_osm_point LIMIT 10;
```

---
### Quiz Final

1. **Où se situent physiquement les données OSM importées ?**
    
    - Quelle notion de Docker permet de conserver les données d’un conteneur après son arrêt ?
        
    - Comment retrouver le chemin des volumes Docker sur votre machine ?
        
2. **Quelle est la taille des images, des conteneurs et des volumes ?**
    
    - Quelle commande permet d’afficher la taille des images Docker ?
        
    - Quelle commande permet de voir la taille des volumes Docker ?
        
    - Où peut-on retrouver ces informations dans Portainer ?
        
3. **Comment optimiser la taille des images Docker ?**
    
    - Quels sont les éléments qui influencent la taille d’une image Docker ?
        
    - Quelles stratégies peuvent être mises en place pour réduire la taille des images ?
        
    - Que pensez-vous des images multi-stage builds dans ce contexte ?
        

Répondez aux questions en exécutant les commandes nécessaires et en réfléchissant aux bonnes pratiques d’optimisation. 🚀

## Conclusion

Félicitations ! Vous avez mis en place une infrastructure de données géospatiales avec Docker Compose. Vous pouvez maintenant exploiter ces données dans vos applications SIG.