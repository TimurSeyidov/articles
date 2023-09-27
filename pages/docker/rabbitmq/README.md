# Запуск RabbitMQ в Docker

Создаём файл **docker-compose.yml** со следующим содержимым:
```yml
version: "3.9"
services:
  rabbitmq:
    container_name: rabbitmq
    image: rabbitmq:management-alpine
    hostname: rabbitmq
    restart: always
    environment:
      - RABBITMQ_DEFAULT_USER=user
      - RABBITMQ_DEFAULT_PASS=password
      - RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS=-rabbit log_levels [{connection,error},{default,error}] disk_free_limit 2147483648
    ports:
      - 15672:15672
      - 5672:5672
    volumes:
      - ./rabbitmq:/var/lib/rabbitmq
```

Немного поясним:
- **hostname** - имя сервера
- **restart: always** - даёт указание Docker автоматически перезагружать сервис в случае его внезапной остановки
- **RABBITMQ_DEFAULT_USER** - имя пользователя для авторизации в web-панели
- **RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS=-rabbit disk_free_limit** - говорит о том, что RabbitMQ перейдёт в защиту и перестанет писать в стейт при свободном месте менее 48 мбит, что очень мало — порог пробивается в 90% случаев. А если RabbitMQ попытается записать на диск, где нет места, это с 90% вероятностью уничтожит стейт без возможности восстановления. В примере переопределяем на **2Gb** (**2147483648 bit**)
- **RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS=log_levels [{connection,error},{default,error}]** - уровень логирования **error**. Подробнее [тут](https://www.rabbitmq.com/logging.html#log-message-categories)
- **volumes -> rabbitmq** - устанавливаем, чтобы стейт хранился на сервере, а не только внутри контейнера
- **Port 15672** - порт для web-панели
- **Port 5672** - AMQP

Запускаем:
```bash
docker-compose up -d
```

Перейдите по ссылке [http://localhost:15672](http://localhost:15672) в браузере:

![web-панель](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/rabbitmq/assets/web.png?raw=true)