---
layout: post
title: MySQL, Prometheus, Grafana Install (프로메테우스-그라파나 연동)
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
- 서버는 IDC에 위치한 **On-premise** Server로 OS에 직접 Agent를 설치해야 하는 상황
- Linux Server 는 AD 가입한 서버와 단일 서버(AD 미가입) 서버가 혼재되어 있는 상황
- 단일 서버 OS 계정은 Local 계정으로 System Engineer와 협의는 불 필요
- AD 가입 서버는 정책으로 Group이 새로 생겨 System Engineer와 협의 후 보조 그룹을 생성하여 계정에 추가
## Version 정보
- MySQL: 5.7.21
- OS: CentOS 7.9
- Kernel: 3.10.0
- OS Bit: 64Bit

------
> 테스트 용도로 Grafana 서버를 구성하며 실제 Grafana는 외부 부서에서 제공하는 서비스 사용   
## Grafana Install

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

Prometheus OS 계정 생성   
서비스 계정이므로 로그인 불가능 하도록 쉘 변경   

```bash
sudo useradd -m -r -s /sbin/nologin -d /opt/prometheus prometheus
```

`wget` 명령어를 통하여 Prometheus 파일 Download

```bash
wget "https://github.com/prometheus/prometheus/releases/download/v2.36.2/prometheus-2.36.2.linux-amd64.tar.gz"
```

현 설정은 따로 디렉토리를 분리하여 진행   
Server에 설치하는 외부 툴은 `/opt`에 저장

Version 정보는 문서로 별도 관리   
EOS로 Version 정보 변경 시 Daemon Configure 파일도 변경되어 Version 정보 삭제

```bash
mkdir -p /opt

tar -xvzf prometheus-2.36.2.linux-amd64.tar.gz -C /opt
mv /opt/prometheus-2.36.2.linux-amd64 /opt/prometheus
```

Proemtheus 서비스 계정을 분리하여 데몬 실행하여 해당 데몬을 시작할 수 있도록   
디렉토리 권한 및 소유자 변경
```bash
chown -R prometheus.prometheus /opt/prometheus
chmod -R 640 /opt/prometheus
```
systemctl로 데몬을 관리 할 수 있도록 아래 과정을 진행

```bash
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
> 오랜 시간이 지난 후 봤을 때, 문제가 생긴건지 데몬이 재시작한건지 인지하기 어려운 상황이 생김   
그에 따라 동적으로 설정 추가 및 변경이 가능하도록 파일 기반으로 수집할 서버 목록을 정리   

> 참고한 대시보드에서는 DB/OS 지표를 하나의 대시보드에서 모두 보고 있음, IP:Port 형식으로 데이터 수집 시   
OS/DB 따로 봐야 하는 상황이 생김 그에 따라 IP:Port(Server:Port) 에서 ":Port" 부분을 분리하여   
메타 데이터 저장   


해당 서버는 DNS가 등록되어 있지 않은 서버이며, Dashboard에서 IP로 볼 시 시인성이 떨어져 Local에 DNS 정보 등록
```bash
cat <<EOF | sudo tee -a /etc/hosts
192.168.137.101 MYSQLDB01
192.168.137.102 MYSQLDB02
192.168.137.103 MYSQLDB03
EOF
```
Config 파일을 구분할 수 있도록 과거 설정 파일들 디렉토리 분리

```bash
mkdir -p /opt/prometheus/sd/mysql
mkdir -p /opt/prometheus/config_old
mv /opt/prometheus/prometheus.yml /opt/prometheus/config_old/prometheus.yml.reference
```

---- 

### Prometheus 설정 파일 신규 등록   
아래 내용은 설정 파일에 대한 설명   
> - `scrape_interval`: metric 수집 주기 설정   
> - `evaluation_interval` : Prometheus Rule(Alert Rule, Recording Rule) Reload 주기 설정   
> - `file_sd_configs` : 파일 기반으로 수집 서버 설정   
> - `relabel_configs` : Meta 정보 등 수집되는 정보 변경을 위한 설정
>   - 참고한 대시보드에서 `:Port` 정보를 삭제하기 위하여 해당 설정 적용

```yaml
cat <<EOF | sudo tee -a /opt/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
 
scrape_configs:
  - job_name: "mysql"
    file_sd_configs:
    - files:
      - 'sd/mysql/*.json'
    relabel_configs:
      - source_labels: ['__address__']
        regex:         "(.*):.*"
        target_label:  "instance"
        replacement:   "\${1}"
EOF
```

수접 서버를 json파일 기반으로 작성   
지원하는 파일은 Json, Yaml, Csv 파일 지원   

```json
cat <<EOF | sudo tee -a /opt/prometheus/sd/mysql/static_config.json
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
```

```bash
# 새로 생성한 파일 권한 및 소유자 변경
chown -R prometheus.prometheus /opt/prometheus
chmod -R 640 /opt/prometheus
```

----    

## Install Node Exporter

----  
Agent 설치 하기 위하여 OS에 데몬 실행 계정 생성(서비스 계정)    

```bash
mkdir -p /opt
sudo useradd -m -r -s /sbin/nologin -d /opt/prometheus prometheus
```

필요에 따라 생략하며, 아래는 생략 예시
  - mysqld exporter 설치하면서 미리 진행
  - Server Owner 부서 정책

  ---- 

`wget` 명령어를 통한 node_exporter 압축 파일 Download

```bash
wget "https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz"
```

현 설정은 따로 디렉토리를 분리하여 진행   
Server에 설치하는 외부 툴은 `/opt`에 저장   
Prometheus 관련 Tool은 `/opt/prometheus` 로 저장

Version 정보는 문서로 별도 관리   
EOS로 Version 정보 변경 시 Daemon Configure 파일도 변경되어 Version 정보   

```bash
tar -xvzf node_exporter-1.3.1.linux-amd64.tar.gz -C /opt/prometheus
mv /opt/prometheus/node_exporter-1.3.1.linux-amd64/ /opt/prometheus/node_exporter
chown -R prometheus.prometheus /opt/prometheus
```

root 권한을 가진 사용자 외 접근이 불가능 하도록 디렉토리 권한 설정 변경
- prometheus 계정은 로그인 불가 쉘로 인하여 접근 불가

```bash
chmod -R 700 /opt/prometheus
```

> systemctl로 데몬을 관리 할 수 있도록 아래 과정을 진행

```bash
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
----
## Install mysqld Exporter

node_exporter와 겹치는 설정은 설명을 생략합니다.

```bash
wget "https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.linux-amd64.tar.gz"
mkdir -p /opt
tar -xvzf mysqld_exporter-0.14.0.linux-amd64.tar.gz -C /opt/prometheus
mv /opt/prometheus/mysqld_exporter-0.14.0.linux-amd64/ /opt/prometheus/mysqld_exporter
```

MySQL DB 정보는 DB에 직접 접근하여 System Table 정보를 수집한다.   
그에 따라 DB 계정 정보가 필요하며 해당 정보는 파일로 설정한다.   

```bash
touch /opt/prometheus/mysqld_exporter/my.cnf
```

아래 설정 파일은 console에 노출되지 않도록 가급적 vi/nano 등의 편집기로 수정/등록한다.

```bash
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

---    

mysqld exporter 는 DB 계정이 필요하며 필요한 권한은 아래와 같다.   
mysql 접속 정보는 mysql_config_editor를 통하여 미리 저장해놓았으며   
해당 접속 정보를 기반으로 console에 접속한다.
   - Remote 환경일 경우 Remote로 접근하여 계정을 생성 및 권한 설정을 한다.

```sql
# mysql --login-path=localhost

CREATE USER `prometh_service`@`localhost` idneitifed by "";
GRANT SELECT, PROCESS, REPLICATION CLIENT ON *.* TO `prometh_service`@`localhost`;
```
> DB 계정이 생성이 완료되었으면 systemctl로 데몬을 관리 할 수 있도록 아래 과정을 진행   
> 옵션의 경우 mysqld_exporter github에서 확인할 수 있다.     
> 아래 설정은 일반적인(Reference Default) 설정이다.   
> 상황에 따라 변경하에 적용하자.   


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
----   

데몬 설정파일을 systemctl 에서 읽을 수 있도록 reload
```bash
systemctl daemon-reload
```

서비스 시작 및 재부팅시 자동 시작

```bash
systemctl start mysqld_exporter
systemctl enable mysqld_exporter

systemctl start node_exporter
systemctl enable node_exporter
```
설정한 default port 로 listen 인지 확인

```bash
netstat -nalp | grep 910
tcp6       0      0 :::9100                 :::*                    LISTEN      13007/node_exporter
tcp6       0      0 :::9104                 :::*                    LISTEN      13114/mysqld_export
```

- 만약 port가 listen 하지 않을 시
- `systemctl status mysqld_exporter`, `systemctl status node_exporter` 의 명렁어를 통한 확인
- `journalctl`를 통해서도 확인 가능