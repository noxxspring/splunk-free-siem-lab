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
- nano /opt/filebeat/filebeat.yml

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
## Create Filebeat systemd Service

- sudo systemctl daemon-reload
- sudo systemctl enable filebeat
- sudo systemctl start filebeat



# Attack Simulation and Log Monitoring

### Objective
Simulate an SSH brute-force attack from an attacker Docker container to a Dockerized SSH server, then detect abnormal patterns using Filebeat + Elastic + Kibana.

## Step 1 – Install Docker & Pull Images

```bash
sudo pacman -S docker
sudo systemctl start docker
sudo systemctl enable docker
```
## Step 2 – Run Attacker (Kali Linux in Docker)

```bash
docker run -it --rm kalilinux/kali-rolling /bin/bash

Explanation:
docker run – Runs a Docker container.
-it – Interactive terminal (you get a shell).
--rm – Remove the container when you exit (no leftover files).
kalilinux/kali-rolling – Official Kali Linux Docker image.
/bin/bash – Starts a Bash shell inside Kali.


If the Image has networking and repository issues then Update DNS and APT Sources

- Inside the kali docker run these commands

1. echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/resolv.conf

2. echo "deb http://kali.download/kali kali-rolling main non-free contrib" > /etc/apt/sources.list
```

## Step 3 – Install Hydra in Kali Container
 
 ```bash
 apt update && apt install hydra -y
```
## Step 4 – Run a Dockerized SSH Server (Victim)

 #### Create an empty file on the host. This ensures Docker will mount the log as a file
  ```bash
  sudo touch /var/log/docker-ssh-auth.log
  sudo chmod 666 /var/log/docker-ssh-auth.log
```

#### Run the container. This Maps the container logs to the Host Machine(Arch Linux)
``` bash
docker run -d -p 2224:22 \
 -v /var/log/docker-ssh-auth.log:/var/log/auth.log \
 rastasheep/ubuntu-sshd

 # Add rsyslog manually to log from the container to the Host machine
- docker exec -it <container_id> bash

- apt update && apt install -y rsyslog

- service rsyslog start
```

### Update filebeat config. - Use glob pattern that matches everything. it captures all logs in /var/log/ and subfolder
```nano
paths:
  - /var/log/**/*.log

#Restart Filebeat
- sudo systemctl restart filebeat
```

## Test
```bash
- ssh -p 2224 root@localhost
then input a wrong password 

# from another terminal run

- tail -f /var/log/docker-ssh-auth.log
it will show Failed password logs.
```

## To bruteforce from the kali docker machine
```bash
hydra -l root -P word.txt ssh://127.0.0.1:2224

note: by default the password is root, so the word.txt contains root which we will use to bruteforce the ssh and crack the password.once its cracked you will get the log in "- tail -f /var/log/docker-ssh-auth.log" running live on the terminal.
```

## View the log   in Kibana

```bash
Mounted /var/log/docker-ssh-auth.log from the Docker container to the host.
The filebeat is configured to harvest logs from /var/log/**/*.log which points filebeat to Elasticsearch
```

#### Go to : Kibana --> Discover --> Index --> filebeat-*
#### Search for: docker-ssh-auth.log, "Failed password", "Accepted password"









