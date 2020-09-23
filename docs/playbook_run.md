## Запуск основного playbook'а

Основной playbook целесообразно запускать частями, используя тэги. И между
такими запусками проверять работоспособность развёрнутых компонентов.

Первый такой запуск выполните с тэгом «base»:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t base
```

Основная цель выполнения означенного набора ролей — установить Docker. При
желании можно зайти по ssh на любой из хостов убедиться, что он работает:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

Следующим тэгом следует выбрать «pg»:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t pg
```

Как уже, вероятно, понятно из наименования тэга, должен быть развёнут кластер PostgreSQL. Проверьте факт достижения данной цели командами (от имени root)
на любой из нод:

```bash
# cd /opt/pves/patroni/bin/
# ./patronictl -c /etc/patroni/patroni.yaml topology
+ Cluster: pp-pgcluster (6875590676304178589) --+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pp1    | 10.128.0.8  | Leader  | running |  1 |           |
| + pp2  | 10.128.0.16 | Replica | running |  1 |         0 |
| + pp3  | 10.128.0.10 | Replica | running |  1 |         0 |
+--------+-------------+---------+---------+----+-----------+
```

Дале установите кластер Redis:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t redis
```

Проверку осуществите командами:

```bash
$ docker exec -it redis /bin/sh
/data # redis-cli info replication
```

Для мастер-ноды вывод будет аналогичен следующему:

```
# Replication
role:master
connected_slaves:2
min_slaves_good_slaves:2
slave0:ip=10.128.0.8,port=6379,state=online,offset=61154,lag=1
slave1:ip=10.128.0.10,port=6379,state=online,offset=61154,lag=1
master_replid:984feb033f7940749f78cb06ac500d2d70b6b18e
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:61154
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:61154
```

Для слэйва:

```
# Replication
role:slave
master_host:10.128.0.16
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:18734
slave_priority:100
slave_read_only:1
connected_slaves:0
min_slaves_good_slaves:0
master_replid:984feb033f7940749f78cb06ac500d2d70b6b18e
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:18734
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:18734
```

Затем установите HAProxy:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t haproxy
```

Если зайти броузером зайти на любую ноду на порт 8080, можно будет увидеть
т.н. Statistics Report. Для каждого бэкенда (postgres_back, rdis_back) должен просматриваться мастер в виде ряда таблицы зелёного цвета.

Следующий по списку RabbitMQ. По правде говоря, такой сценарий можно было
реализовтать в роли Ansible, но время поджимало. Поэтому для надёжности лучше
запустить playbook с тэгом «rabbitmq» 2 раза. Первый раз для одной машины,
например:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t rabbitmq -l pp1
```

И второй раз для остальных:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t rabbitmq -l pp2,pp3
```

Для проверки на 1-ой (для которой первый раз запускали) ноде следует запустить просмотр лога RabbitMQ:

```bash
$ docker logs -f rabbitmq
```

И дождаться появления похожих строчек:

```
2020-09-23 08:35:04.640 [info] <0.664.0> node rabbit@pp3 up
2020-09-23 08:35:04.996 [info] <0.664.0> rabbit on node rabbit@pp3 up
2020-09-23 08:35:32.507 [info] <0.664.0> node rabbit@pp2 up
2020-09-23 08:35:32.910 [info] <0.664.0> rabbit on node rabbit@pp2 up
```

Следующая цель — развернуть ELK кластер. Запустите:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t rabbitmq -l nginx-server,elk
```

После того, как отработает playbook зайдите на http://kibana.tl.dr, если
там отобразиться интерфейс Kibana'ы, всё в порядке.

Последний шаг этапа — установка PricePlan. Выполните:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t rabbitmq -l nginx-server,elk
```

Если в процессе проявится подобная ошибка:

```
TASK [priceplan : run UWSGI] ******************************************************************************
fatal: [pp1]: FAILED! => {"changed": false, "msg": "Error creating container: UnixHTTPConnectionPool(host='localhost', port=None): Read timed out. (read timeout=60)"}
...
```

Просто запустите команду ещё раз, это образ докера PricePlan не успел
загрузиться. Вторая попытка должна увенчаться с успехом.
