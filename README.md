# pp-cluster-deployment

Развёртывание кластера PricePlan

## Очень краткое руководство

1. Склонировать проект:

```
$ git clone https://github.com/linskiy/pp-cluster-deployment.git
```

2. Перейти в директорию проекта:

```
$ cd pp-cluster-deployment/
```

3. Создать виртуальное окружение Python:

```
$ python3 -m venv pve
```

4. Активировать его:

```
$ . pve/bin/activate
```

5. Установить Ansible

```
$ pip install ansible
```

6. Положить ключ в директорию keys

7. Скопировать hosts.example в hosts

```
$ cp hosts.example hosts
```

8. Отредактировать hosts

9. Скопировать group_vars/pp_cluster.yaml.example в group_vars/pp_cluster.yaml

```
$ cp group_vars/pp_cluster.yaml.example group_vars/pp_cluster.yaml
```

10. Отредактировать group_vars/pp_cluster.yaml

11. Можно начинать запускать плэйбуки:

* Для яндекса это сначала bootstrap_u.yaml

```
$ ansible-playbook playbooks/bootstrap_u.yaml
```

* И в завершение:

```
$ time ansible-playbook playbooks/pp-cluster.yaml
```

