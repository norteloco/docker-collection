# Описание

Репозиторий с коллекцией популярных и не очень сборок для запуска приложений и сервисов в контейнерезированном окружении Docker.  

- [Состав](#состав)
- [Запуск](#запуск)
    - [Команды](#команды)
- [Дополнительно](#дополнительно)
- [TO DO](#to-do)

## Состав

- [MongoDB](./mongodb/)
    - Включает в себя и Mongo Express для просмотра БД через Web-интерфейс
- [Prometheus](./prometheus/)
    - Также Alertmanager и несколько популярных экспортеров: Node Exporter, Blackbox Exporter, cAdvisor.

## Запуск

0. Установка компонент Docker
    - [Docker Engine](https://docs.docker.com/engine/install/) или [Docker Desktop](https://docs.docker.com/desktop/) в зависимости от платформы.
    - [Docker Compose](https://docs.docker.com/compose/install/)

1. Склонировать этот репозиторий  
```
git clone https://github.com/norteloco/docker-collection.git
```

2. Перейти в директорию с требуемой сборкой
```
cd mongodb
```

3. Пользоваться

### Команды

- *Запуск*
```
docker compose up -d
```

- *Остановка*
```
docker compose down
```

- *Посмотреть логи*
```
docker compose logs <имя-контейнера>
```
- *Запуск отдельного контейнера*
```
docker compose up <имя-контейнера>
```

- *Остановка*
```
docker compose stop <имя-контейнера>
```


## Дополнительно

Внутри каждой сборки лежит отдельный readme-файл с описанием.

## TO DO

- Grafana
- Nginx (+Certbot)
- Graylog
- Zabbix (упрощение официальной сборки [zabbix-docker](https://github.com/zabbix/zabbix-docker))
- Portainer
- AWX
- HashiCorp Vault

