Описание и важные тезисы
--------------

* Роль для установки и конфигурирования Redis Server в Linux
* Может устанавливать и настраивать как Redis StandAlone, так и Redis Cluster, так и Redis-Replica (ro\rw)
* Сознательно не затрагивается sysctl, т.к. возможен ролевой конфликт параметров
* Инициализация кластера происходит только на одном хосте (первом из play, у которого уникальная redis_server.cluster.ansible_groupname) - поэтому важно как запускается playbook
* Приглашение в кластер происходит через CLUSTER MEET с одного хоста (первом из play, у которого уникальная redis_server.cluster.ansible_groupname) - поэтому важно как запускается playbook
* Конфигурирование реплики производится на основе конфигурационного файла и только
* Допускается (экспериментально) автоматизированное приглашение в Redis Cluster новых хостов [по CLUSTER MEET](https://redis.io/commands/cluster-meet/)
  * Reshard выполняется вручную
  * Если приглашенные не ответили - в кластере они не появятся, ошибок не будет (приглашение же отправлено)

Доступные теги:
--------------
* config - отработать только конфиг самого Redis Server (и перезапустить его в случае изменений)
* stop_redis - остановить Redis Server через systemd
* start_redis - запусить Redis Server через systemd
* stop_redis_all - если на хосте несколько инстансов Redis Server - это завершит их всех
* logrotate - отработать только конфиг для logrotated
* unit - отработать только unit systemd (и сделать reload для systemd в случае изменений, Redis перезапущен не будет, даже если команда запуска в unit измениться)
* install_instance - по сути сделает все необходимое при чистой установке Redis Server без доп. фич вида реплик\кластера
* init_cluster - выполнить таски по инициализации кластера (Redis Server'ы должны быть запущены)
* meet_cluster - выполнить таски по приглашению новых нод в кластер (Redis Server'ы должны быть запущены)
* manage_cluster - выполнить кластерные таски (по сути инициализация и приглашение). Reshard нужно делать вручную!
* setup_cluster_commands - в целях упрощения чтения/дебага/воспроизведения формируется bash-скрипт, который и определяет, что будет выполняться. Этот тег позволяет его сгенерировать по пути /tmp/redis_cluster_commands.<redis_server.listen_port>.sh и посмотреть\потыкать руками
* remove_redis_data - удалить redis_server.workdir (т.е. данные инстанса (или нескольких инстансов при определенных условиях))

Пример описания для хоста
--------------

Для настройки используется dict redis_server

Для глубокого ознакомления с какой-либо директивой (в случае неясности) лучше сверяться с файлом templates/redis.conf.j2 (в нем подробные комментарии для каждой директивы и их подстановка методами Jinja)

#### Частный пример описания dict redis_server

(по сути является default конфигурацией Redis Server, dict можно скопировать и использовать где угодно)

__Примечание__: некоторые значения определены как __пустая строка__ ("") - это значит, что:
1. либо используется значение по-умолчанию (вшитое в сам Redis Server) без создания строчки в конфигурации
2. либо нет возможности определить значение заранее (например, IP-адрес Redis Master при создании реплики)

### Пояснение по секции redis_server.logrotate

Из файла templates/redis.logrotate.j2 формируется файл конфигурации для logrotated (с наибольшей долей вероятности его не придется менять), но все же вероятность есть

```
redis_server:
  config_file: "/etc/redis.conf" # Создать здесь файл конфигурации Redis Server
  systemd:
    service_name: "redis"        # По этому имени будем обращаться к Redis Server через systemd
  package_name: "redis"          # Производить установку этого пакета
  logrotate:
    enabled: True                # Если False не будет попытки изменять logrotate файл
    filename: "redis"
    rotate_filemask: "/var/log/redis/*.log"
    period: "weekly"
    count: "10"
    copytruncate: True
    delaycompress: True
    compress: True
    notifempty: True
    missingok: True
  protected_mode: "yes"
  bind: "127.0.0.1"              # Неочевидный нюанс: если будет кластер и\или реплики, 1-ым IP нужно указывать не LOOP, т.е. например: "192.168.200.2 127.0.0.1"
  listen_port: "6379"
  tcp_backuplog: "511"
  unix_socket:
    enabled: False
    path: "/tmp/redis.sock"
    mode: "700"
  close_connection_after_timeout: "0"
  tcp_keepalive: "300"
  daemonize: "no"
  supervised: "no"
  pid_file: "/var/run/redis_6379.pid"
  log_level: "notice"
  logfile: "/var/log/redis/redis.log"
  syslog:
    enabled: False
    ident: "redis"
    facility: "local0"
  databases: "16"
  always_show_logo: "yes"
  db_saves:
    rule1:
      enabled: True
      after_seconds: "900"
      when_changed: "1"
    rule2:
      enabled: True
      after_seconds: "300"
      when_changed: "10"
    rule3:
      enabled: True
      after_seconds: "60"
      when_changed: "10000"
  stop_writes_on_bgsave_error: "yes"
  rdbcompression: "yes"
  rdbchecksum: "yes"
  workdir: "/var/lib/redis"
  dbfilename: "dump.rdb"
  replica:
    enabled: False
    masterip: "127.0.0.1"
    masterport: "6379"
    masterauth_password: ""
    replica_serve_stale_data: "yes"
    replica_read_only: "yes"
    repl_ping_replica_period: ""
    announce:
      enabled: False
      replica-announce-ip: ""
      replica-announce-port: "6379"
    ignore_maxmemory: ""
  repl_diskless_sync: "no"
  repl_diskless_sync_delay: "5"
  repl_timeout: ""
  repl_disable_tcp_nodelay: "no"
  repl_backlog_size: ""
  repl_backlog_ttl: ""
  replica_priority: "100"
  min_replicas_to_write: ""
  min_replicas_max_lag: ""
  passauth_password: ""
  command_renames:
    CONFIG:
      enable_renaming: False
      value: ""
  maxclients: ""
  maxmemory_limit:
    enabled: False
    value: "1gb"
    policy: "noeviction"
  maxmemory_samples: ""
  lazyfree_lazy_eviction: "no"
  lazyfree_lazy_expire: "no"
  lazyfree_lazy_server_del: "no"
  replica_lazy_flush: "no"
  appendonly: "no"
  appendfilename: "appendonly.aof"
  appendfsync: "everysec"
  no_appendfsync_on_rewrite: "no"
  auto_aof_rewrite_percentage: "100"
  auto_aof_rewrite_min_size: "64mb"
  aof_load_truncated: "yes"
  aof_use_rdb_preamble: "yes"
  lua_time_limit: "5000"
  cluster:
    enabled: False
    ansible_groupname: "<some_ansible_group>"   # группа ansible, которая и определяет, какие хосты должны быть в Redis Cluster
    replicas: "1"
    config_file: "cluster-1.conf"
    node_timeout_ms: "15000"
    replica_validity_factor: ""
    migration_barrier: ""
    require_full_coverage: ""
    replica_no_failover: ""
    announce:
      enabled: False
      cluster_announce_ip: ""
      cluster_announce-port: "6379"
      cluster_announce_bus_port: "16739"
  slowlog_log_slower_than_ms: "10000"
  slowlog_max_len: "128"
  latency_monitor_threshold_ms: "0"
  notify_keyspace_events: ""
  hash_max_ziplist_entries: "512"
  hash_max_ziplist_value: "64"
  list_max_ziplist_size: "-2"
  list_compress_depth: "0"
  set_max_intset_entries: "512"
  zset_max_ziplist_entries: "128"
  zset_max_ziplist_value: "64"
  hll_sparse_max_bytes: "3000"
  stream_node_max_bytes: "4096"
  stream_node_max_entries: "100"
  activerehashing: "yes"
  client_output_buffer_limits:
    normal:
      values: "0 0 0"
    replica:
      values: "256mb 64mb 60"
    pubsub:
      values: "32mb 8mb 60"
  client_query_buffer_limit: ""
  proto_max_bulk_len: ""
  hz: "10"
  dynamic_hz: "yes"
  aof_rewrite_incremental_fsync: "yes"
  rdb_save_incremental_fsync: "yes"
  lfu_log_factor: ""
  lfu_decay_time: ""
  activedefrag:
    enabled: False
    ignore_bytes: "100mb"
    threshold_lower: "10"
    threshold_upper: "100"
    cycle_min: "5"
    cycle_max: "75"
    max_scan_fields: "1000"
```

Пример playbook
--------------

```
---
- hosts: redis
  become: True
  become_method: sudo
  roles:
    - redis
```

Запуск playbook
--------------

```
ansible-playbook -i inventory -l <some_host> playbooks/redis.yml [-CD]
```

#### Пояснение, что такое "первый из play, у которого уникальная redis_server.cluster.ansible_groupname"

Допустим у нас inventory выглядит так:
cat inventory

```
[redis_cluster_1]
host11
host12
host13
host14
host15
host16

[redis_cluser_2]
host23
host22
host21

[redis_cluser_3]
host32
host33
host31

[redis]
[redis:children]
redis_cluser_1
redis_cluser_2
redis_cluser_3
```

cat group_vars/redis_cluster_1.yml
```
redis_server:
  protected_mode: "no"
  bind: "{{hostvars[inventory_hostname].ansible_host}} 127.0.0.1"
  cluster:
    enabled: True
    ansible_groupname: "redis_cluster_1"
    replicas: "1"
```

cat group_vars/redis_cluster_2.yml
```
redis_server:
  protected_mode: "no"
  bind: "{{hostvars[inventory_hostname].ansible_host}} 127.0.0.1"
  cluster:
    enabled: True
    ansible_groupname: "redis_cluster_2"
    replicas: "0"
```

cat group_vars/redis_cluster_3.yml
```
redis_server:
  protected_mode: "no"
  bind: "{{hostvars[inventory_hostname].ansible_host}} 127.0.0.1"
  cluster:
    enabled: True
    ansible_groupname: "redis_cluster_3"
    replicas: "0"
```

Вариант реализации
```
ansible-playbook -i inventory -l redis playbooks/redis.yml -t install_instance
### Производится установка пакета, настройка systemd unit, redis.conf и запуск (на всех хостах)
ansible-playbook -i inventory -l redis playbooks/redis.yml -t init_cluster
### У нас есть 3 уникальных <<ansible_groupname>> в 3 группах (12 хостах)
### Инициализация Redis Cluster для группы хостов redis_cluster_1 будет произведена на одном хосте, т.е. на хосте host11 (т.к. он первый в группе redis_cluster_1)
### Инициализация Redis Cluster для группы хостов redis_cluster_2 будет произведена на одном хосте, т.е. на хосте host23 (т.к. он первый в группе redis_cluster_2)
### Инициализация Redis Cluster для группы хостов redis_cluster_3 будет произведена на одном хосте, т.е. на хосте host32 (т.к. он перевый в группе redis_clister_3)
```
или
```
ansible-playbook -i inventory -l redis playbooks/redis.yml
```
