# Домашнее задание к занятию 4 «Оркестрация группой Docker контейнеров на примере Docker Compose» - `Горелов Николай`

## Задача 1

### Установка Docker и Docker Compose Plugin

Установка Docker и Docker Compose Plugin на Debian:

```bash
# Установка зависимостей
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Добавление официального GPG ключа Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Добавление репозитория Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Установка Docker и Docker Compose Plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

<!-- ### Настройка Docker Hub Mirror

Создаем файл `/etc/docker/daemon.json`:

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://daocloud.io",
    "https://c.163.com/",
    "https://registry.docker-cn.com"
  ]
}
EOF
sudo systemctl restart docker
``` -->

### Создание Dockerfile и сборка образа

Создаем `Dockerfile`:

```dockerfile
FROM nginx:1.21.1
COPY index.html /usr/share/nginx/html/index.html
```

Создаем `index.html`:

```html
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
```

Собираем образ:

```bash
docker build -t custom-nginx:1.0.0 .
```

### Публикация образа на Docker Hub

```bash
docker login
docker tag custom-nginx:1.0.0 <username>/custom-nginx:1.0.0
docker push <username>/custom-nginx:1.0.0
```

Ссылка на репозиторий: [custom-nginx](https://hub.docker.com/repository/docker/nikogorelov/custom-nginx/general)

## Задача 2

### Запуск контейнера

```bash
docker run -d --name gorelovnn-custom-nginx-t2 -p 127.0.0.1:8080:80 custom-nginx:1.0.0
docker rename gorelovnn-custom-nginx-t2 custom-nginx-t2
```

### Выполнение команды

```bash
date +"%d-%m-%Y %T.%N %Z"; sleep 0.150; docker ps; ss -tlpn | grep 127.0.0.1:8080; docker logs custom-nginx-t2 -n1; docker exec -it custom-nginx-t2 base64 /usr/share/nginx/html/index.html
```

### Проверка доступности

```bash
curl http://127.0.0.1:8080
```

![](img/virtd-04-task-02.png)

## Задача 3

### Подключение к потокам контейнера

```bash
docker attach custom-nginx-t2
# Нажимаем Ctrl+C
```

### Проверка состояния контейнера

```bash
docker ps -a
```

Контейнер остановился, потому что сигнал Ctrl+C был передан процессу nginx внутри контейнера, что привело к его завершению.

### Перезапуск контейнера

```bash
docker start custom-nginx-t2
```

### Редактирование конфигурации nginx

```bash
docker exec -it custom-nginx-t2 bash
apt update && apt install -y nano
nano /etc/nginx/conf.d/default.conf
# Меняем порт с 80 на 81
nginx -s reload
curl http://127.0.0.1:80
curl http://127.0.0.1:81
exit
```

### Проверка портов

```bash
ss -tlpn | grep 127.0.0.1:8080
docker port custom-nginx-t2
curl http://127.0.0.1:8080
```

**Проблема**: После изменения порта nginx на 81 внутри контейнера, внешний порт 8080 продолжает перенаправлять на порт 80 контейнера, который больше не используется.