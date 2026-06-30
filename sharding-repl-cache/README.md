# sharding-repl-cache

Шардированный кластер MongoDB с репликацией (каждый шард — replica set из 3 узлов)
и кешированием запросов через Redis.
Приложение `pymongo-api` подключается к роутеру `mongos` и к Redis.

## Состав кластера

| Сервис          | Тип                       | replSet         | Порт |
|-----------------|---------------------------|-----------------|------|
| `pymongo_api`   | приложение                | —               | 8080 |
| `redis`         | кеш (Redis)               | —               | 6379 |
| `mongos_router` | роутер (mongos)           | —               | 27017 |
| `configSrv`     | config server             | `config_server` | 27017 |
| `shard1-1`      | шард 1, узел 1 (PRIMARY)  | `shard1`        | 27018 |
| `shard1-2`      | шард 1, узел 2 (secondary)| `shard1`        | 27018 |
| `shard1-3`      | шард 1, узел 3 (secondary)| `shard1`        | 27018 |
| `shard2-1`      | шард 2, узел 1 (PRIMARY)  | `shard2`        | 27018 |
| `shard2-2`      | шард 2, узел 2 (secondary)| `shard2`        | 27018 |
| `shard2-3`      | шард 2, узел 3 (secondary)| `shard2`        | 27018 |

Кеширование включается переменной окружения `REDIS_URL: "redis://redis:6379"`
у сервиса `pymongo_api`. Кешируется эндпоинт `/<collection_name>/users`.

## Запуск

Из директории `sharding-repl-cache`:

```shell
docker compose up -d --build
```

## Инициализация кластера

Выполните шаги по порядку.

### 1. Инициализация config server

```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
})
EOF
```

### 2. Настройка репликации для shard1 (3 реплики)

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1-1:27018" },
    { _id: 1, host: "shard1-2:27018" },
    { _id: 2, host: "shard1-3:27018" }
  ]
})
EOF
```

### 3. Настройка репликации для shard2 (3 реплики)

```shell
docker compose exec -T shard2-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2-1:27018" },
    { _id: 1, host: "shard2-2:27018" },
    { _id: 2, host: "shard2-3:27018" }
  ]
})
EOF
```

Дайте replica set'ам несколько секунд, чтобы выбрать PRIMARY:

```shell
sleep 10
```

### 4. Добавление шардов в кластер и включение шардирования

Шарды добавляем как replica set'ы (имя replSet + список узлов).
Коллекцию шардируем **до** наполнения данными — по хешированному ключу `name`.

```shell
docker compose exec -T mongos_router mongosh --port 27017 --quiet <<EOF
sh.addShard("shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018")
sh.addShard("shard2/shard2-1:27018,shard2-2:27018,shard2-3:27018")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { name: "hashed" })
EOF
```

### 5. Наполнение данными (1000 документов)

```shell
docker compose exec -T mongos_router mongosh --port 27017 --quiet <<EOF
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i })
EOF
```

## Проверка

### Через приложение

Откройте http://localhost:8080 — в ответе будет:

- `mongo_topology_type: "Sharded"`;
- список шардов с указанием реплик в каждом
  (`shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018`);
- общее количество документов (≥ 1000).

```shell
curl --silent http://localhost:8080/
```

### Количество документов в каждом шарде

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

```shell
docker compose exec -T shard2-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

### Статус репликации (количество реплик в группе)

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.status().members.map(m => m.name + " - " + m.stateStr)
EOF
```

```shell
docker compose exec -T shard2-1 mongosh --port 27018 --quiet <<EOF
rs.status().members.map(m => m.name + " - " + m.stateStr)
EOF
```

### Проверка кеширования

Эндпоинт `/<collection_name>/users` кешируется в Redis на 60 секунд.
Первый запрос идёт в MongoDB (искусственная задержка ~1 секунда), повторные
отдаются из кеша — время должно упасть до < 100 мс.

```shell
echo "Первый запрос (из MongoDB):"
curl --silent -o /dev/null -w "  время: %{time_total} c\n" http://localhost:8080/helloDoc/users

echo "Второй запрос (из кеша Redis):"
curl --silent -o /dev/null -w "  время: %{time_total} c\n" http://localhost:8080/helloDoc/users
```

Что в ответе приложения по адресу http://localhost:8080/ должно стоять
`"cache_enabled": true`.

## Доступные эндпоинты

Swagger: http://localhost:8080/docs
