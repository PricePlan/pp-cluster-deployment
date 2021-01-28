## Восстановление отдельной ноды кластера

Будем предполагать, что все манипуляции проводятся на хосте, где только была
установлена ОС Centos 8. Если у вас другая ОС, напишите запрос на
support@priceplan.pro для получения инструкций, адаптированных под вашу
операционную систему.

После постановки нового хоста, предназначенного для замещения вышедшей из
строя ноды кластера PricePlan, под управление можно начинать процесс
восстановления состояния полноты кластера.

Целесообразно устанавливать необходимые компоненты на новую ноду поэтапно.

### Установка базовых зависимостей

Для начала следует выполнить следующую команду

```
$ ansible-playbook playbooks/pp-cluster.yaml -t base -l <nodename>
```

Плэйбук должен отработать без ошибок.

### Установка etcd

От сервиса [etcd](https://etcd.io/) зависит наше решение на базе
[Patroni](https://patroni.readthedocs.io/), обеспечивающее репликацию и
файловер в кластере PostgreSQL.

По нашим наблюдениям *etcd* хорошо переносит включение и выключение отдельных
нод и всего кластера, но восстановление одной ноды может оказаться весьма
нетривиальным занятием. Специально для такого случая, мы предусмотрели
режим быстрой переустановки всего кластера *etcd*:

```
$ ansible-playbook playbooks/pp-cluster.yaml -t etcd -l pp_cluster \
	-e etcd_reinit=true
```

Выполнение всех действий данной команды требует порядка 30 секунд. При этом
возможна ситуация, когда PostgreSQL на несколько секунд может оказаться
недоступен т.к. HAProxy проверяет доступность нод через REST API Patroni, сам
же PostgreSQL своей работы не прерывает. Поэтому проведение работ по
восстановлению нод желательно планировать на время минимальной нагрузки на
систему.

После успешного завершения работы приведённой выше команды следует зайти по
*ssh* на любую ноду кластера и убедиться, что все экземпляры etcd работают
соглассовано:

```
$ docker exec -it etcd /bin/bash
I have no name!@pp1:/opt/bitnami/etcd$ etcdctl -w table endpoint --cluster status
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|        ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://10.128.0.14:2379 | 276a9a318f28dbc9 |  3.4.14 |   20 kB |     false |      false |         2 |       8735 |               8735 |        |
|  http://10.128.0.4:2379 | 84875baefd65cfc1 |  3.4.14 |   20 kB |      true |      false |         2 |       8735 |               8735 |        |
| http://10.128.0.12:2379 | bad217d26fb95564 |  3.4.14 |   20 kB |     false |      false |         2 |       8735 |               8735 |        |
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Примерно такой должен получиться вывод `etcdctl`.

### Установка PostgreSQL и Patroni

Перед установкой и запуском ноды PostgreSQL следует убедиться, что директория
данных СУБД в файловой системе данного хоста пуста (или ещё отсутствует).
После чего можно запускать плэйбук:

```
$ ansible-playbook playbooks/pp-cluster.yaml -t pg --skip-tags etcd \
	-l <nodename> -e assume_replica=true
```

Проверить, что кластер пополнился ещё одной нодой, после того, как команда
благополучно завершится, можно на любой ноде, вызвав `patronictl`:

```
# . /opt/pves/patroni/bin/activate
(patroni) [root@pp1 data]# patronictl -c /etc/patroni/patroni.yaml topology
+ Cluster: pp-pgcluster (6876007901448217093) --+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pp2    | 10.128.0.12 | Leader  | running | 27 |           |
| + pp1  | 10.128.0.4  | Replica | running | 27 |         0 |
| + pp3  | 10.128.0.14 | Replica | running | 27 |         0 |
+--------+-------------+---------+---------+----+-----------+
```

### Установка Redis

Чтобы восстановить ноду Redis, необходимо предварительно узнать, какой хост
в данный момент является мастером. На мастере вывод `info replication`
будет примерно таким (вместе с набранными командами):

```
$ docker exec -it redis /bin/sh
/data # redis-cli
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
min_slaves_good_slaves:1
slave0:ip=10.128.0.14,port=6379,state=online,offset=10071,lag=1
master_replid:746686015505634dd73b5b5ef7e54c19f1a56a92
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:10071
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:10071
```

После того как запущенный таким образом:

```
$ ansible-playbook playbooks/pp-cluster.yaml -t redis -l <nodename> \
	-e redis_master_host=<master_nodename>

```

плэйбук отработает без ошибок, вывод должен стать похож на следующий образец:

```
# Replication
role:master
connected_slaves:2
min_slaves_good_slaves:2
slave0:ip=10.128.0.14,port=6379,state=online,offset=20110,lag=0
slave1:ip=10.128.0.4,port=6379,state=online,offset=20110,lag=0
master_replid:746686015505634dd73b5b5ef7e54c19f1a56a92
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:20110
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:20110
```

### Установка RabbitMQ

При желании можно предварительно уточнить версию RabbitMQ на уже работающих
нодах (той же командой, что и для проверки успешности восстановления) и
приведённый ниже вариант запуска плэйбука дополнить параметрами:
`-e rabbitmq_version=<specified_version>`.

Установка RabbitMQ осуществляется командой:

```
$ ansible-playbook playbooks/pp-cluster.yaml -t rabbitmq -l <nodename>
```

Проверка — набором команд:

```
$ docker exec -it rabbitmq /bin/sh
/ # rabbitmqctl cluster_status
Cluster status of node rabbit@pp2 ...
Basics

Cluster name: pp_mq

Disk Nodes

rabbit@pp1
rabbit@pp2
rabbit@pp3

Running Nodes

rabbit@pp1
rabbit@pp2
rabbit@pp3

Versions

rabbit@pp1: RabbitMQ 3.8.9 on Erlang 23.1.4
rabbit@pp2: RabbitMQ 3.8.9 on Erlang 23.1.4
rabbit@pp3: RabbitMQ 3.8.9 on Erlang 23.1.4

Maintenance status

Node: rabbit@pp1, status: not under maintenance
Node: rabbit@pp2, status: not under maintenance
Node: rabbit@pp3, status: not under maintenance
...
```

### Установка HAProxy, nginx

Просто запустите следующую команду и дождитесь её успешного завершения:

```
$ ansible-playbook playbooks/pp-cluster.yaml -t haproxy,nginx-server \
	-l <nodename>
```

### Установка ELK

Команда, используемая для восстановление сервисов ELK, тривиальна:

```
$ ansible-playbook playbooks/pp-cluster.yaml -t elk -l <nodename>
```

Проверка осуществляется: 1) контролем вывода `docker ps` на целевой ноде.
В нём должны присутствовать данные о контейнерах *elasticsearch* и
*logstash* (и *kibana*, если она базируется на данном хосте), подтверждающие
их песпроблемный старт. 2) на любой ноде:

```
$ curl -s http://localhost:9200/_cluster/health | jq .
{
  "cluster_name": "pp-elk-cluster",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 10,
  "active_shards": 20,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 0,
  "active_shards_percent_as_number": 100
}
```

3) Если Kibana подлежала восстановлению, она будет доступна для взаимодействия
с ней посредством броузера на прежнем URL.

### Финальный этап — установка PricePlan

Команда для восстановления PricePlan на вновь вводимой в эксплуатацию ноде
кластера отличается от команды первоначальной установке (как и практически
все в данном разделе руководства) только указанием, выполнять тэггированный
набор действий плэйбука на задданном хосте:

```
$ ansible-playbook playbooks/pp-cluster.yaml -t pp -l <nodename>
```

Вывод команды `docker ps -a --filter="name=pp_"` на целевой ноде будет
свидетельствоать об успешном завершении процесса восстановления.
