# Playing with Loki, Promtail, and logcli


```
curl -fSL -o logcli.gz "https://github.com/grafana/loki/releases/download/v1.4.1/logcli-linux-amd64.zip"
curl -fSL -o loki.gz "https://github.com/grafana/loki/releases/download/v1.4.1/loki-linux-amd64.zip"
curl -fSL -o promtail.gz "https://github.com/grafana/loki/releases/download/v1.4.1/promtail-linux-amd64.zip"

gunzip logcli.gz
gunzip loki.gz
gunzip promtail.gz

chmod +x loki promtail logcli

mkdir bin
mv loki bin
mv promtail bin
mv logcli bin
```

config-promtail.yaml
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/promtail/loki/stuff/positions.yaml

clients:
  - url: http://127.0.0.1:3100/loki/api/v1/push

scrape_configs:
  - job_name: journal
    journal:
      max_age: 12h
      path: /var/log/journal
      labels:
        job: systemd-journal
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'

  - job_name: foo
    static_configs:
    - targets:
        - localhost
      labels:
        job: foo
        __path__: /tmp/promtail-loki/mylog/folder1/*

  - job_name: bar
    static_configs:
    - targets:
       - localhost
      labels:
       job: bar
       __path__: /tmp/promtail-loki/mylog/folder2/*
```

config-loki.yaml
```
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
  - from: 2018-04-15
    store: boltdb
    object_store: filesystem
    schema: v9
    index:
      prefix: index_
      period: 168h

storage_config:
  boltdb:
    directory: /tmp/promtail-loki/stuff/index

  filesystem:
    directory: /tmp/promtail-loki/stuff/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0

table_manager:
  chunk_tables_provisioning:
    inactive_read_throughput: 0
    inactive_write_throughput: 0
    provisioned_read_throughput: 0
    provisioned_write_throughput: 0
  index_tables_provisioning:
    inactive_read_throughput: 0
    inactive_write_throughput: 0
    provisioned_read_throughput: 0
    provisioned_write_throughput: 0
  retention_deletes_enabled: false
  retention_period: 0
```

### Shell 1:
```
./bin/promtail -config.file config-promtail.yaml
```

### Shell 2:
```
./bin/loki -config.file config-loki.yaml
```

### Shell 3:
```
./bin/logcli

./bin/logcli labels 
./bin/logcli labels job

./bin/logcli query '{job="foo"}'
./bin/logcli query '{job="bar"}'

./bin/logcli query '{job="varlogs"}' --tail


./bin/logcli query '{job="foo", job="bar"}'
./bin/logcli query '{job="foo", job!~"bar"}'



./bin/logcli query '{job="bar"} |= "Hello"'
./bin/logcli query '{job="bar"} |= "world"'

./bin/logcli query '{job="bar"} |= "Hello" != "world"

```

> Check [LogQL](https://github.com/grafana/loki/blob/master/docs/sources/logql/_index.md) syntax

# Reset
> Delete `stuff` folder if you want to start over

## URLS
http://localhost:9080/targets
http://localhost:3100/metrics

