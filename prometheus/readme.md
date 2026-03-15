# Prometheus в Docker-окружении

- [Требования](#требования)
- [Описание](#описание)
    - [Сервисы](#сервисы)
        - [Prometheus](#prometheus)
        - [Node Exporter](#node-exporter)
        - [Cadvisor](#cadvisor)
        - [Blackbox Exporter](#blackbox-exporter)
        - [Alertmanager](#alertmanager)
    - [Конфигурации](#конфигурации)
## Требования

## Описание

### Сервисы

Список сервисов и описание их параметров.

#### Prometheus
    image: prom/prometheus:v3.10.0

Официальный образ Prometheus без изменений. Вы можете указать любую подходящую версию.

    container_name: prometheus
    hostname: prometheus

Параметры, которые я указываю для удобства обращения и понимания того, какой контейнер у меня открыт в случае необходимости отладки и проверки какого-либо функционала с использованием внутренней консоли контейнера.

    restart: unless-stopped

[Политика перезапуска](https://docs.docker.com/reference/compose-file/services/#restart) контейнера, в случае его остановки.  

    ports:
      - 9090:9090

Порт хоста может отличаться от стандартного порта для Prometheus, что позволит немного повысить безопасность, однако в данном примере он остается стандартным.

    volumes:
      - ./prom_env/prometheus:/etc/prometheus
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./prom_data/prometheus:/prometheus/data


Путь ко хранилищу данных в данном случае указан локальным. Связано это с тем, что [внутри образа](https://github.com/prometheus/prometheus/blob/main/Dockerfile) и, соответственно, контейнера работа системы осуществляется от имени пользователя _nobody_. Соответственно необходимо либо запускаться от пользователя _root_ чтобы использовать data-volume или же использовать локальное хранилище. 

    env_file:
        - .prom_env  

Атрибут с указанием на файл параметров конфигурации для контейнеров. Повышает читаемость compose-файлов и позволяет хранить все переменные в одном месте.

    command:
      - '--config.file=/etc/prometheus/prometheus.yaml'
      - '--storage.tsdb.path=/prometheus/data'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=30GB'
      - '--storage.tsdb.wal-compression'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'  

`--config.file=/etc/prometheus/prometheus.yaml`  
Это явное указание, где хранится файл конфигурации. Такой путь - стандартное расположение, но для перестраховки лучше его указать.  

Информация о параметрах флагов `storage` описана в [документации](https://prometheus.io/docs/prometheus/latest/storage/#operational-aspects). 

`--web.enable-lifecycle`  
`--web.enable-admin-api`  
Информация об этих флагах также есть в [документации](https://prometheus.io/docs/operating/security/#prometheus).  
Говоря в кратце, эти флаги позволяют управлять перезапуском и остановкой Prometheus, а также получить доступ к административному HTTP API для, например, удаления значений временных рядов. В продуктивном контуре не рекомендуется включать. В данном примере используется для демонстрации.  

    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s 

Проверка работоспособности сервиса.

    networks:
      prom_net:
        aliases:
          - prometheus
          - prom

Указание на используемую сервисом сеть и внутренние имена (алиасы). Далее будет полное описание параметров сети.

#### Node Exporter

_Чтобы не дублировать информацию, далее будут описаны только отличающиеся или важные для сервисов параметры._

    image: prom/node-exporter

Официальный образ Node Exporter без изменений. Вы можете указать любую подходящую версию.

    ports:
      - 9100:9100

Порт сервиса. Внешний порт можно указать по желанию.

    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

Чтобы параметры метрик брались не из контейнера, а с хоста, мы пробрасываем нашу [виртуальную файловую систему интерфейса внутренних систем ядра](https://www.kernel.org/doc/html/latest/filesystems/proc.html) в контейнер, а также разделы файловой системы.

    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/var/lib/docker/aufs|rootfs/run/docker/netns)($|/)'

Далее мы как раз указываем откуда экспортеру забирать метрики контейнера, а т.к. они прокидываются с хоста, то получаем мы их с хоста.

`--collector.filesystem.mount-points-exclude`  
В данном флаге просто указываем информацию о каких файловых системах мы не хотим собирать.

    networks:
      prom_net:
        aliases:
          - prometheus-node-exporter
          - prom-node-exporter
          - node-exporter

Указание на используемую сервисом сеть и внутренние имена (алиасы). Далее будет полное описание параметров сети.

#### Cadvisor
    image: gcr.io/cadvisor/cadvisor:v0.55.1  

Официальный образ cAdvisor без изменений. Вы можете указать любую подходящую версию.

    ports:
      - 8080:8080

Порт сервиса. Внешний порт можно указать по желанию.

    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

По аналогии с Node Exporter "прокидываем" системные директории и директорию с Docker (или другой системой контейнеризации).

    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s   

Проверка работоспособности сервиса.

     networks:
      prom_net:
        aliases:
          - prometheus-cadvisor
          - prom-cadvisor
          - cadvisor

Указание на используемую сервисом сеть и внутренние имена (алиасы). Далее будет полное описание параметров сети.

#### Blackbox Exporter
    image: prom/blackbox-exporter

Официальный образ Blackbox Exporter без изменений. Вы можете указать любую подходящую версию.

    ports:
      - 9115:9115

Порт сервиса. Внешний порт можно указать по желанию.

    volumes:
      - ./prom_env/blackbox:/etc/blackbox

Указываем путь ко хранилищу данных. Продолжим использовать локальный относительный путь. 

    command: 
      - '--config.file=/etc/blackbox/blackbox.yaml'

Указываем, где находится файл конфигурации.

    networks:
      prom_net:
        aliases:
          - prometheus-blackbox-exporter
          - prometheus-blackbox
          - prom-blackbox
          - blackbox

Указание на используемую сервисом сеть и внутренние имена (алиасы). Далее будет полное описание параметров сети.

#### Alertmanager

    image: prom/alertmanager

Официальный образ Alertmanager без изменений. Вы можете указать любую подходящую версию.

    ports:
      - 9093:9093

Порт сервиса. Внешний порт можно указать по желанию.


    volumes:
      - ./prom_env/alertmanager:/etc/alertmanager
      - ./prom_data/alertmanager:/alertmanager/data

Для удобства единообразия хранения данных тут путь ко хранилищу данных также указан локальным и относительным. 
Один для файла конфигурации, второй для данных сервиса.

    secrets:
      - BOT_TOKEN
      - CHAT_ID

Секреты, которые будут использоваться в контейнере сервиса.

    command:
      - '--config.file=/etc/alertmanager/alertmanager.yaml'
      - '--storage.path=/alertmanager/data'

Указываем параметры флагов для расположения файла конфигурации и данных сервиса.

    networks:
      prom_net:
        aliases:
          - prometheus-alertmanager
          - prom-alertmanager
          - alertmanager

Указание на используемую сервисом сеть и внутренние имена (алиасы). Далее будет полное описание параметров сети.

#### Секреты

secrets:
  BOT_TOKEN:
    file: ./env_vars/.BOT_TOKEN
  CHAT_ID:
    file: ./env_vars/.CHAT_ID

Указываем расположение файлов [секретов](https://docs.docker.com/reference/compose-file/secrets/), в которых будут храниться наши данных об идентификаторе чата и токене бота.  

_Внимание! Как можно заметить, их нет в проекте т.к. они добавлены в файл .gitignore. Вам необходимо создать их и заполнить самостоятельно._

#### Сеть

    networks:
    prom_net:
        driver: bridge
        enable_ipv6: false
        ipam:
        driver: default
        config: 
        - subnet: 172.16.232.0/24

Параметры сети Docker. В них описывается название сети, драйвер и подсеть. Вся подробная информация есть в официальной [документации](https://docs.docker.com/reference/compose-file/networks/).

### Конфигурации

Параметры конфигураций распологаются по пути `./prom_env/<имя сервиса>/`. Используется YAML-формат. Приложенные конфигурации сервисов представляют из себя базовый пример того, что и как можно поставить на мониторинг. Вся подробная и точная информация есть в официальной документации.  

- [Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Blackbox](https://github.com/prometheus/blackbox_exporter/blob/master/CONFIGURATION.md)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/configuration/)

Отдельно дополню, что в конфигурации Alertmanager в параметрах `receivers` для Telegram можно указать просто `bot_token` и `chat_id` как переменные, что не очень безопасно, либо же указать их как `bot_token_file` и `chat_id_file`, которые можно перенаправить на ранее созданные [секреты](#секреты).  

Пример из конфигурации в проекте: 

    receivers:
    - name: 'telegram'
        telegram_configs:
        - bot_token_file: /run/secrets/BOT_TOKEN
        chat_id_file: /run/secrets/CHAT_ID
        message_thread_id: 4
        message: '{{ template "telegram.default" .}}'
        parse_mode: "HTML"

Также у Prometheus имеются правила (rules) для оповещений. Они также считываются Alertmanager и согласно этим правилам оповещения и отправляются к соответствующему получателю. Шаблон отправляемых сообщений хранится в файлах `*.tmpl` в конфигурационной директории Alertmanager.