![Схема кэширования](image.png)

Для запуска проекта необходимо выполнить

```bash
docker-compose up -d
```

Если экземпляры mongodb не настраивались:
1) Подключиться к серверу конфигурации 
```bash
docker exec -it configSrv mongosh --port 27017
```
И выполнить инициализацию репликационного сета для конфигурационного сервера в MongoDB

```bash
rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
```
Отключиться от контейнера
```
exit();
```

2) Инициализировать шарды и реплики

Подключиться к первому шарду
```bash
docker exec -it shard1_primary mongosh --port 27018
```

Этот код инициализирует репликационный сет с именем shard1_primary, состоящий из одного узла с адресом shard1_primary:27018, который будет использоваться как шард в шардированной системе MongoDB.

```bash
rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1_primary:27018" },
        { _id : 1, host : "shard1_secondary:27021" }
      ]
    }
);
```

Отключиться от контейнера
```
exit();
```

Подключиться ко второму шарду
```bash
docker exec -it shard2_primary mongosh --port 27019
```

инициализировать репликационный сет для shard2

```bash
rs.initiate(
    {
      _id : "shard2",
      members: [
       { _id : 0, host : "shard2_primary:27019" },
       { _id : 1, host : "shard2_secondary:27022" }
      ]
    }
  );
```
Отключиться от контейнера
```
exit();
```

3) Проинициализировать роутер

Подключиться к роутеру
```bash
docker exec -it mongos_router mongosh --port 27020
```
Добавить в роутер шарды

```bash
sh.addShard( "shard1/shard1_primary:27018");
sh.addShard( "shard2/shard2_primary:27019");

sh.enableSharding("somedb");
sh.shardCollection("somedb.users", { "name" : "hashed" } )
```
Вставить тестовые данные
```
use somedb
for(var i = 0; i < 1000; i++) db.users.insert({age:i, name:"ly"+i})
```

Проверить, что данные вставились 
```
db.users.countDocuments()
```

Отключиться от контейнера
```
exit();
```

Инициализировать redis кластер
```
docker exec -it redis_1 redis-cli --cluster create \
  173.17.0.2:6379 173.17.0.3:6379 173.17.0.4:6379 \
  173.17.0.5:6379 173.17.0.6:6379 173.17.0.7:6379 \
  --cluster-replicas 1
```
Получить информацию о созданном кластере
```
docker exec -it redis_1  redis-cli cluster nodes
```

На данном этапе у меня не получилось запустить app.py с кластером редиса, при запуске код зависает. После пары дней ресерча так и не понял в чем проблема, возможно проблема в самой библиотеке redis.cluster.