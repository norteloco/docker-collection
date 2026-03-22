# Grafana в Docker-окружении

- [Требования](#требования)
- [Описание](#описание)
    - [Сервисы](#сервисы)
        - [Grafana](#grafana)
    - [Конфигурация](#конфигурация)


## Требования

1. [Docker Engine](https://docs.docker.com/engine/install/) или [Docker Desktop](https://docs.docker.com/desktop/) в зависимости от платформы.
2. [Docker Compose](https://docs.docker.com/compose/install/)

## Описание

### Сервисы

#### Grafana

    image: grafana/grafana:12.4

Официальный образ Grafana без изменений. Вы можете указать любую подходящую версию.
Также можно использовать Grafana OSS, в таком случае образ будет `grafana/grafana-oss:<version>`

    container_name: grafana
    hostname: grafana

Параметры, которые я указываю для удобства обращения и понимания того, какой контейнер у меня открыт в случае необходимости отладки и проверки какого-либо функционала с использованием внутренней консоли контейнера.

    restart: unless-stopped

[Политика перезапуска](https://docs.docker.com/reference/compose-file/services/#restart) контейнера, в случае его остановки.  

    env_file:
      - .grf_env
Атрибут с указанием на файл параметров конфигурации для контейнеров сервисов. Повышает читаемость compose-файлов и позволяет хранить все переменные в одном месте.

    ports:
      - 3000:3000

Порт хоста может отличаться от стандартного порта для Grafana, что позволит немного повысить безопасность, однако в данном примере он остается стандартным.\

    volumes:
      - grafana-data:/var/lib/grafana

    volumes:
        grafana-data:

Указание на [хранилище данных](https://docs.docker.com/reference/compose-file/services/#volumes). Можно использовать стандартное расположение `grafana-data`, которое ссылается на `/var/lib/grafana` или же указать локальное расположение, например `./grf_data:/var/lib/grafana`  

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

[Проверка](https://docs.docker.com/reference/compose-file/services/#healthcheck) работоспособности сервиса.

    secrets:
      - ADMIN_USER
      - ADMIN_PASSWORD

    secrets:
        ADMIN_USER:
            file: ./env_vars/.ADMIN_USER
        ADMIN_PASSWORD:
            file: ./env_vars/.ADMIN_PASSWORD

[Секреты](https://docs.docker.com/reference/compose-file/services/#secrets), которые будут использоваться в контейнере сервиса. Далее в конфигурации также указано расположение этих секретов.


    networks:
      grf_net:
        aliases:
          - grafana-server
          - grafana

Указание на используемую сервисом [сеть](https://docs.docker.com/reference/compose-file/services/#networks) и внутренние имена (алиасы).  

    networks:
        grf_net:
            driver: bridge
            enable_ipv6: false
            ipam:
                driver: default
                config: 
                - subnet: 172.16.232.0/24

Непосредственно сама [настройка сети](https://docs.docker.com/reference/compose-file/networks/).


### Конфигурация

Конфигурацию можно настроить двумя разными способами:  
1. Через файл `grafana.ini`, который в дальнейшем прокинуть через _volume_
2. Через .env-файл, как в данном примере.

Я предпочитаю в Docker-compose использовать именно .env-файлы для настройки параметров, т.к. они удобнее, легче читаются и их всегда можно дополнить в процессе CI/CD, для внесения срочных изменений или тестирования функционала.

Если вы знакомы с файлом конфигурации Grafana, то многие параметры могут выглядеть знакомо.  
Подробная информация о составлении переменных окружения для Grafana есть в [официальной документации](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#override-configuration-with-environment-variables).  

Кратко, то переменные окружения составляются по принициу `GF_<SECTION NAME>_<KEY>`, где:  
`<SECTION NAME>` - назвение секции параметров из файла конфигурации, например `[server]` или `[security]`, а `<KEY>` - соответствующее имя параметра из этой секции.  

Таким образом 

    [security]
    admin_user = admin

превращается в  

    GF_SECURITY_ADMIN_USER=admin

Также приписка `__FILE` позволяет записать параметры окружения в определенный файл, как у меня это делано для логина и пароля администратора, которые в самом `docker-compose.yaml` указаны как секреты.

    GF_SECURITY_ADMIN_USER__FILE=/run/secrets/ADMIN_USER
    GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/ADMIN_PASSWORD

Путь `/run/secrets` - это стандартный путь расположения секретов в контейнера.

Больше информации в [официальной документации Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/).