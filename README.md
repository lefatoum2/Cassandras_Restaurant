# Cassandras_Restaurant

# Création du cluster de 2 base de données Cassandra

```

version: '2'

services:
 cas1:
    container_name: cas1
    image: cassandra:latest
    volumes:
      - /Development/PetProjects/CassandraTut/data/node1:/var/lib/cassandra/data
    ports:
      - 9042:9042
    environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter1
 cas2:
  container_name: cas2
  image: cassandra:latest
  volumes:
      - /Development/PetProjects/CassandraTut/data/node2:/var/lib/cassandra/data
  ports:
      - 9043:9042
  command: bash -c 'sleep 60;  /docker-entrypoint.sh cassandra -f'
  depends_on:
    - cas1
  environment:
      - CASSANDRA_START_RPC=true
      - CASSANDRA_CLUSTER_NAME=MyCluster
      - CASSANDRA_ENDPOINT_SNITCH=GossipingPropertyFileSnitch
      - CASSANDRA_DC=datacenter1
      - CASSANDRA_SEEDS=cas1
      - 
```

# Insertion des requêtes
