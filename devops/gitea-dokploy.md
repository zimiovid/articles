# Gitea + Actions Runner в Dokploy: Полное руководство

Пошаговая инструкция по развёртыванию Gitea с CI/CD и Container Registry в Dokploy.

## Требования

- Сервер с установленным **Dokploy** (https://dokploy.com)
- Домен с настроенным DNS (например, `gitea.example.com`)
- SSH-доступ к серверу для начальной настройки

## Что получим в итоге

- Gitea — свой Git-сервер с веб-интерфейсом
- Actions Runner — выполнение CI/CD пайплайнов (аналог GitHub Actions)
- Container Registry — хранение Docker-образов
- Автоматическая сборка и публикация образов при push

---

## Часть 1: Подготовка сервера

Эти команды выполняются на сервере через SSH (не в Dokploy).

### 1.1 Создание сети Docker

На сервере выполни:

```bash
docker network create gitea-net
```

Эта сеть нужна чтобы контейнеры Gitea, PostgreSQL, Runner и job-контейнеры могли общаться между собой.

### 1.2 Создание директории для конфига

```bash
sudo mkdir -p /opt/gitea
```

### 1.3 Создание конфига Runner

```bash
sudo nano /opt/gitea/runner-config.yaml
```

Вставь содержимое:

```yaml
log:
  level: info

runner:
  file: .runner
  capacity: 1
  timeout: 3h
  insecure: false
  fetch_timeout: 5s
  fetch_interval: 2s
  labels:
    - "ubuntu-latest:docker://catthehacker/ubuntu:act-latest"
    - "ubuntu-22.04:docker://catthehacker/ubuntu:act-22.04"

cache:
  enabled: true

container:
  network: "gitea-net"
  privileged: true
  options: ""
  docker_host: ""
  force_pull: false
```

Ключевые параметры:
- `container.network: "gitea-net"` — job-контейнеры подключаются к сети и видят Gitea
- `container.privileged: true` — разрешает docker build внутри контейнеров
- `labels` — какие образы использовать для `runs-on: ubuntu-latest`

---

## Часть 2: Создание сервиса в Dokploy

### 2.1 Создание Compose-сервиса

1. Открой Dokploy в браузере
2. Перейди в нужный проект (или создай новый)
3. Нажми **+ Create Service** → **Compose**
4. Дай имя сервису (например, `gitea`)
5. В поле Compose вставь:

```yaml
version: "3.8"
services:
  gitea:
    image: docker.gitea.com/gitea:1.25.4
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      USER_UID: 1000
      USER_GID: 1000
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: postgres:5432
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: ${GITEA_DB_PASSWORD}
      GITEA__server__ROOT_URL: https://gitea.example.com/
      GITEA__server__SSH_DOMAIN: gitea.example.com
      GITEA__server__LFS_START_SERVER: "true"
      GITEA__service__DISABLE_REGISTRATION: "true"
      GITEA__service__ALLOW_ONLY_EXTERNAL_REGISTRATION: "true"
      GITEA__service__SHOW_REGISTRATION_BUTTON: "false"
      GITEA__actions__ENABLED: "true"
      GITEA__packages__ENABLED: "true"
      GITEA__repository__DEFAULT_PRIVATE: "private"
      GITEA__repository__DEFAULT_BRANCH: master
      GITEA__session__PROVIDER: db
      GITEA__session__COOKIE_SECURE: "true"
    volumes:
      - gitea-data:/data
    expose:
      - "3000"
      - "22"
    networks:
      gitea-net: {}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/healthz"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  postgres:
    image: postgres:14
    restart: unless-stopped
    environment:
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: ${GITEA_DB_PASSWORD}
      POSTGRES_DB: gitea
    volumes:
      - pg-data:/var/lib/postgresql/data
    expose:
      - "5432"
    networks:
      gitea-net: {}

  runner:
    image: gitea/act_runner:latest
    restart: always
    depends_on:
      gitea:
        condition: service_healthy
        restart: true
    environment:
      CONFIG_FILE: /config.yaml
      GITEA_INSTANCE_URL: http://gitea:3000
      GITEA_RUNNER_REGISTRATION_TOKEN: ${RUNNER_TOKEN}
      GITEA_RUNNER_NAME: dokploy-runner
    volumes:
      - /opt/gitea/runner-config.yaml:/config.yaml:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - runner-data:/data
    networks:
      gitea-net: {}

volumes:
  gitea-data:
  pg-data:
  runner-data:

networks:
  gitea-net:
    external: true
    name: gitea-net
```

### 2.2 Настройка переменных окружения

В Dokploy перейди во вкладку **Environment**:

1. Нажми **Add Variable**
2. Добавь переменные:

| Переменная | Значение |
|------------|----------|
| `GITEA_DB_PASSWORD` | Надёжный пароль для PostgreSQL |
| `RUNNER_TOKEN` | Пока оставь пустым, заполним позже |

3. Нажми **Save**

### 2.3 Настройка домена

В Dokploy перейди во вкладку **Domains**:

1. Нажми **Add Domain**
2. Заполни:
   - **Host**: `gitea.example.com` (твой домен)
   - **Container Port**: `3000`
   - **HTTPS**: включи
   - **Service Name**: `gitea` (имя сервиса из compose)
3. Нажми **Save**

### 2.4 Первый деплой

1. Нажми **Deploy** в правом верхнем углу
2. Дождись завершения деплоя
3. Runner упадёт с ошибкой — это нормально, токена ещё нет

---

## Часть 3: Настройка Gitea

### 3.1 Первоначальная настройка

Открой `https://gitea.example.com/` и пройди мастер установки:
- База данных уже настроена через environment
- Создай администратора

### 3.2 Получение токена для Runner

1. Войди как администратор
2. Перейди: **Site Administration** → **Actions** → **Runners**
3. Нажми **Create new Runner**
4. Скопируй **Registration Token**

### 3.3 Добавление токена в Dokploy

1. Вернись в Dokploy → твой сервис → вкладка **Environment**
2. Найди переменную `RUNNER_TOKEN`
3. Вставь скопированный токен
4. Нажми **Save**
5. Нажми **Redeploy**

### 3.4 Проверка Runner

После деплоя проверь:
- В Gitea: **Site Administration** → **Actions** → **Runners** — должен появиться `dokploy-runner` со статусом Online
- На сервере: `docker logs <runner-container>` — должно быть "Runner registered successfully"

---

## Часть 4: Настройка Container Registry

### 4.1 Создание токена для CI

Gitea Actions token по умолчанию не имеет прав на packages. Нужен Personal Access Token.

1. Перейди: **Профиль** → **Настройки** → **Приложения**
2. Создай новый токен:
   - Имя: `ci-packages`
   - Права: **package** → **Чтение и запись**
3. Скопируй токен (показывается один раз!)

### 4.2 Добавление секрета в репозиторий

1. Открой репозиторий → **Настройки** → **Действия** → **Секреты**
2. Добавь секрет:
   - Имя: `REGISTRY_TOKEN`
   - Значение: токен из шага 4.1

---

## Часть 5: Создание CI/CD пайплайна

### 5.1 Структура проекта

```
my-project/
├── .gitea/
│   └── workflows/
│       └── build.yaml
├── Dockerfile
└── ... остальные файлы
```

### 5.2 Workflow файл

Создай `.gitea/workflows/build.yaml`:

```yaml
name: Build and Push

on:
  push:
    branches: [master]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to registry
        run: echo "${{ secrets.REGISTRY_TOKEN }}" | docker login gitea.example.com -u ${{ gitea.actor }} --password-stdin
          
      - name: Build
        run: docker build -t gitea.example.com/org/repo:${{ gitea.sha }} .
        
      - name: Push
        run: docker push gitea.example.com/org/repo:${{ gitea.sha }}
```

Замени:
- `gitea.example.com` — твой домен
- `org/repo` — организация/репозиторий

### 5.3 Пример Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

### 5.4 Запуск

```bash
git add .
git commit -m "Add CI/CD"
git push
```

Перейди в репозиторий → **Действия** — увидишь запущенный workflow.

---

## Часть 6: Дополнительные возможности

### 6.1 Тегирование образов

Добавь тег `latest` и версию:

```yaml
      - name: Build and tag
        run: |
          docker build -t gitea.example.com/org/repo:${{ gitea.sha }} .
          docker tag gitea.example.com/org/repo:${{ gitea.sha }} gitea.example.com/org/repo:latest
          
      - name: Push
        run: |
          docker push gitea.example.com/org/repo:${{ gitea.sha }}
          docker push gitea.example.com/org/repo:latest
```

### 6.2 Запуск тестов

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      # ... сборка только после успешных тестов
```

### 6.3 Деплой после сборки

```yaml
      - name: Deploy
        run: |
          curl -X POST "https://dokploy.example.com/api/deploy" \
            -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" \
            -d '{"appId": "your-app-id"}'
```

---

## Устранение проблем

### Runner не регистрируется

```bash
# Проверь логи
docker logs <runner-container>

# Убедись что сеть существует
docker network ls | grep gitea-net

# Проверь доступность gitea из runner
docker exec <runner-container> wget -qO- http://gitea:3000/api/healthz
```

### Job не может сделать docker build

Проверь что в `/opt/gitea/runner-config.yaml`:
- `container.privileged: true`
- `container.network: "gitea-net"`

После изменения конфига:
```bash
docker restart <runner-container>
```

### Ошибка "unauthorized" при push образа

1. Проверь что секрет `REGISTRY_TOKEN` добавлен в репозиторий
2. Проверь что токен имеет права `write:package`
3. Проверь правильность домена в `docker login`

### Duplicate mount point

В `runner-config.yaml` убери `-v /var/run/docker.sock:/var/run/docker.sock` из `container.options` — он монтируется автоматически.

---

## Полезные команды

```bash
# Статус всех контейнеров
docker ps | grep -E "gitea|runner|postgres"

# Логи runner
docker logs -f <runner-container>

# Логи job-контейнера (пока выполняется)
docker logs -f GITEA-ACTIONS-TASK-*

# Перерегистрация runner
docker stop <runner-container>
docker rm <runner-container>
docker volume rm <runner-volume>
# Затем redeploy в Dokploy

# Проверка сети
docker network inspect gitea-net
```

---

## Итог

Теперь у тебя есть:
- ✅ Gitea с веб-интерфейсом
- ✅ CI/CD на базе Actions
- ✅ Приватный Container Registry
- ✅ Автоматическая сборка при push

Каждый push в master автоматически:
1. Собирает Docker-образ
2. Пушит в твой registry
3. Готов к деплою