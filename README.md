# Проектная работа 4 спринта

Онлайн-магазин «Мобильный мир»: pymongo-api, MongoDB с шардированием и репликацией,
кеширование в Redis.

## Состав репозитория

- `api_app`, `compose.yaml`, `scripts` - исходный стенд PoC: один инстанс приложения и одна нода MongoDB
- `mongo-sharding` - задание 2: шардирование, 2 шарда
- `mongo-sharding-repl` - задание 3: шардирование и репликация, 2 шарда по 3 реплики
- `sharding-repl-cache` - задания 2-4: шардирование, репликация, кеширование в Redis
- `scheme-final.drawio`, `scheme-final.png` - итоговая схема по заданиям 1, 5 и 6

Схема каждого этапа лежит в соответствующей директории в файле `scheme.drawio`.

## Запуск финальной реализации

Проверяемая директория - `sharding-repl-cache`.

```shell
cd sharding-repl-cache
docker compose up -d
./scripts/init-sharding.sh
```

Скрипт инициализирует группы репликации config server и обоих шардов, регистрирует шарды
в mongos, включает шардирование коллекции `somedb.helloDoc` и наполняет её 1000 документами.

## Проверка

Приложение доступно на http://localhost:8080, документация API - на http://localhost:8080/docs.

```shell
curl http://localhost:8080/
```

В ответе `collections.helloDoc.documents_count` равен 1000, в `shards` перечислены оба шарда
с тремя репликами, `cache_enabled` равен `true`.

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

Кеширование, второй и последующие вызовы выполняются менее 100 мс:

```shell
for i in 1 2 3; do
  curl -s -o /dev/null -w "%{time_total}s\n" http://localhost:8080/helloDoc/users
done
```

## Порты

Порты публикуются через переменные окружения со значениями по умолчанию (8080 для приложения,
27017-27039 для MongoDB, 6379 для Redis). Если порты заняты, скопируйте `.env.example` в `.env`
и укажите свои значения.
