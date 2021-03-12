# Cassandras_Restaurant

## Contexte du projet

Le CRKI est contacté par un client qui a un gros besoin en big data. Il souhaite centraliser dans une base de données, les résultats des inspections des restaurants. La base est appelée à grossir de façon continue. Il faut donc une technologie scalable, sans perdre en temps de réponse. La solution semble être Cassandra et c'est ce que vous allez devoir prouver en déployant un cluster.

Le plus simple, pour prouver la faisabilité du projet, est de créer ce cluster avec des conteneurs. Puis de préparer une API pour tester l'accessibilité et le temps de réponse de la base.

## Modalités pédagogiques

En binôme vous allez préparer un docker-compose qui permet de déployer un cluster Cassandra avec 2 noyaux. Il faut impérativement un volume pour garantir la persistance des données.

Le cluster monté, vous devez suivre les instructions pour créer la base et importer les données csv.

Enfin, coder une petit API qui propose 4 url pour accéder :

aux infos d'un restaurant à partir de son id,
à la liste des noms de restaurants à partir du type de cuisine,
au nombre d'inspection d'un restaurant à partir de son id restaurant,
les noms des 10 premiers restaurants d'un grade donné.
Et en bonus, intégrer l'API dans le docker-compose.

## Critères de performance

Les noeuds et les volumes se construisent bien à partir du docker-compose. L'API retourne les bonnes données.

## Modalités d'évaluation

Démonstration au formateur.

## Livrables

Un lien vers un github qui contient le docker-compose du cluster et le code de l'API.


## Création du cluster de 2 base de données Cassandra

docker-compose.yml

```

version: '2'

services:
 cassa1:
    container_name: cassa1
    image: cassandra:latest
    volumes:
      - ./Cassandras_Restaurant:/var/lib/cassandra/data
    ports:
      - 9042:9042
    environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter1
 cassa2:
  container_name: cassa2
  image: cassandra:latest
  volumes:
      - ./Cassandras_Restaurant:/var/lib/cassandra/data
  ports:
      - 9043:9042
  command: bash -c 'sleep 60;  /docker-entrypoint.sh cassandra -f'
  depends_on:
    - cassa1
  environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter1
      - CASSANDRA_SEEDS=cassa1
      
```

Deuxième version du docker-compose:

```
version: '2'

services:
 cassa1:
    container_name: cassa1
    image: cassandra:latest
    volumes:
      - ./Cassandras_Restaurant:/var/lib/cassandra/data
    ports:
      - 9042:9042
    environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter
      
    command: CREATE KEYSPACE IF NOT EXISTS resto WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor': 2};
    command: USE resto;
    command: create table restaurant (id int,
               name text,
               borough text,
               buildingnum text,
               street text,
               zipcode int,
               phone text,
               cuisinetype text,
               primary key (id));

    command: create table inspection (idrestaurant int,
               inspectiondate date,
               violationcode text,
               violationdescription text,
               criticalflag text,
               zipcode int,
               score int,
               grade text,
               primary key (idrestaurant,inspectiondate));
    command: CREATE INDEX IF NOT EXISTS index_cuisinetype ON resto.restaurant (cuisinetype);

    command: CREATE INDEX IF NOT EXISTS index_inspection ON resto.inspection (grade);
    command: docker cp ./restaurants.csv cassa1:/
    command: docker cp ./restaurants_inspections.csv cassa1:/
    command: COPY Restaurant (id, name, borough, buildingnum, street, zipcode, phone, cuisinetype) FROM 'restaurants.csv' WITH DELIMITER=',';
    command: COPY Inspection (idrestaurant, inspectiondate, violationcode, violationdescription, criticalflag, score, grade) FROM 'restaurants_inspections.csv' WITH           DELIMITER=',';
    
 cassa2:
  container_name: cassa2
  image: cassandra:latest
  volumes:
      - ./Cassandras_Restaurant:/var/lib/cassandra/data
  ports:
      - 9043:9042
  command: bash -c 'sleep 60;  /docker-entrypoint.sh cassandra -f'
  depends_on:
    - cassa1
  environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter1
      - CASSANDRA_SEEDS=cassa1
```
Puis lancez :

```

docker-compose up -d

```

## Insertion des requêtes

Pour utiliser le CQL, il faut entrer dans la bash du conteneur :
```
docker exec -it cassa1 /bin/bash

```

Le client, en ligne de commandes, de Cassandra se nomme CQLSH :

```
# cqlsh

```
Création d'un keyspace:

```

CREATE KEYSPACE IF NOT EXISTS resto WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor': 2};

```

Descriptif des Keyspaces:
```
DESCRIBE keyspaces;
DESCRIBE KEYSPACE resto;
```

Positionnez-vous sur le keyspace: 
```
USE resto;

```
Ajouter une table restaurant :
```
   id => int et primary key
   name => varchar
   borough => varchar,
   buildingnum => varchar,
   street => varchar,
   zipcode => int,
   phone => text,
   cuisinetype => varchar
```
```
create table restaurant (id int,
name text,
borough text,
buildingnum text,
street text,
zipcode int,
phone text,
cuisinetype text,
primary key (id));

```
Ajouter une table inspection :
```
   idrestaurant => int et primary key
   inspectiondate => date et primary key,
   violationcode => varchar,
   violationdescription => varchar,
   criticalflag => varchar,
   score => int,
   grade => varchar
```

```
create table inspection (idrestaurant int,
inspectiondate date,
violationcode text,
violationdescription text,
criticalflag text,
zipcode int,
score int,
grade text,
primary key (idrestaurant,inspectiondate));

```
Créez un index sur cuisinetype de la table restaurant:
```
CREATE INDEX IF NOT EXISTS index_cuisinetype
ON resto.restaurant (cuisinetype);
```

Créez un index sur grade de la table inspection:

```
CREATE INDEX IF NOT EXISTS index_inspection
ON resto.inspection (grade);
```

Importez les fichiers dans le conteneur à partir de docker grâce à la commande suivante :

```

docker cp ./restaurants.csv cassa1:/

docker cp ./restaurants_inspections.csv cassa1:/

```

Importez les données du fichier restaurants.csv dans la table restaurant (commande cqlsh) :

```
COPY Restaurant (id, name, borough, buildingnum, street, zipcode, phone, cuisinetype) FROM 'restaurants.csv' WITH DELIMITER=',';
```

Importez les données du fichier restaurants_inspections.csv dans la table inspection (en vous inspirant de la dernière commande):

```
COPY Inspection (idrestaurant, inspectiondate, violationcode, violationdescription, criticalflag, score, grade) FROM 'restaurants_inspections.csv' WITH DELIMITER=',';

```

## Création du dockerfile de l'API :

```
FROM python:3
RUN pip install -r requirements.txt
CMD python api.py

```

## Création du requirements.txt :

```
Flask
FastAPI
Optional
Uvicorn
Flask-cqlalchemy
CQL
```

Docker-compose.yml mis à jour:

```
version: '2'

services:
 cassa1:
    container_name: cassa1
    image: cassandra:latest
    volumes:
      - ./Cassandras_Restaurant:/var/lib/cassandra/data
    ports:
      - 9042:9042
    environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter1
 cassa2:
  container_name: cassa2
  image: cassandra:latest
  volumes:
      - ./Cassandras_Restaurant:/var/lib/cassandra/data
  ports:
      - 9043:9042
  command: bash -c 'sleep 60;  /docker-entrypoint.sh cassandra -f'
  depends_on:
    - cassa1
  environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter1
      - CASSANDRA_SEEDS=cassa1

  app:
    container_name: api1
    build:
      context: ./
      dockerfile: ./Dockerfile
    ports:
      - "5000:5000"
    links:
      - cassa1


```
## Création de api.py

```
import cql
from typing import Optional
from fastapi import FastAPI
import uvicorn  # ASGI server
from fastapi.encoders import jsonable_encoder
from urllib.parse import urlparse

class DataB:
    @classmethod
    def connexion(cls):
        host = '127.0.0.1'
        port = '9042'
        keyspace = 'resto'
        cls.con = cql.connect(host, port, keyspace)
        cls.cursor = cls.con.cursor()

    @classmethod
    def disconnect(cls):
        cls.cursor.close()
        cls.con.close()

    # infos d'un restaurant à partir de son id
    @classmethod
    def infos(cls, id):
        cls.connexion()
        query = "select * from restaurant where id = %s ", [id]
        res = cls.cursor.execute(query)
        return res

    #  liste des noms de restaurants à partir du type de cuisine
    @classmethod
    def noms(cls, cuisinetype):
        cls.connexion()
        query = "select name from restaurant where type = %s", [cuisinetype]
        res = cls.cursor.execute(query)
        return res

    # les noms des 10 premiers restaurants d'un grade donné
    @classmethod
    def grades(cls, grad):
        cls.connexion()
        query = "select name from restaurant inner join inspection on restaurant.id = inspection.idrestaurant  where grade = %s limit 10", [
            grad]
        res = cls.cursor.execute(query)
        return res

    # nombre d'inspection d'un restaurant à partir de son id restaurant
    @classmethod
    def inspec(cls, id):
        cls.connexion()
        query = "select count(inspectiondate) from inspection where idrestaurant = %s", [id]
        res = cls.cursor.execute(query)
        return res


app = FastAPI(redoc_url=None)


# infos d'un restaurant à partir de son id
@app.get("/infos/{id}")
async def infos(id: int):
    DataB.connexion()
    data = DataB.infos(id)
    DataB.disconnect()
    return data


# liste des noms de restaurants à partir du type de cuisine
@app.get("/infos/type/{cuisinetype}")
async def noms(cuisinetype: str):
    DataB.connexion()
    data = DataB.noms(cuisinetype)
    DataB.disconnect()
    return data


# les noms des 10 premiers restaurants d'un grade donné
@app.get("/inspec/grades/{grad}")
async def grades(grad: str):
    DataB.connexion()
    data = DataB.grades(grad)
    DataB.disconnect()
    return data


# nombre d'inspection d'un restaurant à partir de son id restaurant
@app.get("/inspec/{id}")
async def inspec(id: int):
    DataB.connexion()
    data = DataB.inspec(id)
    DataB.disconnect()
    return data


if __name__ == "__main__":
    uvicorn.run(app, port=8000)


```

