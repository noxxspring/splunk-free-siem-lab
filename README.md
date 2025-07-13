 # Splunk-Free SIEM Lab (Arch Linux)

 This project builds a full SIEM (Security Information and Event Management) systemt from scratch using open-source tools on Arch linux

## Components Installed

- **Elasticsearch** - log index and search engine
- **Kibana** - frontend dashboard and visualization
- **Filebeat** - lightweight log shipper

##  Elasticsearch Setup

### Config File: `/etc/elasticsearch/elasticsearch.yml`

```yaml
network.host: 127.0.0.1
http.port: 9200
bootstrap.memory_lock: true
action.destructive_requires_name: true
```
### Memory Lock systemd Override 
- file /etc/systemd/system/elasticsearch.service.d/override.conf
[Service]
LimitMEMLOCK=infinity

### Enable + Start ElasticSearch
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

### Test the Elasticsearch
curl http://localhost:9200


---

# Kibana Setup(v7.10.2)

### Download and Extract kibana


```bash
cd /opt
sudo curl -L -O https://artifacts.elastic.co/downloads/kibana/kibana-7.10.2-linux-x86_64.tar.gz
sudo tar -xzf kibana-7.10.2-linux-x86_64.tar.gz
sudo mv kibana-7.10.2-linux-x86_64 kibana
sudo chown -R $USER:$USER /opt/kibana
```
### Configure kibana
- sudo nano /opt/kibana/config/kibana.yml

#### then add these
```yaml
server.host: "127.0.0.1"
server.port: 5601
elasticsearch.hosts: ["http://localhost:9200"]
```

Then create a systemd service for the kibanato keep it actively runnng

#### Open Kibana Web UI
- http://localhost:5061



# Filebeat Setup (v7.10.2- Manual Install)

## Install

```bash
cd /opt
sudo curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.10.2-linux-x86_64.tar.gz
sudo tar -xzf filebeat-7.10.2-linux-x86_64.tar.gz
sudo mv filebeat-7.10.2-linux-x86_64 filebeat
sudo chown -R $USER:$USER /opt/filebeat
```
### configure
```yaml
filebeat.inputs:
- type: filestream
  id: system-logs
  enabled: true
  paths:
    - /var/log/*.log

output.elasticsearch:
  hosts: ["http://localhost:9200"]

setup.kibana:
  host: "localhost:5601"
```
# Create Filebeat systemd Service

- sudo systemctl daemon-reload
- sudo systemctl enable filebeat
- sudo systemctl start filebeat
