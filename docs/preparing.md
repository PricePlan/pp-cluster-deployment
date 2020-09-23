## Подготовка к запуску основного playbook'а

Для выполнения шагов данного и последующих этапов крайне желательно знакомство
с [Ansible](https://docs.ansible.com/ansible/latest/index.html)

### 1. Ключи для пользователя Ansible

Если у вас имеется ECDSA или RSA ключ (или есть возможность его создать)
для пользовтеля, от имени которого будут выполняться действия playbook'ов,
поместите означенный ключ в директорию *keys* проекта.

Если нет, используйте подключение с паролем.

### 2. Файл inventory

Скопируйте *hosts.example* в *hosts*

```bash
$ cp hosts.example hosts
```

Отредактируйте *hosts*. Он представляет собой [inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#intro-inventory).
В нём следует отредактировать, минимум, первые 7 строк:

```yaml
# Inventory example
[master]
pp1 ansible_host=100.200.50.1

[replica]
pp2 ansible_host=100.200.50.2
pp3 ansible_host=100.200.50.3

```

Задайте свои имена нод (опционально). Чем компактнее они будут, тем лучше.
**IP-адреса** здесь следует указать, необходимые **для подключения Ansible**
к нодам по протоколу SSH.

### 3. Файл настроек группы pp_cluster

Скопируйте *group_vars/pp_cluster.yaml.example* в *group_vars/pp_cluster.yaml*

```bash
$ cp group_vars/pp_cluster.yaml.example group_vars/pp_cluster.yaml
```

Отредактируйте *group_vars/pp_cluster.yaml*. Здесь заданы все важные настройки
для развёртывания и работы кластера. В этом файле осознанно изменять можно
любые параметры. Помимо всех паролей и секретов, которым необходимо присвоить
собственные уникальные значения, уделить внимание следует таким переменным,
как:

* cluster (**IP-адреса** указать **для коммуникации сервисов внутри кластера**)
* dns_servers
* web_domain_name

Следующая секция может быть раскомментирована и использована только для запуска
*bootstrap.yaml*, если пользователь для ansible ещё не настроен и разрешён
доступ привилегированному пользователю по SSH.

```yaml
# Bootstrap
# ansible_user: root
# ansible_ssh_pass: 'ChangeThisRightNow' # Change it!
```

В дальнейшем следует настроить и использовать параметры из секции Common.
Подробнее см. в [документации](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#connection-variables).

### 4. Запуск проверочно-подготовительного playbook'а

Перед запуском основного playbook'а следует запустить подготовительный:

```bash
$ ansible-playbook playbooks/bootstrap.yaml

PLAY [prepare new hosts to work with Ansible] ****************************************************************

TASK [install Python] ****************************************************************************************
changed: [pp1] => (item=python3)
changed: [pp2] => (item=python3)
changed: [pp1] => (item=python3-dnf)
changed: [pp2] => (item=python3-dnf)
changed: [pp1] => (item=python3-libselinux)
changed: [pp2] => (item=python3-libselinux)
changed: [pp2] => (item=python3-libsemanage)
changed: [pp1] => (item=python3-libsemanage)
changed: [pp3] => (item=python3)
changed: [pp3] => (item=python3-dnf)
changed: [pp3] => (item=python3-libselinux)
changed: [pp3] => (item=python3-libsemanage)

TASK [ensure docker group exists] ****************************************************************************
changed: [pp3]
changed: [pp2]
changed: [pp1]

TASK [ensure locksmith user exists] **************************************************************************
changed: [pp2]
changed: [pp3]
changed: [pp1]

TASK [ensure locksmith user exists] **************************************************************************
skipping: [pp3]
skipping: [pp1]
skipping: [pp2]

TASK [check authorized key file] *****************************************************************************
ok: [pp3]
ok: [pp2]
ok: [pp1]

TASK [set authorized key if file exists] *********************************************************************
ok: [pp3]
ok: [pp1]
ok: [pp2]

PLAY RECAP ***************************************************************************************************
pp1                        : ok=3    changed=3    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
pp2                        : ok=3    changed=3    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
pp3                        : ok=3    changed=3    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

```

Добейтесь успешного вывода команды, это будет означать, что на данном
этапе всё настроено правильно.

После того, как *bootstrap.yaml* отработает, если вы раскоментировали секцию Bootstrap, закомментируйте её.
