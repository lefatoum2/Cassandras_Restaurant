# Cassandras Resto

```
version: '2'

services: 
 cas1: 
    container_name: cas1
    image: cassandra:latest
    volumes:
      - ./cassa1:/var/lib/cassandra/data
    ports:
      - 9042:9042
    environment:
      - CASSANDRA_CLUSTER_NAME=MyCluster
      
 cas2:
  container_name: cas2
  image: cassandra:latest
  volumes:
      - ./cassa2:/var/lib/cassandra/data
  ports:
      - 9043:9042
  command: bash -c 'sleep 60;  /docker-entrypoint.sh cassandra -f'
  depends_on:
    - cas1
  environment: 
      - CASSANDRA_CLUSTER_NAME=MyCluster
     
```
