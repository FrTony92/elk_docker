## ELK full stack on docker-compose

Goal is to setup an elk cluster based on 3 nodes es01, es02 and es03 and a kibana node.

On the target machine install docker and docker-compose.

Create a directory for hosting the compose file:

    mkdir /data/elk_compose
    chown 1000:1000 /data/elk_compose

Inside the directory copy the files:
 - .env 
 - docker-compose.yml 
 - es01_elasticsearch.yml
 - kibana.yml

Because we are going to create a 3 nodes cluster, duplicate and adjust es01_elasticsearch.yml for the 2 other nodes.
Update .env to your needs for cluster name and or password provided for kibana and elastic superuser.
On the local server update the /etc/hosts in order to map local IP of the server with es01,es02 and es03.

when everything is OK start the cluster:

    chown -R 1000:1000 /data/elk_compose
    cd /data/elk_compose
    docker-compose up -d

Directory and all files must be attached to user / group 1000 / 1000 which are the one defined by elasticsearch.
Default volumes location **/var/lib/docker/volumes** will be modified and store in /data/elk_compose.
To stop stack:

    cd /data/elk_compose
    docker-compose down


**
