# mongo-sharding

Шардированный кластер MongoDB с приложением `pymongo-api`.

## Состав кластера

| Сервис          | Тип                | Порт (внутри контейнера) |
|-----------------|--------------------|--------------------------|
| `pymongo_api`   | приложение         | 8080                     |
| `mongos_router` | роутер (mongos)    | 27017                    |
| `configSrv`     | config server      | 27017                    |
| `shard1`        | шард (mongod)      | 27018                    |
| `shard2`        | шард (mongod)      | 27018                    |

Приложение обращается только к роутеру `mongos_router`, тот распределяет данные между `shard1` и `shard2`.

## Запуск

Из директории `mongo-sharding`:

```shell
docker compose up -d
```

## Инициализация шардирования

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

### 2. Инициализация шардов

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [{ _id: 0, host: "shard1:27018" }]
})
EOF
```

```shell
docker compose exec -T shard2 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [{ _id: 0, host: "shard2:27018" }]
})
EOF
```

### 3. Добавление шардов в кластер и включение шардирования

Коллекцию шардируем **до** наполнения данными, чтобы документы распределились
по шардам. В качестве ключа шардирования используем хешированное поле `name`.

```shell
docker compose exec -T mongos_router mongosh --port 27017 --quiet <<EOF
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27018")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { name: "hashed" })
EOF
```

### 4. Наполнение данными (1000 документов)

```shell
docker compose exec -T mongos_router mongosh --port 27017 --quiet <<EOF
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i })
EOF
```

## Проверка

### Через приложение

Откройте http://localhost:8080 — в ответе будет `mongo_topology_type: "Sharded"`,
список шардов и общее количество документов (≥ 1000).

Количество документов в коллекции:

```shell
curl --silent http://localhost:8080/helloDoc/count
```

### Напрямую по шардам

Общее количество (через роутер):

```shell
docker compose exec -T mongos_router mongosh --port 27017 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Количество в каждом шарде (сумма по шардам равна общему числу):

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

```shell
docker compose exec -T shard2 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

## Доступные эндпоинты

Swagger: http://localhost:8080/docs
