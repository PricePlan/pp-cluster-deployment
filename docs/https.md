## Переход на HTPPS

На ранне развёрнутом кластере, если необходимо переключить приложение на
работу по HTTPS, проделайте следующие манипуляции:

1. Убедитесь, что в файле *group_var/all.yaml* присутсвует строка:

```yaml
web_protocol: https
```

2. Поместите в директорю *files/ssl* файлы *priceplan.crt* и *priceplan.key*.

3. Обновите конфигурацию виртуального хоста nginx командой:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t settings -l pp_cluster
```

4. Внесите изменения в конфигурацию HAProxy на балансировщиках:

```bash
$ time ansible-playbook playbooks/pp-cluster.yaml -t haproxy_config \
	-l load_balancers
```

5. Зайдите в броузере на порты 8080 хостов, где развёрнуты балансировщики,
убедитесь, что все ноды работают в штатном режиме.
