# Log Management Role

## Purpose

This Ansible role automates the setup of a log management system, including:

1. **Elasticsearch Cluster:** Configures a 3-node Elasticsearch cluster for distributed log storage.
2. **Kibana:** Sets up a visualization interface for Elasticsearch data.
3. **Kafka:** Configures Kafka for temporary log storage.
4. **Fluentd:**
   - Reads logs from Kafka and sends them to Elasticsearch.
   - Reads system and application logs and sends them to Kafka.

---

## How to Run

1. **Prepare Inventory File:**
   - Define your hosts in an inventory file, specifying their roles (`elasticsearch`, `kibana`, `kafka`).

2. **Execute Playbook:**
   - Run the following command to deploy the log management system:

```bash
ansible-playbook -i inventory.yml logs.yml
```

---

## Role Functionality

### 1. Elasticsearch Cluster Setup
- Installs Elasticsearch on three nodes.
- Configures the cluster name, node roles, network settings, and disables x-pack security.
- Creates an index `syslogs` with:
  - 3 shards.
  - 2 replicas per shard.

### 2. Kibana Setup
- Installs Kibana.
- Configures Kibana to connect to the Elasticsearch cluster.
- Ensures data remains accessible even if one or two Elasticsearch nodes are unavailable.

### 3. Kafka Setup
- Installs and configures Zookeeper and Kafka.
- Manages Zookeeper and Kafka services using systemd.

### 4. Fluentd Setup
#### a. Kafka to Elasticsearch
- Reads logs from Kafka topics.
- Sends processed logs to the Elasticsearch index `syslogs`.

#### b. System and Application Logs to Kafka
- Reads logs from system files and application logs (e.g., `/var/log/syslog`, `/var/log/nginx/*.log`).
- Sends these logs to Kafka in JSON format.
