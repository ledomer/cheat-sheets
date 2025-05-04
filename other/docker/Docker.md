### Шпаргалка по работе с Docker

**[<<Оглавление](../../TableOfContents.md)**

#### Ссылки
[Репозиторий образов](https://hub.docker.com/search?q=python)

**Основные команды Docker CLI**

```
# Запуск контейнера на основе указанного образа
docker run <Имя контейнера>:

# Показать список активных контейнеров
docker ps

# Показать все контейнеры, включая остановленные
docker ps -a

# Остановить контейнер
docker stop

# Удалить контейнер
docker rm 

# Показать список всех локальных образов
docker images

# Загрузить образ из Docker Hub
docker pull 

# Удалить локальный образ
docker rmi 
```


**Работа с контейнерами**

```
# Запустить контейнер и открыть в нём bash
docker run -it  bash

# Зайти в уже запущенный контейнер
docker exec -it  bash

# Копировать файл в контейнер
docker cp  :/путь/внутри/контейнера

# Получить информацию о контейнере
docker inspect 
```


**Создание и работа с образами**

```
# Сборка образа на основе Dockerfile
docker build -t : 

# Пометить образ новым тегом
docker tag  

# Отправка образа в Docker Hub или другой реестр
docker push /:
```


**Управление контейнерами**

```
# Запуск остановленного контейнера
docker start 

# Перезапуск контейнера
docker restart 

# Удаление нескольких контейнеров
docker rm  
```


**Работа с Dockerfile**

- `FROM` - базовый образ.
- `RUN` - выполнение команды при сборке.
- `COPY`/`ADD` - копирование файлов.
- `WORKDIR` - рабочая директория.
- `CMD` - команда по умолчанию при запуске контейнера.
- `ENTRYPOINT` - основная команда для контейнера.[4][10]

**Пример Dockerfile для Python-приложения:**
```
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "main.py"]
```


---

## Шпаргалка по работе с docker-compose

**Базовый шаблон docker-compose.yml:**
```yaml
services:
  :
    image: 
    container_name: 
    hostname: 
    restart: unless-stopped
    environment:
      TZ: "Europe/Moscow"
```


**Команды docker-compose:**
```
# Запуск сервисов в фоне
docker-compose up -d

# Остановка сервисов
docker-compose down

# Просмотр логов
docker-compose logs

# Перезапуск сервисов
docker-compose restart

# Просмотр состояния сервисов
docker-compose ps
```


---

## Полезные параметры команд

- `-d` - запуск в фоне (detached mode).
- `-p` - проброс портов (например, `-p 8080:80`).
- `-v` - монтирование тома (например, `-v $(pwd):/app`).
- `-e` - установка переменной окружения (например, `-e VAR=value`).
- `--rm` - удалить контейнер после остановки.[6][8]

---

## Работа с Docker Swarm (оркестрация)

```
# Инициализация Swarm
docker swarm init --advertise-addr 

# Создание сервиса
docker service create --name  -p 8080:80 

# Масштабирование сервиса
docker service scale =3

# Список сервисов
docker service ls

# Список узлов
docker node ls
```


---

## Быстрые советы

- Для получения справки по любой команде используйте `docker  --help`.
- Для очистки неиспользуемых ресурсов: `docker system prune`.
- Для входа в Docker Hub: `docker login`.[6][8]

---

Эта шпаргалка поможет быстро вспомнить основные команды и подходы при работе с Docker и docker-compose.

Citations:
[1] https://tproger.ru/articles/raspilit-monolit-na-mikroservisy-polnaya-wpargalka-po-rabote-s-doker
[2] https://habr.com/ru/companies/flant/articles/336654/
[3] https://devops.org.ru/docker-summary
[4] https://habr.com/ru/companies/kokocgroup/articles/802039/
[5] https://www.dmosk.ru/miniinstruktions.php?mini=docker-compose-examples
[6] https://dockerhosting.ru/blog/komandy-docker-shpargalka/
[7] https://cloud.croc.ru/upload/Docker-Cheat-Sheet-Cloud.pdf
[8] https://serverspace.ru/articles/kratkaya-shpargalka-po-docker/
[9] https://gist.github.com/rtplv/c5422078dbcf79d076ebc740fdc92eb0
[10] https://devops.org.ru/dockerfile-summary

---
Answer from Perplexity: pplx.ai/share:
https://www.perplexity.ai/search/shpargalka-po-rabote-s-docker-MQZlyAllSl2YriABqBdgbw

**[<<Оглавление](../TableOfContents.md)**