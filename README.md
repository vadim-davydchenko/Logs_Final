#### 1.Setup an Elasticsearch cluster of three nodes
`wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.6.1-amd64.deb`

`sudo dpkg -i elasticsearch-8.6.1-amd64.deb`

Change the parameters in `/config/elasticsearch.yml`

```
cluster.name: es-cluster
node.name: node1
node.roles: [master, data]
network.host: 10.110.0.4
discovery.seed_hosts: ["10.110.0.6", "10.110.0.2", "10.110.0.5"]
cluster.initial_master_nodes: ["node1", "node2", "node3"]
#http.host

xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl:
  enabled: false
xpack.security.transport.ssl:
  enabled: false
ingest.geoip.downloader.enabled: false
```

`sudo swapoff -a`

`sudo sysctl -w vm.max_map_count=262144`

`sudo systemctl start elasticsearch`

**Checking and verifying who is the master**

`curl -XGET 'http://10.110.0.6:9200/_cluster/health?pretty'`

`curl -sS -XGET 'http://10.110.0.6:9200/_cat/master?pretty'`

****Create an index named syslogs to save logs. Set 3 shards for it and 2 replicas for each primary shard**

`curl -XPUT http://10.110.0.2:9200/syslogs -H "Content-Type: application/json" -d '{"settings" : { "index" : {"number_of_shards" : 3, "number_of_replicas" : 2 } } }'`

Check shards

`curl -XGET http://10.110.0.2:9200/_cat/shards/syslogs`

#### 2.Configure on the server:
- kibana - in such a way that even if there is no connection with one or two Elasticsearch nodes, the data for search and display is available.
- apache.kafka - for temporary storage of log messages, before they are processed and sent to the elasticsearch cluster.
- fluentd - subtract messages from kafka, roll them out and send them to the appropriate index in elasticsearch.

`wget https://artifacts.elastic.co/downloads/kibana/kibana-8.6.1-amd64.deb`

`sudo dpkg -i kibana-8.6.1-amd64.deb`

Setup `/etc/kibana/kibana.yml`

```
server.port: 5601
server.host: "161.35.154.61"
server.name: "kibana"
elasticsearch.hosts: ["http://10.110.0.6:9200", "http://10.110.0.2:9200", "http:10.110.0.5:9200"]
```

Install Kafka

`apt update && apt install default-jre -y`

`wget https://downloads.apache.org/kafka/3.4.0/kafka_2.12-3.4.0.tgz`

```
tar -xzf kafka_2.12-3.4.0.tgz
mv kafka_2.12-3.4.0 kafka
```

Setup Zookeeper /etc/systemd/system/zookeeper.service

```
[Unit]
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=root
ExecStart=/home/user/kafka/bin/zookeeper-server-start.sh /home/user/kafka/config/zookeeper.properties
ExecStop=/home/user/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

Setup Kafka /etc/systemd/system/kafka.service

```
[Unit]
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=root
ExecStart=/bin/sh -c '/home/user/kafka/bin/kafka-server-start.sh /home/user/kafka/config/server.properties > /home/user/kafka/kafka.log 2>&1'
ExecStop=/home/user/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

Install and setup fluentd

`curl -fsSL https://toolbelt.treasuredata.com/sh/install-ubuntu-focal-td-agent4.sh | sh`

`sudo systemctl start td-agent.service`

Setting up the subtraction of logs from Kafka and sending them to index Elasticsearch

```
<source>
  @type kafka
  brokers 10.110.0.4:9092
  topics log_topic
  format json
  message_key log
</source>

<match **>
  @type elasticsearch
  hosts 10.110.0.6:9200,10.110.0.2:9200,10.110.0.5:9200
  index_name syslogs
  type_name syslog
  logstash_prefix syslogs
  logstash_format true
  include_tag_key true
  tag_key @log_name
  <buffer tag,time>
    @type memory
    flush_at_shutdown true
    flush_mode interval
    flush_interval 30s
    timekey 1d
  </buffer>
</match>
```

#### 3.Setting up fluentd on servers**
- subtraction of all current and historical system logs.
- Subtraction of all current and historical application logs.
- sending all the read messages to kafka.

`sudo td-agent-gem install fluent-plugin-kafka`

```
<source>
  @type tail
  path /var/log/syslog,/var/log/*.log, /var/log/nginx/*.log
  pos_file /var/log/td-agent/syslog.pos
  tag syslog
  format syslog
</source>

<match **>
  @type kafka2
  brokers 10.110.0.4:9092
  default_topic log_topic
  output_data_type json
  output_include_tag true
  output_include_time true
  <format>
    @type json
  </format>
  <buffer>
    @type memory
    flush_thread_count 4
    flush_interval 5s
    chunk_limit_size 5M
    queue_limit_length 128
    retry_max_interval 30
    overflow_action block
  </buffer>
</match>
```
