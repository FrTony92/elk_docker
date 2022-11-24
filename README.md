
# ELK full stack on docker-compose

## Docker checks

Check docker configuration in case of proxy

    mkdir -p /etc/systemd/system/docker.service.d
    vi /etc/systemd/system/docker.service.d/http-proxy.conf
Add those lines:
(for HTTPS_PROXY use http if you have handshake error)

    [Service]
    Environment="HTTP_PROXY=http://proxy.example.com:80"
    Environment="HTTPS_PROXY=http://proxy.example.com:443"
    Environment="NO_PROXY=localhost,127.0.0.1"

restart docker:

    systemctl daemon-reload
    systemctl restart docker
    systemctl show --property=Environment docker

In order to make curl working inside containern, you must define proxy also in docker config.

    vi ~/.docker/config.json

Add those lines:
```
{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://161.89.61.209:8080",
     "httpsProxy": "http://161.89.61.209:8080",
     "noProxy": "127.0.0.1,localhost,es01,161.89.70.0/24"
   }
 }
}
```

Goal is to setup an elk cluster based on 3 nodes es01, es02 and es03 and a kibana node.

On the target machine install docker and docker-compose.

## ELK Stack 7.17.4 - docker-compose

Sources:
Stack 7.17.x => https://www.elastic.co/guide/en/elastic-stack-get-started/7.17/get-started-docker.html#get-started-docker-tls

Create a directory for hosting the compose file:

    mkdir /data/elk_7.14.4_docker
    chown 1000:1000 /data/elk_7.14.4_docker
    cd /data/elk_7.14.4_docker

Inside the directory create the files:
 - instances.yml
 - .env
 - create-certs.yml 
	 - => change line "- certs:/certs" by "- ./certs:/certs" 
	 - it's to have a local /data/elk_7.14.4_docker/certs where all certs will be availables
 - elastic-docker-tls.yml
	 - Change the 3 lines **ES_JAVA_OPTS=-Xms512m -Xmx512m** to adjust the memory given to each nodes

when everything is OK create certificats:

    docker-compose -f create-certs.yml run --rm create_certs

All certificats are avalables in /data/elk_7.14.4_docker/certs
Bring up the three-node Elasticsearch cluster:

    docker-compose -f elastic-docker-tls.yml up -d

Run the `elasticsearch-setup-passwords` tool to generate passwords for all built-in users, including the `kibana_system` user:

    docker exec es01 /bin/bash -c "bin/elasticsearch-setup-passwords auto --batch --url https://es01:9200"

Update `elastic-docker-tls.yml` with the `kibana_system`password generated.
Add the following line under kibana volumes section to export outside the container the kibana.yml file :

     - ./kibana.yml:/usr/share/kibana/config/kibana.yml

Create a file kibana.yml and add the following lines:

    server.host: "0.0.0.0"
    server.shutdownTimeout: "5s"
    elasticsearch.hosts: [ "https://elasticsearch:9200" ]
    monitoring.ui.container.elasticsearch.enabled: true
    xpack.monitoring.ui.container.elasticsearch.enabled: true
    xpack.encryptedSavedObjects.encryptionKey: Hp5jW2nMY4hn3Jhvzsvl9G9crEBFxA9b
    xpack.security.enabled: true

Copy the `elastic-docker-tls.yml` updated to a  new `docker-compose.yml` file.

Start the cluster:

    docker-compose up -d

After check status of the cluster:

    docker-compose ps

## ELK Stack 8.2.2

Stack 8.x => https://www.elastic.co/guide/en/elastic-stack-get-started/8.2/get-started-stack-docker.html#get-started-docker-tls

Create a directory for hosting the compose file:

    mkdir /data/elk_8.2.2_docker
    chown 1000:1000 /data/elk_8.2.2_docker
    cd /data/elk_8.2.2_docker

Inside the directory create the files:
 - .env
 - docker-compose.yml
	 - Change line "- certs:/usr/share/elasticsearch/config/certs" by "- ./
	 - certs:/usr/share/elasticsearch/config/certs"

Add the following line under kibana volumes section to export outside the container the kibana.yml file :

     - ./kibana.yml:/usr/share/kibana/config/kibana.yml

Create a file kibana.yml and add the following lines:

    server.host: "0.0.0.0"
    server.shutdownTimeout: "5s"
    elasticsearch.hosts: [ "https://elasticsearch:9200" ]
    monitoring.ui.container.elasticsearch.enabled: true
    xpack.monitoring.ui.container.elasticsearch.enabled: true
    xpack.encryptedSavedObjects.encryptionKey: Hp5jW2nMY4hn3Jhvzsvl9G9crEBFxA9b
    xpack.security.enabled: true
 
## Check and post configuration ELK Stack (7 or 8)

After starting of the cluster try a connection:

    curl -k -X GET "https://es01:9200/?pretty" -u elastic:[PASSWORD] --cacert /data/[stack_location]/certs/ca/ca.crt
    curl -k -X GET "https://es01:9200/_cat/nodes?pretty" -u elastic:[PASSWORD] --cacert /data/[stack_location]/certs/ca/ca.crt

Directory and all files must be attached to user / group 1000 / 1000 which are the one defined by elasticsearch.
Default volumes location **/var/lib/docker/volumes** .
To stop stack:

    cd /data/[stack_location]
    docker-compose down

After first launch connect with elastic superuser and create your own superuser.

To allow a local metricbeat to connect to the node or kibana update metricbeat.yml with the following section:

    # =================================== Kibana ===================================
    setup.kibana:
       host: "http://127.0.0.1:5601"
       protocol: https
       username: "USER"
       password: "PASSWORD"
       ssl.enabled: true
       ssl.verification_mode: none
       ssl.certificate_authorities: ["/data/[stack_location]/certs/ca/ca.crt"]
    #---------------------------- Elasticsearch output -----------------------------
    output.elasticsearch:
       hosts: ["es01:9200"]
       protocol: https
       username: "USER"
       password: "PASSWORD"
       ssl.certificate_authorities: ["/data/[stack_location]/certs/ca/ca.crt"]
       ssl.verification_mode: "none"
       proxy_disable: true
    
To allow a local logstash to connect to the node update logstash.yml with the following section:

    # ------------ X-Pack Settings (not applicable for OSS build)--------------
    xpack.management.enabled: true
    xpack.management.elasticsearch.hosts:
       - "https://es01:9200"
    xpack.management.logstash.poll_interval: 10s
    xpack.management.pipeline.id: ["beats"]
    xpack.management.elasticsearch.username: USER
    xpack.management.elasticsearch.password: PASSWORD
    xpack.management.elasticsearch.ssl.certificate_authority: /data/[stack_location]/certs/ca/ca.crt
    xpack.management.elasticsearch.ssl.verification_mode: "none"
    
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.username: USER
    xpack.monitoring.elasticsearch.password: PASSWORD
    xpack.monitoring.elasticsearch.hosts:
       - "https://es01:9200"
    xpack.monitoring.elasticsearch.ssl.certificate_authority: /data/[stack_location]/certs/ca/ca.crt
    xpack.monitoring.elasticsearch.ssl.verification_mode: "none"
    # -------------------------------------------------------------------------

On Kibana, create a basic "beats" logstash pipeline:

    input {
      beats {
        port => 5044
      }
    }
    filter {
    }    
    output {
      elasticsearch {
        index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
        hosts => ["https://localhost:9200"]
        user => "USER"
        password => "PASSWORD"
        cacert => "/data/[stack_location]/certs/ca/ca.crt"
        ssl => true
        ssl_certificate_verification => false
      }
    }

## Check and enable firewall rule for logstash

On the serveur check that the firewall allow incoming traffic on tcp/5044

    systemctl status firewalld
    ● firewalld.service - firewalld - dynamic firewall daemon
       Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
       Active: active (running) since Fri 2022-04-22 00:56:08 CEST; 3min 2s ago
Check of firewall:

    firewall-cmd --state 
    firewall-cmd --get-default-zone
    firewall-cmd --get-active-zones
    firewall-cmd --list-all
    firewall-cmd --get-services

Create a logstash-beat firewalld service:

    vi /etc/firewalld/services/logstash-beats.xml

Copy the following lines:

    <?xml version="1.0" encoding="utf-8"?>
    <service>
      <short>Logstash</short>
      <description>Logstash service is listening Elastic Beats before sending it to elasticsearch infra.</description>
      <port protocol="tcp" port="5044"/>
    </service>
Apply it permanently:

    firewall-cmd --zone=public --add-service=logstash-beats --permanent

Restart firewalld:

    systemctl restart firewalld

Try a telnet on port 5044 from a target node to check it's ok.

## For checking / testing

User htop (viusal top command) to track ressource usage
for CentOs 8:

    dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
    dnf update
    dnf install htop -y

For testing you can use stress-ng (for centos8):
    dnf install stress-ng -y

Test 80% of all available CPU during 5 minutes:

    stress-ng -c 0 -l 80 -t 5m

Test add 5Gb of used memory for 5 minutes:

    stress-ng --vm 5 --vm-bytes 1024M -t 5m

## Enable snapshot in stack

On the docker server add a local mount point i.e. `/snampshot`, check that the directory is owned by user uid 1000.

 Open the docker-compose.yml file and for each ELK node add the following lines:

 - Section "volumes":
    -- /snapshot:/usr/share/snapshot
 
 - Section "environment":
    -- path.repo="/usr/share/snapshot"

## Enable preconfigured index for kibana rules
Open the kibana.yml file and add the following line:

    xpack.actions.preconfiguredAlertHistoryEsIndex: true

Restart the stack.
    
**
> Written with [StackEdit](https://stackedit.io/).


> Written with [StackEdit](https://stackedit.io/).
