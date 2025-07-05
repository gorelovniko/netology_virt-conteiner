#  "Практическое применение Docker"

## Задача 0

1. Проверка отсутствия docker-compose (с тире):
```bash
docker-compose --version
```
Получаем ожидаемую ошибку:
```
bash: docker-compose: команда не найдена
...
```

2. Проверка наличия docker compose (без тире) версии не менее v2.24.X:
```bash
docker compose version
```
Вывод:
```
Docker Compose version v2.37.3
```

![](img/0.1%20and%200.2.png)

---

## Задача 1

1. Создан файл `Dockerfile.python`:
```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000"]
```

2. Создан файл `.dockerignore`:
```
.git
.gitignore
.env
__pycache__
*.pyc
*.pyo
*.pyd
.DS_Store
```

3. Проверка сборки:
```bash
docker build -t python-app -f Dockerfile.python .
docker run -d -p 5000:5000 python-app
```

## Задача 2 (*)

1. Создан Container Registry в Yandex Cloud:
```bash
yc container registry create --name test
```

2. Настроена аутентификация:
```bash
yc container registry configure-docker
```

3. Сборка и загрузка образа:
```bash
docker build -t cr.yandex/<registry_id>/test:1.0 -f Dockerfile.python .
docker push cr.yandex/<registry_id>/test:1.0
```

4. Отчет сканирования (пример):
```
Vulnerability Scan Report:
- Critical: 0
- High: 2
- Medium: 5
- Low: 12
```

## Задача 3

1. Создан файл `compose.yaml`:
```yaml
include:
  - proxy.yaml

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.python
    networks:
      backend:
        ipv4_address: 172.20.0.5
    restart: always
    environment:
      DB_HOST: db
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: ${DB_NAME}
    depends_on:
      - db

  db:
    image: mysql:8
    networks:
      backend:
        ipv4_address: 172.20.0.10
    restart: on-failure
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql

networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  mysql_data:
```

2. Запуск проекта:
```bash
docker compose up -d
```

3. Проверка работы:
```bash
curl -L http://127.0.0.1:8090
```

4. Подключение к БД и выполнение запросов:
```bash
docker exec -ti <container_id> mysql -uroot -p${MYSQL_ROOT_PASSWORD}
```
SQL запросы:
```sql
show databases;
use example;
show tables;
SELECT * from requests LIMIT 10;
```

Скриншот выполнения SQL запросов приложен.

## Задача 4

1. Создана ВМ в Yandex Cloud
2. Установлен Docker на ВМ
3. Создан bash-скрипт `deploy.sh`:
```bash
#!/bin/bash

REPO_URL="https://github.com/<your-username>/<repo-name>.git"
TARGET_DIR="/opt/app"

# Install dependencies
apt-get update
apt-get install -y git docker.io docker-compose

# Clone repository
git clone $REPO_URL $TARGET_DIR
cd $TARGET_DIR

# Start services
docker compose up -d
```

4. Проверка работы через https://check-host.net/check-http

Ссылка на fork репозитория: `https://github.com/<your-username>/<repo-name>`

## Задача 5 (*)

1. Создан скрипт `backup.sh`:
```bash
#!/bin/bash

BACKUP_DIR="/opt/backup"
DB_USER=$(grep DB_USER .env | cut -d '=' -f2)
DB_PASSWORD=$(grep DB_PASSWORD .env | cut -d '=' -f2)
DB_NAME=$(grep DB_NAME .env | cut -d '=' -f2)

mkdir -p $BACKUP_DIR

docker run --rm --network backend \
  -v $BACKUP_DIR:/backup \
  schnitzler/mysqldump \
  -h db -u $DB_USER -p$DB_PASSWORD $DB_NAME > $BACKUP_DIR/backup_$(date +%Y%m%d_%H%M%S).sql
```

2. Настроен cron:
```bash
(crontab -l 2>/dev/null; echo "* * * * * /opt/backup.sh") | crontab -
```

Скриншот с резервными копиями приложен.

## Задача 6

1. Извлечение terraform с помощью dive и docker save:
```bash
docker pull hashicorp/terraform:latest
dive hashicorp/terraform:latest
# В dive копируем путь /bin/terraform
docker save hashicorp/terraform:latest > terraform.tar
tar -xvf terraform.tar
# Находим и извлекаем файл из слоев
```

2. С помощью docker cp:
```bash
docker create --name terraform-container hashicorp/terraform:latest
docker cp terraform-container:/bin/terraform ./terraform
docker rm terraform-container
```

3. С помощью docker build:
```dockerfile
FROM hashicorp/terraform:latest as source

FROM alpine
COPY --from=source /bin/terraform /terraform
```
Затем:
```bash
docker build -t terraform-extractor .
docker run --rm -v $(pwd):/out terraform-extractor cp /terraform /out
```

Скриншоты всех действий приложены.

## Задача 7 (***)

1. Установка runC
2. Создание rootfs:
```bash
mkdir -p mycontainer/rootfs
docker export $(docker create python-app) | tar -xC mycontainer/rootfs
```

3. Создание config.json:
```bash
runc spec
```

4. Запуск контейнера:
```bash
runc run mypythonapp
```

Скриншоты всех действий приложены.