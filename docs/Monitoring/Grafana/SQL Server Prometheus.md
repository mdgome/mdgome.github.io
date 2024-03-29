---
layout: post
title: SQL Server(MSSQL), Prometheus, Grafana Install (프로메테우스-그라파나 연동)
parent: Grafana
grand_parent: Monitoring
---
<details open markdown="block">
  <summary>
    목차
  </summary>
{: .no_toc .text-delta }

1. TOC
{:toc}
</details>

# MSSQL 서버의 DB 지표를 Prometheus-Grafana를 활용하여 모니터링

## 개요
MSSQL OS 지표와 DB 지표를 sql_exporter를 이용하여 수집하며,   
Prometheus - Grafana를 사용하여 모니터링 하기 위하여 스크립트 기반으로 문서 작성   
* [MSSQL 대시보드](https://promcat.io/apps/mssql)
* [sql_exporter Github](https://github.com/free/sql_exporter)
* [Agent Download](https://github.com/free/sql_exporter/releases/tag/0.5)
## 상황
- 서버는 IDC에 위치한 **On-premisse** Server로 OS에 직접 Agent를 설치해야 하는 상황

## Version 정보
- SQL Server(MSSQL): SQL Serer 2017
- OS: Windows 2016
- OS Bit: 64Bit    

----
## sql_exporter 설치   

### Metric 수집 내용   
> metric 정보: sql_collector.yml 

```yaml
    # A collector defining standard metrics for Microsoft SQL Server.
    #
    # It is required that the SQL Server user has the following permissions:
    #
    #   GRANT VIEW ANY DEFINITION TO
    #   GRANT VIEW SERVER STATE TO
    #
    collector_name: mssql_standard

    # Similar to global.min_interval, but applies to the queries defined by this collector only.
    #min_interval: 0s

    metrics:
      - metric_name: mssql_local_time_seconds
        type: gauge
        help: 'Local time in seconds since epoch (Unix time).'
        values: [unix_time]
        query: |
          SELECT DATEDIFF(second, '19700101', GETUTCDATE()) AS unix_time
      - metric_name: mssql_connections
        type: gauge
        help: 'Number of active connections.'
        key_labels:
          - db
        values: [count]
        query: |
          SELECT DB_NAME(sp.dbid) AS db, COUNT(sp.spid) AS count
          FROM sys.sysprocesses sp
          GROUP BY DB_NAME(sp.dbid)
      #
      # Collected from sys.dm_os_performance_counters
      #
      - metric_name: mssql_deadlocks
        type: counter
        help: 'Number of lock requests that resulted in a deadlock.'
        values: [cntr_value]
        query: |
          SELECT cntr_value
          FROM sys.dm_os_performance_counters WITH (NOLOCK)
          WHERE counter_name = 'Number of Deadlocks/sec' AND instance_name = '_Total'
      - metric_name: mssql_user_errors
        type: counter
        help: 'Number of user errors.'
        values: [cntr_value]
        query: |
          SELECT cntr_value
          FROM sys.dm_os_performance_counters WITH (NOLOCK)
          WHERE counter_name = 'Errors/sec' AND instance_name = 'User Errors'
      - metric_name: mssql_kill_connection_errors
        type: counter
        help: 'Number of severe errors that caused SQL Server to kill the connection.'
        values: [cntr_value]
        query: |
          SELECT cntr_value
          FROM sys.dm_os_performance_counters WITH (NOLOCK)
          WHERE counter_name = 'Errors/sec' AND instance_name = 'Kill Connection Errors'
      - metric_name: mssql_page_life_expectancy_seconds
        type: gauge
        help: 'The minimum number of seconds a page will stay in the buffer pool on this node without references.'
        values: [cntr_value]
        query: |
          SELECT top(1) cntr_value
          FROM sys.dm_os_performance_counters WITH (NOLOCK)
          WHERE counter_name = 'Page life expectancy'
      - metric_name: mssql_batch_requests
        type: counter
        help: 'Number of command batches received.'
        values: [cntr_value]
        query: |
          SELECT cntr_value
          FROM sys.dm_os_performance_counters WITH (NOLOCK)
          WHERE counter_name = 'Batch Requests/sec'
      - metric_name: mssql_log_growths
        type: counter
        help: 'Number of times the transaction log has been expanded, per database.'
        key_labels:
          - db
        values: [cntr_value]
        query: |
          SELECT rtrim(instance_name) AS db, cntr_value
          FROM sys.dm_os_performance_counters WITH (NOLOCK)
          WHERE counter_name = 'Log Growths' AND instance_name <> '_Total'
      - metric_name: mssql_buffer_cache_hit_ratio
        type: gauge
        help: 'Ratio of requests that hit the buffer cache'
        values: [cntr_value]
        query: |
          SELECT cntr_value
          FROM sys.dm_os_performance_counters
          WHERE [counter_name] = 'Buffer cache hit ratio'
      - metric_name: mssql_checkpoint_pages_sec
        type: gauge
        help: 'Checkpoint Pages Per Second'
        values: [cntr_value]
        query: |
          SELECT cntr_value
          FROM sys.dm_os_performance_counters
          WHERE [counter_name] = 'Checkpoint pages/sec'
      #
      # Collected from sys.dm_io_virtual_file_stats
      #
      - metric_name: mssql_io_stall_seconds
        type: counter
        help: 'Stall time in seconds per database and I/O operation.'
        key_labels:
          - db
          - type
        value_label: operation
        values:
          - read
          - write
        query_ref: mssql_io_stall
      - metric_name: mssql_io_stall_total_seconds
        type: counter
        help: 'Total stall time in seconds per database.'
        key_labels:
          - db
          - type
        values:
          - io_stall
        query_ref: mssql_io_stall

      #
      # Collected from sys.dm_os_process_memory
      #
      - metric_name: mssql_resident_memory_bytes
        type: gauge
        help: 'SQL Server resident memory size (AKA working set).'
        values: [resident_memory_bytes]
        query_ref: mssql_process_memory

      - metric_name: mssql_virtual_memory_bytes
        type: gauge
        help: 'SQL Server committed virtual memory size.'
        values: [virtual_memory_bytes]
        query_ref: mssql_process_memory

      - metric_name: mssql_memory_utilization_percentage
        type: gauge
        help: 'The percentage of committed memory that is in the working set.'
        values: [memory_utilization_percentage]
        query_ref: mssql_process_memory

      - metric_name: mssql_page_fault_count
        type: counter
        help: 'The number of page faults that were incurred by the SQL Server process.'
        values: [page_fault_count]
        query_ref: mssql_process_memory

      #
      # Collected from sys.dm_os_sys_memory
      #
      - metric_name: mssql_os_memory
        type: gauge
        help: 'OS physical memory, used and available.'
        value_label: 'state'
        values: [used, available]
        query: |
          SELECT
            (total_physical_memory_kb - available_physical_memory_kb) * 1024 AS used,
            available_physical_memory_kb * 1024 AS available
          FROM sys.dm_os_sys_memory
      - metric_name: mssql_os_page_file
        type: gauge
        help: 'OS page file, used and available.'
        value_label: 'state'
        values: [used, available]
        query: |
          SELECT
            (total_page_file_kb - available_page_file_kb) * 1024 AS used,
            available_page_file_kb * 1024 AS available
          FROM sys.dm_os_sys_memory
      - metric_name: mssql_disk_space_used_database_bytes_total
        type: gauge
        help: 'Disk used by database.'
        key_labels:
          - db
          - type_desc
        values: [CurrentSizeB]
        query: |
          SELECT DB_NAME(database_id) AS db, type_desc, sum(size) * 1024.0 AS CurrentSizeB
          FROM sys.master_files
          WHERE database_id > 0 AND type IN (0,1)
          GROUP BY type_desc, DB_NAME(database_id)
      - metric_name: mssql_disk_space_avaiable_bytes_total
        type: gauge
        help: 'Available space from mount point'
        key_labels:
          - volume_mount_point
        values: [available_bytes]
        query_ref: mssql_disk_space_avaiable_bytes_total
      - metric_name: mssql_disk_space_total_bytes_total
        type: gauge
        help: 'Used space from mount point'
        key_labels:
          - volume_mount_point
        values: [total_bytes]
        query_ref: mssql_disk_space_avaiable_bytes_total
      - metric_name: mssql_table_size_bytes_total
        type: gauge
        help: 'Disk used by table.'
        query_ref: mssql_table_size
        key_labels:
          - db
          - obj_name
        values: [total_space]
      - metric_name: mssql_table_free_size_bytes_total
        type: gauge
        help: 'Disk used by table.'
        query_ref: mssql_table_size
        key_labels:
          - db
          - obj_name
        values: [unused_space]
      - metric_name: mssql_table_used_size_bytes_total
        type: gauge
        help: 'Disk used by table.'
        query_ref: mssql_table_size
        key_labels:
          - db
          - obj_name
        values: [used_space]
      - metric_name: mssql_transactions_total
        type: counter
        help: 'Transactions for each db'
        query_ref: mssql_transactions_total
        key_labels:
          - db
        values: [transactions_total]
    queries:
      - query_name: mssql_disk_space_avaiable_bytes_total
        query: |
          SELECT sum(available_bytes) available_bytes ,sum(total_bytes) total_bytes, ISNULL(volume_mount_point,'') volume_mount_point
          FROM sys.master_files AS f
          CROSS APPLY sys.dm_os_volume_stats(f.database_id, f.file_id)
          GROUP BY volume_mount_point;
      - query_name: mssql_transactions_total
        query: |
          SELECT instance_name as db, cntr_value as transactions_total
          FROM sys.dm_os_performance_counters
          WHERE counter_name = 'Transactions/sec'
      # Populates `mssql_table_size`
      - query_name: mssql_table_size
        query: |
          IF OBJECT_ID('tempdb.dbo.#space') IS NOT NULL
              DROP TABLE #space

          CREATE TABLE #space (
                                  [db_name] SYSNAME
              , obj_name SYSNAME
              , total_pages BIGINT
              , used_pages BIGINT
              , total_rows BIGINT
          )

          DECLARE @SQL NVARCHAR(MAX)

          SELECT @SQL = STUFF((
                                  SELECT '
              USE [' + d.name + ']
              INSERT INTO #space ([db_name], obj_name, total_pages, used_pages, total_rows)
              SELECT DB_NAME(), SCHEMA_NAME(o.[schema_id]) + ''.'' + o.name, t.total_pages, t.used_pages, t.total_rows
              FROM (
                  SELECT
                        i.[object_id]
                      , total_pages = SUM(a.total_pages)
                      , used_pages = SUM(a.used_pages)
                      , total_rows = SUM(CASE WHEN i.index_id IN (0, 1) AND a.[type] = 1 THEN p.[rows] END)
                  FROM sys.indexes i
                  JOIN sys.partitions p ON i.[object_id] = p.[object_id] AND i.index_id = p.index_id
                  JOIN sys.allocation_units a ON p.[partition_id] = a.container_id
                  WHERE i.is_disabled = 0
                      AND i.is_hypothetical = 0
                  GROUP BY i.[object_id]
              ) t
              JOIN sys.objects o ON t.[object_id] = o.[object_id]
              WHERE o.name NOT LIKE ''dt%''
                  AND o.is_ms_shipped = 0
                  AND o.type = ''U''
                  AND o.[object_id] > 255;'
                                  FROM sys.databases d
                                  WHERE d.[state] = 0
                                  FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 2, '')

          EXEC sys.sp_executesql @SQL

          SELECT
              [db_name] as db
              , obj_name
              , total_space = CAST(total_pages * 1024.0 * 8. AS DECIMAL(18,2))
              , used_space = CAST(used_pages * 1024.0 * 8. AS DECIMAL(18,2))
              , unused_space = CAST((total_pages - used_pages) * 1024.0 * 8. AS DECIMAL(18,2))
          FROM #space
      # Populates `mssql_io_stall` and `mssql_io_stall_total`
      - query_name: mssql_io_stall
        query: |
          SELECT
              type_desc as type,
              cast(DB_Name(a.database_id) as varchar) AS [db],
              sum(io_stall_read_ms) / 1000.0 AS [read],
              sum(io_stall_write_ms) / 1000.0 AS [write],
              sum(io_stall) / 1000.0 AS io_stall
          FROM
              sys.dm_io_virtual_file_stats(null, null) a
                  INNER JOIN sys.master_files b ON a.database_id = b.database_id AND a.file_id = b.file_id
                  CROSS APPLY sys.dm_os_volume_stats(a.database_id, a.file_id)
          GROUP BY a.database_id, type_desc
      # Populates `mssql_resident_memory_bytes`, `mssql_virtual_memory_bytes`, `mssql_memory_utilization_percentage` and
      # `mssql_page_fault_count`.
      - query_name: mssql_process_memory
        query: |
          SELECT
            physical_memory_in_use_kb * 1024 AS resident_memory_bytes,
            virtual_address_space_committed_kb * 1024 AS virtual_memory_bytes,
            memory_utilization_percentage,
            page_fault_count
          FROM sys.dm_os_process_memory
---
```

### 수집 Agent 설정 sql_exporter.yml   

> Agent Run: 아래 파일을 기준으로 실행하게 됨.   

```yaml
# Global defaults.
    global:
      # Subtracted from Prometheus' scrape_timeout to give us some headroom and prevent Prometheus from timing out first.
      scrape_timeout_offset: 500ms
      # Minimum interval between collector runs: by default (0s) collectors are executed on every scrape.
      min_interval: 0s
      # Maximum number of open connections to any one target. Metric queries will run concurrently on multiple connections,
      # as will concurrent scrapes.
      max_connections: 3
      # Maximum number of idle connections to any one target. Unless you use very long collection intervals, this should
      # always be the same as max_connections.
      max_idle_connections: 3

    # The target to monitor and the collectors to execute on it.
    target:
      # Data source name always has a URI schema that matches the driver name. In some cases (e.g. MySQL)
      # the schema gets dropped or replaced to match the driver expected DSN format.
      data_source_name: 'sqlserver://USER:PASSWORD@mssql:1433'

      # Collectors (referenced by name) to execute on the target.
      collectors: [mssql_standard]

    # Collector files specifies a list of globs. One collector definition is read from each matching file.
    collector_files:
      - "C:\Prometheus\*.collector.yml"
```     

----   
#### Agent Windows Service Create    

```powershell
New-Service -name "SqlExporterSvc" `
-BinaryPathName "C:\Prometheus\sql_exporter.exe -config.file C:\Prometheus\sql_exporter.yml" `
-StartupType Automatic `
-DisplayName "Prometheus SQL Exporter"
```


#### Grafana Dashboard File
- [Dashboard Json File](https://github.com/mdgome/mdgome.github.io/blob/main/asset/file/sqlserver_grafa_dashboard.json)
- [Dashboard Ref](https://promcat.io/apps/mssql)



----
### Prometheus Configure    

> Windows Server(MSSQL, SQL Server) DB 서버 Agent 설치 완료 이후
> 아래와 같이 Prometheus 서버 설정 변경

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

