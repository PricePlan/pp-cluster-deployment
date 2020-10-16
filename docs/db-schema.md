## Создание схемы данных

Если вы не переносите данные организации с другого сервера, создание схемы
данных и наполнение БД стартовой информацией произведите командами:

```bash
$ docker exec -it pp_uwsgi /bin/sh
/pp $ PRICEPLAN_ADMIN_PASSWORD="<pp_postgres_password>" ./manage.py create_schema main
```
