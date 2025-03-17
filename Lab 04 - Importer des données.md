---
share: "true"
---

## Objectifs

Dans ce lab, vous allez :

- Ajouter un service **FileServer** dans `docker-compose.yml` pour exposer les volumes des conteneurs
- Importer des données OpenStreetMap (OSM) sur **Saint-Malo**
- Installer et activer l'extension **pgRouting** pour le réseau routier
- Styliser les **points d'intérêt (POI)** dans GeoServer
- Télécharger et intégrer des données raster marines depuis **DataShom**

---

## 1. Ajout de FileServer dans `docker-compose.yml`

Nous allons ajouter un service `filebrowser` qui permettra d'accéder aux volumes depuis une interface web.

### Modification du fichier `docker-compose.yml`

Ajoutez le bloc suivant :

```yaml
  filebrowser:
    image: hurlenko/filebrowser
    container_name: filebrowser
    restart: always
    expose:
      - "8081:80"
    environment:
      - PUID=1000
      - PGID=1000
      - FB_BASEURL=/data
    volumes:
      - postgis_data:/data/postgis
      - geoserver_data:/data/geoserver
      - geoserver_styles:/data/geoserver/styles
      - osm_data:/data/osm
    networks:
      sdi_apps:
        ipv4_address: 172.24.0.21
```

Lancez le service :

```bash
docker-compose up -d filebrowser
```

Accédez à l'interface : **[http://localhost:8081](http://localhost:8081/)**

---

## 2. Import des données OpenStreetMap sur Saint-Malo

Nous allons récupérer et importer les données OSM via `osm2pgsql`.

### 2.1 Téléchargement des données OSM

```bash
wget https://download.geofabrik.de/europe/france/bretagne-latest.osm.pbf -O saint-malo.osm.pbf
```

### 2.2 Installation de `osm2pgsql`

Si ce n'est pas encore fait :

```bash
sudo apt install osm2pgsql
```

### 2.3 Activation de l'extension `pgRouting`

Dans `pgAdmin`, exécutez :

```sql
CREATE EXTENSION pgrouting;
```

### 2.4 Import du réseau routier

```bash
osm2pgsql -d ensgdb -U ensgadmin -H localhost -P 5432 -W --create --slim --hstore --prefix saintmalo saint-malo.osm.pbf
```

Vérification :

```sql
SELECT * FROM saintmalo_line WHERE highway IS NOT NULL LIMIT 10;
```

 Autres possibilité: 

```sql
SELECT highway, ST_AsText(way) FROM planet_osm_line LIMIT 10;
```


### 2.5 Ajout d'un style SLD dans geoserver

Créez un fichier `osm_roads.sld` et ajoutez le code suivant :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<StyledLayerDescriptor version="1.0.0"
    xmlns="http://www.opengis.net/sld"
    xmlns:ogc="http://www.opengis.net/ogc"
    xmlns:xlink="http://www.w3.org/1999/xlink"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.opengis.net/sld StyledLayerDescriptor.xsd">
    
    <NamedLayer>
        <Name>osm_roads</Name>
        <UserStyle>
            <Title>OSM Road Styles</Title>
            <FeatureTypeStyle>
                <!-- Autoroutes -->
                <Rule>
                    <Name>Motorway</Name>
                    <Title>Autoroutes</Title>
                    <ogc:Filter>
                        <ogc:PropertyIsEqualTo>
                            <ogc:PropertyName>highway</ogc:PropertyName>
                            <ogc:Literal>motorway</ogc:Literal>
                        </ogc:PropertyIsEqualTo>
                    </ogc:Filter>
                    <LineSymbolizer>
                        <Stroke>
                            <CssParameter name="stroke">#ff0000</CssParameter>
                            <CssParameter name="stroke-width">5</CssParameter>
                        </Stroke>
                    </LineSymbolizer>
                </Rule>
                
                <!-- Routes principales -->
                <Rule>
                    <Name>Primary</Name>
                    <Title>Routes primaires</Title>
                    <ogc:Filter>
                        <ogc:PropertyIsEqualTo>
                            <ogc:PropertyName>highway</ogc:PropertyName>
                            <ogc:Literal>primary</ogc:Literal>
                        </ogc:PropertyIsEqualTo>
                    </ogc:Filter>
                    <LineSymbolizer>
                        <Stroke>
                            <CssParameter name="stroke">#ff9900</CssParameter>
                            <CssParameter name="stroke-width">4</CssParameter>
                        </Stroke>
                    </LineSymbolizer>
                </Rule>
            </FeatureTypeStyle>
        </UserStyle>
    </NamedLayer>
</StyledLayerDescriptor>
```

---

## 3. Import du style dans GeoServer

1. Accédez à l'interface web de GeoServer : `http://geoserver.local/web/`
2. Allez dans **Styles** et cliquez sur **Ajouter un style**
3. Choisissez le format **SLD**, puis copiez-collez le contenu du fichier `osm_roads.sld`
4. Cliquez sur **Valider** pour vous assurer qu'il n'y a pas d'erreur
5. Enregistrez le style

### **4. Application du Style à la Couche**

1. Allez dans **Couches**    
2. Sélectionnez la couche contenant les routes OSM
3. Cliquez sur **Modifier** et appliquez le style `osm_roads`    
4. Sauvegardez les modifications
5. Vérifiez le rendu sur l’aperçu GeoServer

## **Vérification et Ajustements**

- Si certaines routes ne s'affichent pas correctement, vérifiez les valeurs `highway` dans la base PostGIS
- Adaptez les couleurs et épaisseurs en fonction du rendu souhaité
- Testez l'affichage sur MapStore2 ou un client GIS compatible WMS

---

## Quizz

1. Où sont stockées physiquement les données OSM importées dans PostGIS ?
2. Quel est l'intérêt d'utiliser `pgRouting` pour les données routières ?
3. Comment réduire la taille des fichiers raster dans GeoServer ?
4. Quels sont les avantages d'un serveur de fichiers comme FileServer dans une infrastructure Docker ?

---

Félicitations ! 🎉 Vous avez appris à structurer et visualiser des données géospatiales avec Docker et GeoServer.


