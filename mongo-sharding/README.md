# mongo-sharding

MongoDB с шардированием: configSrv, shard1, shard2, mongos_router, pymongo-api.

## Порты

По умолчанию 8080, 27017-27020. Переопределяются через `.env` (см. `.env.example`).

## Запуск

```shell
docker compose up -d
./scripts/init-sharding.sh
```

## Шаги инициализации

Скрипт `scripts/init-sharding.sh` выполняет следующее.

1. Инициализация реплики config server:

```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({_id: "config_server", configsvr: true, members: [{_id: 0, host: "configSrv:27017"}]});
EOF
```

2. Инициализация реплик шардов:

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({_id: "shard1", members: [{_id: 0, host: "shard1:27018"}]});
EOF

docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({_id: "shard2", members: [{_id: 0, host: "shard2:27019"}]});
EOF
```

3. Регистрация шардов и включение шардирования коллекции:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018");
sh.addShard("shard2/shard2:27019");
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

Общая информация и количество документов:

```shell
curl http://localhost:8080/
```

Количество документов в каждом шарде:

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Документация API: http://localhost:8080/docs
