---
layout: default
title: SQL Server(MSSQL), Prometheus, Grafana Install
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

# MSSQL 서버의 DB 지표를 Prometheus-Grafana를 활용하여 모니터링

## 개요
MSSQL OS 지표와 DB 지표를 node_exporter, mysqld_exporter를 이용하여 수집하며,   
Prometheus - Grafana를 사용하여 모니터링 하기 위하여 스크립트 기반으로 문서 작성   
* [MSSQL 대시보드](https://grafana.com/grafana/dashboards/7362-mysql-overview/)
* [node_exporter Github](https://github.com/prometheus/node_exporter)
* [mysqld_exporter Github](https://github.com/prometheus/mysqld_exporter)
## 상황
- 서버는 IDC에 위치한 **On-premisse** Server로 OS에 직접 Agent를 설치해야 하는 상황

## Version 정보
- SQL Server(MSSQL): SQL Server 2017
- OS: Windows 2016
- OS Bit: 64Bit

## sql_exporter 설치

----
### Prometheus Configure
```bash
cat <<EOF | sudo tee -a /etc/hosts
192.168.137.201 MSSQLDB01
192.168.137.202 MSSQLDB02
192.168.137.203 MSSQLDB03
EOF

mkdir -p /opt/prometheus/sd/mysql
mkdir -p /opt/prometheus/sd/mssql
mkdir -p /opt/prometheus/config_old
mv /opt/prometheus/prometheus.yml /opt/prometheus/config_old

cat <<EOF | sudo tee -a /opt/prometheus/prometheus.yml
  - job_name: "mssql"
    file_sd_configs:
    - files:
      - 'sd/mssql/*.json'
    relabel_configs:
      - source_labels: ['__address__']
        regex:         "(.*):.*"
        target_label:  "instance"
        replacement:   "${1}"
      - source_labels: ['__address__']
        regex:         "(.*):.*"
        target_label:  "job"
        replacement:   "${1}"
EOF

cat <<EOF | sudo tee -a /opt/prometheus/sd/mssql/static_config.yml
[
    {
        "targets": [
            "MSSQLDB01:9389",
            "MSSQLDB02:9389",
            "MSSQLDB03:9389"
        ],
        "labels": {
            "job": "mssql"
        }
    }
]
EOF
```

