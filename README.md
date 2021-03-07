# Cassandras_Restaurant

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

CREATE KEYSPACE IF NOT EXISTS Movies WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor': 2};

```
