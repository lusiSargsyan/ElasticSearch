# ElasticSearch

Elasticsearch is a real-time, distributed storage, search, and analytics engine. It can be used for many purposes, but one context where it excels is indexing streams of semi-structured data, such as logs or decoded network packets.

ELK stack setup has four main components:

* Logstash: The server component of Logstash that processes incoming logs
* Elasticsearch: Stores all of the logs
* Kibana: Web interface for searching and visualizing logs, which will be proxied through Nginx
* Filebeat: Installed on client servers that will send their logs to Logstash, Filebeat serves as a log shipping agent that utilizes the lumberjack networking protocol to communicate with Logstash

# Required apps

* Java 8
* ElasticSearch
* Kibana
* Logstash


To setup ElasticSearch 
          

     curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.1-linux-x86_64.tar.gz
     tar -xzvf elasticsearch-7.9.1-linux-x86_64.tar.gz
     cd elasticsearch-7.9.1
     ./bin/elasticsearch


It is recommended to restrict the access to port 9200 from outside so outsiders can’t read your data or shutdown your Elasticsearch cluster through the HTTP API.
Find the line that specifies network.host, uncomment it, and replace its value with “localhost” so it looks like this:
              

       network.host: localhost


     

#####Install Kibana
          

     echo "deb http://packages.elastic.co/kibana/4.5/debian stable main" | sudo tee -a /etc/apt/sources.list.d/kibana-4.5.x.list
     sudo apt-get update
     sudo apt-get -y install kibana

Or setup manually
         

  
     wget https://artifacts.elastic.co/downloads/kibana/kibana-7.9.1-amd64.deb
     shasum -a 512 kibana-7.9.1-amd64.deb 
     sudo dpkg -i kibana-7.9.1-amd64.deb
     #and start with systemd
     sudo /bin/systemctl daemon-reload
     sudo /bin/systemctl enable kibana.service
     sudo systemctl start kibana.service

     

#####Install LogStash
     

          

     echo 'deb http://packages.elastic.co/logstash/2.2/debian stable main' | sudo tee /etc/apt/sources.list.d/logstash-2.2.x.list
     sudo apt-get update
     sudo apt-get install logstash
######Configure LogStash
Logstash configuration files are in the JSON-format, and reside in /etc/logstash/conf.d. The configuration consists of three sections: 
* inputs
* filters
* outputs 
  ###### Let’s create a configuration file called 02-beats-input.conf and set up our “filebeat” input
          

     input {
         beats {
         port => 5044
         ssl => true
         ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
         ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
       }
     }

As a next step we will create a filter file configuration.

     sudo vi /etc/logstash/conf.d/10-syslog-filter.conf

     
This configuration is for files woth type "syslog" files.This type will be given in Filebeat side in client server.
Also this is going to use grok to parse that files and make them structured.

     filter {
       if [type] == "syslog" {
        grok {
          match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
          add_field => [ "received_at", "%{@timestamp}" ]
          add_field => [ "received_from", "%{host}" ]
        }
        syslog_pri { }
        date {
            match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
       }
     }

 And finally we need to create an output file.

     sudo vi /etc/logstash/conf.d/30-elasticsearch-output.conf  
     output {
        elasticsearch {
          hosts => ["localhost:9200"]
          sniffing => true
          manage_template => false
          index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
          document_type => "%{[@metadata][type]}"
        }
     }

This output basically configures Logstash to store the beats data in Elasticsearch which is running at localhost:9200, in an index named after the filebeat used.
To check our Logstash configuration we need to run

     sudo service logstash configtest
In case of "OK" message we can restart and enable Logstash to enable our configurations.
Otherwise please fix errors based on given error message.

     sudo service logstash restart
     sudo update-rc.d logstash defaults 96 9
     
     
 ######Load Filebeat Index Template in Elasticsearch
 As a next step we need to give our index templates to ElasticSearch.
 At firs we need to download index template and then push it to ElasticSearch
 
      curl -O https://gist.githubusercontent.com/thisismitch/3429023e8438cc25b86c/raw/d8c479e2a1adcea8b1fe86570e42abab0f10f364/filebeat-index-template.json
      curl -XPUT 'http://localhost:9200/_template/filebeat?pretty' -d@filebeat-index-template.json
      
##### Set Up FileBeat in client servers
Copy created certificates from ELK server to client server under /etc/pki/tls/certs folder.You need to cretae the folder at first.
Install FileBeat

     echo "deb https://packages.elastic.co/beats/apt stable main" |  sudo tee -a /etc/apt/sources.list.d/beats.list
     sudo apt-get update
     sudo apt-get install filebeat
     
######Configure FileBeat
     sudo vi /etc/filebeat/filebeat.yml
Here under paths: section we can see which log files it is going to send to LogStash.
By default it is 
 
     paths:
    - /var/log/*.log
We can change this line, add new ones etc.
We can change type (or document-type) to syslog or other depends on requirements .
And we need to change logstash host if it is not in our localhost

            
     logstash:
    # The Logstash hosts
    hosts: ["ELK_server_private_IP:5044"]
 Pay attention also to "bulk_max_size" value and be sure that 
 tls part is modified as well for secure connection. 
 
 As a final step please restart the FileBeat

            
     sudo service filebeat restart
     sudo update-rc.d filebeat defaults 95 10