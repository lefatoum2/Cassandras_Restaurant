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


# Cassandras Resto

```
version: '2'

services: 
 cassa1: 
    container_name: cassa1
    image: cassandra:latest
    volumes:
      - ./cassa1:/var/lib/cassandra/data
    ports:
      - 9042:9042
    environment:
      - CASSANDRA_CLUSTER_NAME=Cluster1
      
 cassa2:
  container_name: cassa2
  image: cassandra:latest
  volumes:
      - ./cassa2:/var/lib/cassandra/data
  ports:
      - 9043:9042
  command: bash -c 'sleep 60;  /docker-entrypoint.sh cassandra -f'
  depends_on:
    - cassa1
  environment: 
      - CASSANDRA_CLUSTER_NAME=Cluster1
     
```

Puis lancez :

```

docker-compose up -d

```

## Insertion des requêtes

Pour utiliser le CQL d'un des conteneurs :

```
docker exec -it cassa2  cqlsh 

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

Ajoutez une table restaurant :
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

Ajoutez une table inspection :

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

Importez les fichiers dans le conteneur à partir de docker grâce à la commande suivante(sortir de CQLSH avec exit ) (command line Windows ou Linux) :

```

docker cp ./restaurants.csv cassa2:/

docker cp ./restaurants_inspections.csv cassa2:/

```


Importez les données du fichier restaurants.csv dans la table restaurant (commande cqlsh) :

```
USE  resto;
```

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
Uvicorn
Cassandra
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

```python
from cassandra.cluster import Cluster
import uvicorn
from fastapi import FastAPI
from fastapi.encoders import jsonable_encoder
from flask import current_app, flash, jsonify, make_response, redirect, request, url_for


class DataB:

    @classmethod
    def connexion(cls):
        cls.cluster = Cluster(['127.0.0.1'], port=9043)
        cls.session = cls.cluster.connect('resto', wait_for_all_pools=True)

    # infos d'un restaurant à partir de son id

    @classmethod
    def infos(cls, id):
        cls.connexion()
        query = f"select * from restaurant where id = {id}"
        res = cls.session.execute(query)
        list1 = []
        for row in res:
            list1.append(row)
        return list1


    #  liste des noms de restaurants à partir du type de cuisine
    @classmethod
    def noms(cls, cuisinetype):
        cls.connexion()
        query = "select * from restaurant"
        res = cls.session.execute(query)
        list1 = []
        for row in res:
            list1.append(row.name)
        return list1

    # accéder aux noms des 10 premiers restaurants d'un grade donné.
    @classmethod
    def restos_grade(cls, grade):
        dico = {}
        idrestaurant = []
        liste_grade = []
        cls.connexion()
        rows = cls.session.execute(
            f"SELECT idrestaurant FROM inspection WHERE grade= '{grade}' GROUP BY idrestaurant limit 10;")
        for row in rows:
            idrestaurant.append(row.idrestaurant)
        for id in idrestaurant:
            info = cls.session.execute(f"SELECT name  FROM restaurant WHERE id = {id};")
            info = list(info)
            dico = {id: info}
            liste_grade.append(dico)

        return liste_grade

    # nombre d'inspection d'un restaurant à partir de son id restaurant
    @classmethod
    def inspec(cls, idresto):
        cls.connexion()
        query = "select * from inspection "
        res = cls.session.execute(query)
        fg = 0
        for row in res:
            if row[0] == idresto:
                fg += 1
        return fg


app = FastAPI(redoc_url=None)


# infos d'un restaurant à partir de son id
@app.get("/infos/{id}")
async def infos(id: int):
    DataB.connexion()
    data = DataB.infos(id)
    return data


# liste des noms de restaurants à partir du type de cuisine
@app.get("/type/{cuisinetype}")
async def noms(cuisinetype: str):
    DataB.connexion()
    data = DataB.noms(cuisinetype)
    return data


# les noms des 10 premiers restaurants d'un grade donné
@app.get("/inspec/grades/{grad}")
async def grades(grad: str):
    DataB.connexion()
    data = DataB.grades(grad)
    return data


# nombre d'inspection d'un restaurant à partir de son id restaurant
@app.get("/inspec/{id}")
async def inspec(id: int):
    DataB.connexion()
    data = DataB.inspec(id)
    return data


if __name__ == "__main__":
    uvicorn.run(app, host='127.0.0.1', port=8000)

```
