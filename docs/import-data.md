## Импорт данных, выгруженных на другом сервере

Если вы переносите данные организации с другого сервера, загрузите дамп и разархивируйте его на хосте, являющимся в данный момент мастером для
PostgreSQL, в директорию из которой планируете осуществить импорт.

Если вы ранее создавали схему с демонстрационными данными или проводили импорт,
перед осуществлением импорта следует выполнить команду:

```bash
$ psql --dbname=postgres://priceplan:<pp_postgres_password>@127.0.0.1:5432/priceplan \
  -с 'drop schema main cascade;'
```

Импорт производится следующей командой:

```bash
$ psql --dbname=postgres://priceplan:<pp_postgres_password>@127.0.0.1:5432/priceplan \
  -qf main-<YYMMDD>.sql
```

В процессе импорта никаких ошибок в консоль выведено быть не должно.

Для того, чтобы поиск в интерфейсе приложения работал корректно следует
запустить на одной из нод кластера такие команды:

```bash
$ docker exec -it pp_uwsgi /bin/sh
/pp $ PRICEPLAN_ADMIN_PASSWORD="<pp_postgres_password>" \
  ./manage.py es_reindex --schema=main
```