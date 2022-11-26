---
layout: default
title: MySQL, Prometheus, Grafana Install
parent: Grafana
grand_parent: Monitoring
---
<details open markdown="block">
  <summary>
    목차
  </summary>
  {: .text-delta }
{:toc}
</details>

# MySQL, Prometheus, Grafana Install

## 개요
MySQL OS 지표와 DB 지표를 node_exporter, mysqld_exporter를 이용하여 수집하며, Prometheus - Grafana를 사용하여 모니터링 하기 위하여 스크립트 기반으로 문서 작성

- MySQL: 5.7.21
- OS: CentOS 7.9
- Kernel: 3.10.0
- OS Bit: 64Bit

## Grafana Install
Install Grafana
```bash 
cat <<EOF | sudo tee -a /etc/yum.repo.d/grafna.repo
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
sudo yum check-update
sudo yum install grafana

sudo systemctl daemon-reload

# default:
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Change Admin password
sudo grafana-cli admin reset-admin-password "Password"
```

## Install Prometheus
```bash
# prometheus archive file download
wget "https://github.com/prometheus/prometheus/releases/download/v2.36.2/prometheus-2.36.2.linux-amd64.tar.gz"
mkdir -p /opt
tar -xvzf prometheus-2.36.2.linux-amd64.tar.gz -C /opt
mv /opt/prometheus-2.36.2.linux-amd64 /opt/prometheus

sudo groupadd sudouser
#visudo -f /etc/sudoers.d/sudouser

cat <<EOF | sudo tee -a /etc/sudoers.d/sudouser
%sudouser  ALL=(ALL)       NOPASSWD: ALL
EOF

sudo useradd -m -r -s /sbin/nologin -G sudouser -d /opt/prometheus prometheus

cat <<EOF | sudo tee -a /usr/lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network.target

[Service]
User=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF
```

## Install Node Exporter

```bash
sudo groupadd sudouser

cat << EOF | sudo tee -a /etc/sudoers.d/sudouser
%sudouser  ALL=(ALL)       NOPASSWD: ALL
EOF
mkdir -p /opt
sudo useradd -m -r -s /sbin/nologin -G sudouser -d /opt/prometheus prometheus
# wget "https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz"
tar -xvzf node_exporter-1.3.1.linux-amd64.tar.gz -C /opt/prometheus
mv /opt/prometheus/node_exporter-1.3.1.linux-amd64/ /opt/prometheus/node_exporter
chown -R prometheus.prometheus /opt/prometheus/node_exporter
chmod -R 700 /opt/prometheus/node_exporter


cat << EOF | sudo tee -a /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
Wants=network.target
After=network.target

[Service]
User=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/node_exporter/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```
## Install MySQLd Exporter
```bash
wget "https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.linux-amd64.tar.gz"
mkdir -p /opt
tar -xvzf mysqld_exporter-0.14.0.linux-amd64.tar.gz -C /opt/prometheus
mv /opt/prometheus/mysqld_exporter-0.14.0.linux-amd64/ /opt/prometheus/mysqld_exporter

touch /opt/prometheus/mysqld_exporter/my.cnf

cat << EOF | sudo tee -a /opt/prometheus/mysqld_exporter/my.cnf
[client]
port=3306
socket= /tmp/mysql.sock
default-character-set=utf8
user=export_service
password=Grafana@@0707
EOF

chown -R prometheus.prometheus /opt/prometheus/mysqld_exporter
chmod -R 700 /opt/prometheus/mysqld_exporter
chmod 600 /opt/prometheus/mysqld_exporter/my.cnf

```
```sql
# mysql --login-path=localhost

CREATE USER `prometh_service`@`localhost` idneitifed by "";
GRANT SELECT, PROCESS, REPLICATION CLIENT ON *.* TO `prometh_service`@`localhost`;
```

```bash
cat << EOF | sudo tee -a /usr/lib/systemd/system/mysqld_exporter.service
[Unit]
Description=Prometheus MySQL Exporter
After=network.target

[Service]
Type=simple
Restart=always
User=prometheus
Group=prometheus
ExecStart=/opt/prometheus/mysqld_exporter/mysqld_exporter \
--config.my-cnf=/opt/prometheus/mysqld_exporter/my.cnf \
--web.listen-address=0.0.0.0:9104 \
--collect.global_status \
--collect.global_variables \
--collect.info_schema.clientstats \
--collect.info_schema.innodb_metrics \
--collect.info_schema.innodb_tablespaces \
--collect.info_schema.innodb_cmp \
--collect.info_schema.innodb_cmpmem \
--collect.info_schema.processlist \
--collect.info_schema.processlist.min_time=0 \
--collect.info_schema.query_response_time \
--collect.info_schema.replica_host \
--collect.info_schema.tables \
--collect.info_schema.tables.databases=‘*’ \
--collect.info_schema.tablestats \
--collect.info_schema.schemastats \
--collect.info_schema.userstats \
--collect.mysql.user \
--collect.perf_schema.eventsstatements \
--collect.perf_schema.eventsstatements.digest_text_limit=120 \
--collect.perf_schema.eventsstatements.limit=250 \
--collect.perf_schema.eventsstatements.timelimit=86400 \
--collect.perf_schema.eventsstatementssum \
--collect.perf_schema.eventswaits \
--collect.perf_schema.file_events \
--collect.perf_schema.file_instances \
--collect.perf_schema.file_instances.remove_prefix=false \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.memory_events \
--collect.perf_schema.memory_events.remove_prefix=false \
--collect.perf_schema.tableiowaits \
--collect.perf_schema.tablelocks \
--collect.perf_schema.replication_group_members \
--collect.perf_schema.replication_group_member_stats \
--collect.perf_schema.replication_applier_status_by_worker \
--collect.slave_status \
--collect.slave_hosts \
--collect.heartbeat \
--collect.heartbeat.database=true \
--collect.heartbeat.table=true \
--collect.heartbeat.utc

[Install]
WantedBy=multi-user.target
EOF
```

## Daemon Run
```bash
systemctl daemon-reload

systemctl start mysqld_exporter
systemctl enable mysqld_exporter

systemctl start node_exporter
systemctl enable node_exporter

netstat -nalp | grep 910
tcp6       0      0 :::9100                 :::*                    LISTEN      13007/node_exporter
tcp6       0      0 :::9104                 :::*                    LISTEN      13114/mysqld_export
```