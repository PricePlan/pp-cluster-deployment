## Подготовка к запуску основного playbook'а

1. Если у вас имеется ECDSA или RSA ключ (или есть возможность его создать)
для пользовтеля, от имени которого будут выполняться действия playbook'ов,
поместите означенный ключ в директорию keys проекта.

2. Скопируйте hosts.example в hosts

```bash
$ cp hosts.example hosts
```

3. Отредактируйте hosts

4. Скопировать group_vars/pp_cluster.yaml.example в group_vars/pp_cluster.yaml

```
$ cp group_vars/pp_cluster.yaml.example group_vars/pp_cluster.yaml
```

5. Отредактировать group_vars/pp_cluster.yaml

6. Можно начинать запускать плэйбуки:

* Для яндекса это сначала bootstrap_u.yaml

```
$ ansible-playbook playbooks/bootstrap_u.yaml
```
