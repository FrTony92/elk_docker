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

Because we are going to create a 3 nodes cluster, duplicate and adjust es01_elasticsearch.yml for the 2 other nodes (es02 and es03).

Update .env to your needs for cluster name and or password provided for kibana and elastic superuser.
On the local server update the /etc/hosts in order to map local IP of the server with es01,es02 and es03.
According to the available memory on the host update parameter "-XmsXX" and "-XmxXX" which will be used by each docker node.
For exemple setting "-Xms1g" and "-Xmx1g" will allow to have 1Gb by node and, when started, 3 Gb for the cluster.

when everything is OK start the cluster:

    chown -R 1000:1000 /data/elk_compose
    cd /data/elk_compose
    docker-compose up -d

Directory and all files must be attached to user / group 1000 / 1000 which are the one defined by elasticsearch.
Default volumes location **/var/lib/docker/volumes** will be modified and store in /data/elk_compose.
To stop stack:

    cd /data/elk_compose
    docker-compose down

After first launch connect with elastic superuser and create your own superuser.

To allow a local metricbeat to connect to the node or kibana update metricbeat.yml with the following section:

    # =================================== Kibana ===================================
    setup.kibana:
       host: "http://127.0.0.1:5601"
       protocol: https
       username: "antoine"
       password: "antoine"
       ssl.enabled: true
       ssl.verification_mode: none
       ssl.certificate_authorities: ["/data/elk_compose/certs/ca/ca.crt"]
    #---------------------------- Elasticsearch output -----------------------------
    output.elasticsearch:
       hosts: ["es01:9200"]
       protocol: https
       username: "antoine"
       password: "antoine"
       ssl.certificate_authorities: ["/data/elk_compose/certs/ca/ca.crt"]
       ssl.verification_mode: "none"
    
To allow a local logstash to connect to the node update logstash.yml with the following section:

    # ------------ X-Pack Settings (not applicable for OSS build)--------------
    xpack.management.enabled: true
    xpack.management.elasticsearch.hosts:
       - "https://es01:9200"
    xpack.management.logstash.poll_interval: 10s
    xpack.management.pipeline.id: ["beats"]
    xpack.management.elasticsearch.username: antoine
    xpack.management.elasticsearch.password: antoine
    xpack.management.elasticsearch.ssl.certificate_authority: /etc/logstash/ca.crt
    xpack.management.elasticsearch.ssl.verification_mode: "none"
    
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.username: antoine
    xpack.monitoring.elasticsearch.password: antoine
    xpack.monitoring.elasticsearch.hosts:
       - "https://es01:9200"
    xpack.monitoring.elasticsearch.ssl.certificate_authority: /etc/logstash/ca.crt
    xpack.monitoring.elasticsearch.ssl.verification_mode: "none"
    # -------------------------------------------------------------------------

And copy the ca.crt file in the logstash repository:

    cp /data/elk_compose/certs/ca/ca.crt /etc/logstash/ca.crt
    chown logstash:logstash /etc/logstash/ca.crt

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
        user => "antoine"
        password => "antoine"
        cacert => "/etc/logstash/ca.crt"
        ssl => true
        ssl_certificate_verification => false
      }
    }

On the serveur check the firewall to allow incoming traffic on tcp/5044

    systemctl status firewalld
    ● firewalld.service - firewalld - dynamic firewall daemon
       Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
       Active: active (running) since Fri 2022-04-22 00:56:08 CEST; 3min 2s ago

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

**
> Written with [StackEdit](https://stackedit.io/).
