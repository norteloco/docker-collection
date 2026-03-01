# Пример для запуска MongoDB в Docker-окружении

- [Требования](#требования)
- [Описание](#описание)
    - [Параметры](#параметры)
        - [MongoDB](#mongodb)
        - [Mongo Express](#mongo-express)

## Требования

1. [Docker Engine](https://docs.docker.com/engine/install/) или [Docker Desktop](https://docs.docker.com/desktop/) в зависимости от платформы.
2. [Docker Compose](https://docs.docker.com/compose/install/)

## Описание

Данный стек позволяет запускать MongoDB используя [официальный Docker-образ](https://hub.docker.com/_/mongo) совместно с веб-интерфейсом администратора [mongo-express](https://hub.docker.com/_/mongo-express/) ([проект на GitHub](https://github.com/mongo-express/mongo-express)).

### Параметры

#### MongoDB


    image: mongo:8.2.5

Официальный образ MongoDB. Вы можете указать любую подходящую версию.  

    container_name: mongodb  
    hostname: mongodb  

Параметры, которые я указываю для удобства обращения и понимания того, какой контейнер у меня открыт в случае необходимости отладки и проверки какого-либо функционала с использованием внутренней консоли контейнера.  

    restart: always  

[Политика перезапуска](https://docs.docker.com/reference/compose-file/services/#restart) контейнера, в случае его остановки.  

    ports:  
        - 27017:27017  

Порт хоста может отличаться от стандартного порта для MongoDB, что позволит немного повысить безопасность, но в таком случае стоит не забывать этого. Также публикуемый порт можно вынести в env-file и указать в виде переменной в compose-файле.

        volumes:  
            - mongo-data:/data/db

    volumes:
        mongo-data:

Можно использовать стандартное хранилище как указано в примере или указать свой путь для хранения данных, например `- /opt/mongo/data:/data/db` или относительный `- .data/db:/data/db`

    env_file:
        - .mongo_env  

Аттрибут с указанием на файл параметров конфигурации для контейнеров. Повышает читаемость compose-файлов и позволяет хранить все переменные в одном месте.

        secrets:
          - ROOT_USERNAME
          - ROOT_PASSWORD

    secrets:
      ROOT_USERNAME:
        file: ./env_vars/.MONGO_ROOT_USER
      ROOT_PASSWORD:
        file: ./env_vars/.MONGO_ROOT_PASSWORD  

Немного улучшает безопасность хранения конфиденциальных данных. 
В параметрах достаточно указать расположение секретов в `/run/secrets` вида:  
`MONGO_INITDB_ROOT_USERNAME_FILE=/run/secrets/ROOT_USERNAME`
`MONGO_INITDB_ROOT_PASSWORD_FILE=/run/secrets/ROOT_PASSWORD`

    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 60s
      timeout: 10s
      retries: 5
      start_period: 30s

Проверка работоспособности. В зависимости от версии MongoDB может отличаться.


        networks:
            mongodb_net:
                aliases:
                    - mongo
                    - mongodb

    networks:
    mongodb_net:
        driver: bridge
        enable_ipv6: false
        ipam:
            driver: default
            config: 
                - subnet: 172.16.232.0/24

Определяем используемую сеть и указываем дополнительные имена хостов для других служб в сети. Сеть должна быть предварительно создана или также описана в `docker-compose.yaml`  

#### Mongo Express

    image: mongo-express

Официальный рабочий образ, однако на текущий момент находится в статусе **DEPRECATED**. Лучше использовать сборку из Dockerfile в [официальном репозитории](https://github.com/mongo-express/mongo-express). Самостоятельно пересоберу немного позже.

    depends_on:
      mongodb:
        condition: service_healthy

Позволяет управлять порядком запуска, чтобы `mongo-express` запускался после корректного запуска сервиса `mongodb`.

    secrets:
      - ROOT_USERNAME
      - ROOT_PASSWORD
      - BASIC_USERNAME
      - BASIC_PASSWORD

Переиспользует те же параметры для root-пользователя и добавляет возможность подключения используя учетную запись обычного пользователя.

Остальные параметры схожи с описанными в пункте [MongoDB](#mongodb).
Контейнер mongo-express не хранит внутри себя никаких данных, поэтому не требует атрибута `volume`. Можно добавить `healthcheck` при желании, стандартно по пути `/status`.