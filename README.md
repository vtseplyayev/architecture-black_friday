# pymongo-api: шардирование, репликация и кеширование MongoDB

Проектная работа по архитектуре онлайн-магазина «Мобильный мир».
Приложение `pymongo-api` (образ `kazhem/pymongo_api:1.0.0`) работает с MongoDB,
развёрнутой как шардированный кластер с репликацией, и использует Redis для
кеширования.

## Структура репозитория

| Директория             | Что внутри                                                        |
|------------------------|------------------------------------------------------------------|
| `mongo-sharding`       | Задание 2 — шардирование (2 шарда)                                |
| `mongo-sharding-repl`  | Задание 3 — шардирование + репликация (по 3 реплики на шард)      |
| `sharding-repl-cache`  | **Задания 2 + 3 + 4** — шардирование + репликация + кеш (Redis)   |
| `docs/`                | Схемы draw.io (задания 1, 5, 6)                                   |

**Финальная реализация для проверки — `sharding-repl-cache`.**

Итоговая схема архитектуры (задания 1, 5, 6) — [docs/06-cdn.drawio](docs/06-cdn.drawio).
Промежуточные схемы: [02-sharding](docs/02-sharding.drawio),
[03-replication](docs/03-replication.drawio), [04-caching](docs/04-caching.drawio),
[05-gateway-consul](docs/05-gateway-consul.drawio).

## Как запустить финальный стенд (sharding-repl-cache)

Все команды выполняются из директории `sharding-repl-cache`:

```shell
cd sharding-repl-cache
```

### 1. Запуск сервисов

```shell
docker compose up -d
```

Проверить, что все сервисы поднялись:

```shell
docker compose ps
```

Должны быть запущены: `pymongo_api`, `redis`, `mongos_router`, `configSrv`,
`shard1-1/2/3`, `shard2-1/2/3`.

### 2. Инициализация кластера и наполнение данными

Выполните шаги по порядку (подробности — в
[sharding-repl-cache/README.md](sharding-repl-cache/README.md)).

Инициализация config server:

```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({ _id: "config_server", configsvr: true, members: [{ _id: 0, host: "configSrv:27017" }] })
EOF
```

Репликация shard1 и shard2 (по 3 узла):

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({ _id: "shard1", members: [
  { _id: 0, host: "shard1-1:27018" },
  { _id: 1, host: "shard1-2:27018" },
  { _id: 2, host: "shard1-3:27018" }
] })
EOF

docker compose exec -T shard2-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({ _id: "shard2", members: [
  { _id: 0, host: "shard2-1:27018" },
  { _id: 1, host: "shard2-2:27018" },
  { _id: 2, host: "shard2-3:27018" }
] })
EOF
```

Подождите ~10 секунд, пока выберутся PRIMARY, затем добавьте шарды,
включите шардирование и наполните коллекцию:

```shell
sleep 10

docker compose exec -T mongos_router mongosh --port 27017 --quiet <<EOF
sh.addShard("shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018")
sh.addShard("shard2/shard2-1:27018,shard2-2:27018,shard2-3:27018")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { name: "hashed" })
EOF

docker compose exec -T mongos_router mongosh --port 27017 --quiet <<EOF
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i })
EOF
```

### 3. Проверка

Откройте в браузере http://localhost:8080 — приложение вернёт JSON с информацией
о MongoDB. Ожидаемое:

- `"mongo_topology_type": "Sharded"`;
- `"collections": { "helloDoc": { "documents_count": 1000 } }` (≥ 1000);
- в `"shards"` — оба шарда, каждый со списком из 3 реплик;
- `"cache_enabled": true`.

Количество документов в каждом шарде:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

docker compose exec -T shard2-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Проверка кеша — повторный вызов `/helloDoc/users` выполняется < 100 мс:

```shell
curl --silent -o /dev/null -w "1-й запрос: %{time_total}s\n" http://localhost:8080/helloDoc/users
curl --silent -o /dev/null -w "2-й запрос: %{time_total}s\n" http://localhost:8080/helloDoc/users
```

## Доступные эндпоинты

Swagger: http://localhost:8080/docs
