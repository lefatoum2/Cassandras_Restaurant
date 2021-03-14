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
## Création du data.py : 
```python
from cassandra.cluster import Cluster


# docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name_or_id

class DataB:

    @classmethod
    def connexion(cls):
        cls.cluster = Cluster(['127.0.0.1'], port=9043)
        cls.session = cls.cluster.connect('resto', wait_for_all_pools=True)

    # infos d'un restaurant à partir de son id

    @classmethod
    def infos(cls, id):
        cls.connexion()
        query = "select * from restaurant"
        res = cls.session.execute(query)
        for row in res:
            if row[0] == id:
                print(row)

    #  liste des noms de restaurants à partir du type de cuisine
    @classmethod
    def noms(cls, cuisinetype):
        cls.connexion()
        query = "select * from restaurant"
        res = cls.session.execute(query)
        for row in res:
            if row[3] == cuisinetype:
                print(row.name)

    # les noms des 10 premiers restaurants d'un grade donné
    @classmethod
    def grades(cls, grad):
        cls.connexion()
        query = "select * from inspection "
        res = cls.session.execute(query)
        for row in res:
            if row[3] == grad:
                print(row.idrestaurant)

    # nombre d'inspection d'un restaurant à partir de son id restaurant
    @classmethod
    def inspec(cls, idresto):
        cls.connexion()
        query = "select * from inspection "
        res = cls.session.execute(query)
        for row in res:
            if row[0] == idresto:
                fg = 0 + 1
        return fg


if __name__ == "__main__":
    DataB.inspec(41378305)
```


## Création de api.py

```python
import cql
from typing import Optional
from fastapi import FastAPI
import uvicorn  # ASGI server
from fastapi.encoders import jsonable_encoder
from urllib.parse import urlparse
from cassandra.cluster import Cluster
import cassandra
from cassandra.auth import PlainTextAuthProvider
from cassandra.cluster import Cluster


class DataB:

    @classmethod
    def connexion(cls):
        cls.cluster = Cluster(['127.0.0.1'], port=9044)
        cls.session = cls.cluster.connect(keyspace="resto")

    # infos d'un restaurant à partir de son id
    @classmethod
    def infos(cls, id):
        cls.connexion()
        query = "select * from restaurant where id = %s ", [id]
        res = cls.session.execute(query)
        return res

    #  liste des noms de restaurants à partir du type de cuisine
    @classmethod
    def noms(cls, cuisinetype):
        cls.connexion()
        query = "select name from restaurant where type = %s", [cuisinetype]
        res = cls.session.execute(query)
        return res

    # les noms des 10 premiers restaurants d'un grade donné
    @classmethod
    def grades(cls, grad):
        cls.connexion()
        query = "select name from restaurant inner join inspection on restaurant.id = inspection.idrestaurant  where grade = %s limit 10", [
            grad]
        res = cls.session.execute(query)
        return res

    # nombre d'inspection d'un restaurant à partir de son id restaurant
    @classmethod
    def inspec(cls, id):
        cls.connexion()
        query = "select count(inspectiondate) from inspection where idrestaurant = %s", [id]
        res = cls.session.execute(query)
        return res


app = FastAPI(redoc_url=None)


# infos d'un restaurant à partir de son id
@app.get("/infos/<id>")
async def infos(id: int):
    DataB.connexion()
    data = DataB.infos(id)
    return data


# liste des noms de restaurants à partir du type de cuisine
@app.get("/type/<cuisinetype>")
async def noms(cuisinetype: str):
    DataB.connexion()
    data = DataB.noms(cuisinetype)
    return data


# les noms des 10 premiers restaurants d'un grade donné
@app.get("/inspec/grades/<grad>")
async def grades(grad: str):
    DataB.connexion()
    data = DataB.grades(grad)
    return data


# nombre d'inspection d'un restaurant à partir de son id restaurant
@app.get("/inspec/<id>")
async def inspec(id: int):
    DataB.connexion()
    data = DataB.inspec(id)
    return data


if __name__ == "__main__":
    uvicorn.run(app, port=8000)

```
