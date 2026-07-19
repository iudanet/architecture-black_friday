# mongo-sharding-repl

MongoDB с шардированием и репликацией: config server из 3 нод, 2 шарда по 3 реплики,
mongos_router, pymongo-api.

## Порты

По умолчанию 8080, 27017-27039. Переопределяются через `.env` (см. `.env.example`).

## Запуск

```shell
docker compose up -d
./scripts/init-sharding.sh
```

## Шаги инициализации

Скрипт `scripts/init-sharding.sh` выполняет следующее.

1. Инициализация группы репликации config server из трёх нод:

```shell
docker compose exec -T configSrv1 mongosh --port 27017 --quiet <<EOF
rs.initiate({_id: "config_server", configsvr: true, members: [
  {_id: 0, host: "configSrv1:27017"},
  {_id: 1, host: "configSrv2:27017"},
  {_id: 2, host: "configSrv3:27017"}
]});
EOF
```

2. Инициализация групп репликации шардов, по три реплики в каждой:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({_id: "shard1", members: [
  {_id: 0, host: "shard1-1:27018"},
  {_id: 1, host: "shard1-2:27018"},
  {_id: 2, host: "shard1-3:27018"}
]});
EOF

docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
rs.initiate({_id: "shard2", members: [
  {_id: 0, host: "shard2-1:27019"},
  {_id: 1, host: "shard2-2:27019"},
  {_id: 2, host: "shard2-3:27019"}
]});
EOF
```

3. Регистрация шардов и включение шардирования коллекции:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018");
sh.addShard("shard2/shard2-1:27019,shard2-2:27019,shard2-3:27019");
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", {name: "hashed"});
EOF
```

4. Наполнение коллекции `somedb.helloDoc` 1000 документами:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({age: i, name: "ly" + i});
EOF
```

## Проверка

Общая информация, количество документов и состав шардов:

```shell
curl http://localhost:8080/
```

Количество документов в каждом шарде:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Количество реплик в группах:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.status().members.length
EOF

docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
rs.status().members.length
EOF
```

Документация API: http://localhost:8080/docs
