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

# MySQL 서버의 OS/DB 지표를 Prometheus-Grafana를 활용하여 모니터링

## 개요
MySQL OS 지표와 DB 지표를 node_exporter, mysqld_exporter를 이용하여 수집하며,   
Prometheus - Grafana를 사용하여 모니터링 하기 위하여 스크립트 기반으로 문서 작성   
* [MySQL 대시보드](https://grafana.com/grafana/dashboards/7362-mysql-overview/)
* [node_exporter Offical Guides](https://prometheus.io/docs/guides/node-exporter/)
* [node_exporter Github](https://github.com/prometheus/node_exporter)
* [mysqld_exporter Github](https://github.com/prometheus/mysqld_exporter)
## 상황
- 서버는 IDC에 위치한 **On-premisse** Server로 OS에 직접 Agent를 설치해야 하는 상황
- Linux Server 는 AD 가입한 서버와 단일 서버(AD 미가입) 서버가 혼재되어 있는 상황
- 단일 서버 OS 계정은 Local 계정으로 챙길 상황은 생기지 않음
- AD 가입 서버는 정책으로 Group이 새로 생겨 System Engineer와 협의 후 보조 그룹을 생성하여 계정에 추가
## Version 정보
- MySQL: 5.7.21
- OS: CentOS 7.9
- Kernel: 3.10.0
- OS Bit: 64Bit

------
테스트 용도로 Grafana 서버를 구성하며 실제 Grafana는 외부 부서에서 제공하는 서비스 사용
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
----   
## Install Prometheus
```bash
# Prometheus OS User Create
# 서비스 계정이므로 로그인 불가능 하도록 쉘 변경
sudo useradd -m -r -s /sbin/nologin -d /opt/prometheus prometheus

# prometheus archive file download
wget "https://github.com/prometheus/prometheus/releases/download/v2.36.2/prometheus-2.36.2.linux-amd64.tar.gz"
mkdir -p /opt
tar -xvzf prometheus-2.36.2.linux-amd64.tar.gz -C /opt
mv /opt/prometheus-2.36.2.linux-amd64 /opt/prometheus

chown -R prometheus.prometheus /opt/prometheus
chmod -R 640 /opt/prometheus

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
----
### Prometheus Configure
> 모니터링할 서버가 추가 될 때 마다 Prometheus Daemon을 재기동 해야 하는 상황이 생김.   
오랜 시간이 지난 후 봤을 때, 문제가 생긴건지 데몬이 재시작한건지 인지하기 어려운 상황이 생김   
그에 따라 동적으로 설정 추가 및 변경이 가능하도록 파일 기반으로 수집할 서버 목록을 정리   
참고한 대시보드에서는 DB/OS 지표를 하나의 대시보드에서 모두 보고 있음, IP:Port 형식으로 데이터 수집 시   
OS/DB 따로 봐야 하는 상황이 생김 그에 따라 IP:Port(Server:Port) 에서 ":Port" 부분을 분리하여 메타 데이터 저장
```bash
cat <<EOF | sudo tee -a /etc/hosts
192.168.137.101 MYSQLDB01
192.168.137.102 MYSQLDB02
192.168.137.103 MYSQLDB03
EOF

mkdir -p /opt/prometheus/sd/mysql
mkdir -p /opt/prometheus/sd/mssql
mkdir -p /opt/prometheus/config_old
mv /opt/prometheus/prometheus.yml /opt/prometheus/config_old

cat <<EOF | sudo tee -a /opt/prometheus/prometheus.yml
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
 
scrape_configs:
  - job_name: "mysql"
    file_sd_configs:
    - files:
      - 'sd/mysql/*.json'
    relabel_configs:
      - source_labels: ['__address__']
        regex:         "(.*):.*"
        target_label:  "instance"
        replacement:   "${1}"
EOF

cat <<EOF | sudo tee -a /opt/prometheus/sd/mysql/static_config.yml
[
    {
        "targets": [
            "MYSQLDB01:9104",
            "MYSQLDB02:9104",
            "MYSQLDB03:9104"
        ],
        "labels": {
            "job": "mysql"
        }
    },
    {
        "targets": [
            "MYSQLDB01:9100",
            "MYSQLDB02:9100",
            "MYSQLDB03:9100"
        ],
        "labels": {
            "job": "mysql"
        }
    }
]
EOF

chown -R prometheus.prometheus /opt/prometheus
chmod -R 640 /opt/prometheus
```

## Install Node Exporter

```bash
mkdir -p /opt
sudo useradd -m -r -s /sbin/nologin -G sudouser -d /opt/prometheus prometheus
wget "https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz"
tar -xvzf node_exporter-1.3.1.linux-amd64.tar.gz -C /opt/prometheus
mv /opt/prometheus/node_exporter-1.3.1.linux-amd64/ /opt/prometheus/node_exporter
chown -R prometheus.prometheus /opt/prometheus
chmod -R 700 /opt/prometheus


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
## Install mysqld Exporter
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
chown -R prometheus.prometheus /opt/prometheus
chmod -R 700 /opt/prometheus
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