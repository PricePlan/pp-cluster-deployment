## Клонирование проекта и создание окружения

1. Склонируйте проект командой, находясь в вашей рабочей директории:

```bash
$ git clone https://github.com/linskiy/pp-cluster-deployment.git
```

2. Перейдите в директорию проекта:

```bash
$ cd pp-cluster-deployment/
```

3. Создайте виртуальное окружение Python:

```bash
$ python3 -m venv pve
```

4. Затем активируйте его:

```bash
$ . pve/bin/activate
```

5. В завершение данного этапа установите Ansible

```bash
$ pip install ansible
```
