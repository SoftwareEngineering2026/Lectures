# Базовые практики

## 1. Выбирайте подходящий базовый образ

**Зачем:**  
Базовый образ влияет на размер контейнера, безопасность, скорость сборки и совместимость.

**Плохо:**

```dockerfile
FROM ubuntu:latest
```

Почему хуже:

- образ может быть слишком тяжелым;
- внутри может быть много лишнего.

**Лучше:**

```dockerfile
FROM python:3.12-slim
```

Почему лучше:

- образ уже подготовлен для Python;
- `slim` уменьшает размер;
- версия зафиксирована.

**Совет:**  
Берите официальный образ приложения или языка, если он есть:

- `node:20-alpine` или `node:20-slim`
- `python:3.12-slim`
- `nginx:stable-alpine`.

---

## 2. Избегайте использования `latest` без необходимости

**Зачем:**  
Сборка должна быть воспроизводимой. Сегодня `latest` один, завтра другой.

**Плохо:**

```dockerfile
FROM node:latest
```

**Лучше:**

```dockerfile
FROM node:20.11.1-alpine
```

Или хотя бы:

```dockerfile
FROM node:20-alpine
```

**Когда это особенно важно:**  
В production и CI/CD.

---

## 3. Используйте `.dockerignore`

Docker отправляет в build context все файлы из текущей директории. Лишние файлы замедляют сборку и могут случайно попасть в образ.

**Плохо:**  
Нет `.dockerignore`, и в контейнер переносится:

- `.git`
- `node_modules`
- логи
- `.env`
- локальные артефакты сборки.

**Лучше:**

```dockerignore
.git
node_modules
__pycache__
*.log
.env
dist
build
```

**Пример:**  
Если вы собираете Node.js-приложение, локальный `node_modules` почти никогда не нужен внутри image.

---

## 4. Копируйте только нужные файлы

**Зачем:**  
Чем меньше копируется в образ, тем быстрее сборка и меньше шанс сломать кеш.

**Плохо:**

```dockerfile
COPY . .
RUN npm install
```

Почему хуже:

- любое изменение в проекте сбрасывает кеш слоя;
- можно случайно скопировать лишнее.

**Лучше:**

```dockerfile
COPY package*.json ./
RUN npm ci
COPY . .
```

**Что это дает:**  
Если меняется только исходный код, зависимости не пересобираются.

---

## 5. Ставьте зависимости до копирования всего проекта

**Зачем:**  
Это улучшает кеширование слоев.

**Плохо:**

```dockerfile
COPY . .
RUN pip install -r requirements.txt
```

**Лучше:**

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

**Почему лучше:**  
Если код изменился, а `requirements.txt` нет, слой с зависимостями возьмется из кеша.

---

## 6. Используйте понятную рабочую директорию через `WORKDIR`

**Зачем:**  
Dockerfile становится читаемее, а команды — короче и надежнее.

**Плохо:**

```dockerfile
COPY . /app
RUN cd /app && npm install
CMD ["node", "/app/server.js"]
```

**Лучше:**

```dockerfile
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

---

## 7. Предпочитайте exec-форму для `CMD` и `ENTRYPOINT`

**Зачем:**  
Так сигналы корректно передаются процессу, особенно в Kubernetes и Docker Compose.

**Плохо:**

```dockerfile
CMD python app.py
```

Это shell form.

**Лучше:**

```dockerfile
CMD ["python", "app.py"]
```

**Почему лучше:**  
Процесс запускается напрямую, без промежуточной shell-обертки.

---

## 8. Делайте образ минимальным

**Зачем:**  
Меньше размер — быстрее доставка, меньше поверхность атаки.

**Плохо:**

```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3 python3-pip curl vim
```

**Лучше:**

```dockerfile
FROM python:3.12-slim
```

И устанавливать только то, что реально нужно приложению.

---

## 9. Объединяйте связанные команды `RUN`

**Зачем:**  
Каждая инструкция `RUN` создает новый слой.

**Плохо:**

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean
```

**Лучше:**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**Почему лучше:**  
Меньше слоев, меньше мусора в итоговом image.

---

## 10. Чистите временные файлы после установки пакетов

**Зачем:**  
Чтобы не раздувать образ.

**Плохо:**

```dockerfile
RUN apt-get update && apt-get install -y curl
```

**Лучше:**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**Для Python:**

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

---

## 11. Используйте `EXPOSE` как документацию, а не как механизм публикации порта

**Зачем:**  
`EXPOSE` помогает понять, на каком порту работает приложение.

**Пример:**

```dockerfile
EXPOSE 8080
```

**Важно:**  
Порт наружу все равно публикуется при запуске:

```bash
docker run -p 8080:8080 myapp
```

---

## 12. Держите Dockerfile простым и предсказуемым

**Зачем:**  
Для новичков это особенно важно: проще отлаживать и сопровождать.

**Хуже:**

```dockerfile
RUN if [ "$ENV" = "prod" ]; then do_something; else do_other; fi
```

**Лучше:**  
Либо разные Dockerfile под разные сценарии, либо build args с понятной логикой.

---

# Продвинутые практики

## 13. Используйте multistage build

**Зачем:**  
Позволяет отделить этап сборки от итогового runtime-образа. В финальный образ попадает только результат, без компиляторов и dev-зависимостей.

**Плохо:**

```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o app .
CMD ["./app"]
```

Почему хуже:

- финальный образ содержит весь toolchain Go;
- образ тяжелее и менее безопасен.

**Лучше:**

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY . .
RUN go mod download
RUN CGO_ENABLED=0 GOOS=linux go build -o /bin/app .

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /bin/app /app/app
CMD ["/app/app"]
```

**Что выигрываем:**

- меньше размер;
- меньше лишних бинарников;
- чище production-образ.

---

## 14. Максимально используйте кеширование слоев

**Зачем:**  
Быстрая сборка особенно важна в CI.

**Пример для Node.js:**

```dockerfile
FROM node:20-alpine
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

CMD ["node", "dist/index.js"]
```

**Идея:**  
Редко меняющиеся файлы — выше в Dockerfile. Часто меняющиеся — ниже.

---

## 15. Используйте BuildKit cache mounts там, где это уместно

**Зачем:**  
Ускоряет повторные сборки для пакетных менеджеров.

**Пример:**

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

**Полезно для:**

- `pip`
- `npm`
- `apt`
- `go mod`

Это уже более продвинутый прием, особенно полезный в CI/CD.

---

## 16. Запускайте приложение не от root

**Зачем:**  
Это одна из важнейших практик безопасности.

**Плохо:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]
```

По умолчанию процесс часто идет от `root`.

**Лучше:**

```dockerfile
FROM node:20-alpine
WORKDIR /app

RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -D appuser

COPY package*.json ./
RUN npm ci

COPY . .
RUN chown -R appuser:appgroup /app

USER appuser
CMD ["node", "server.js"]
```

**Что это дает:**  
При компрометации контейнера у злоумышленника будет меньше прав.

---

## 17. Минимизируйте количество установленных системных пакетов

**Зачем:**  
Каждый пакет — это размер и потенциальные уязвимости.

**Плохо:**

```dockerfile
RUN apt-get update && apt-get install -y \
    curl wget vim git net-tools build-essential
```

**Лучше:**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**Правило:**  
Не ставьте утилиты "на всякий случай" в production-образ.

---

## 18. Разделяйте build-time и runtime конфигурацию

**Зачем:**  
Секреты и окружение не должны "запекаться" в образ без крайней необходимости.

**Плохо:**

```dockerfile
ENV DB_PASSWORD=mysecretpassword
```

**Почему плохо:**  
Секрет оказывается в image history и конфигурации образа.

**Лучше:**  
Передавать секреты при запуске:

```bash
docker run -e DB_PASSWORD=... myapp
```

Или использовать менеджер секретов / orchestrator secrets.

**Для build-time настроек:**

```dockerfile
ARG APP_VERSION=1.0.0
ENV APP_VERSION=$APP_VERSION
```

**Разница:**

- `ARG` — только на этапе сборки;
- `ENV` — доступно в контейнере во время выполнения.

---

## 19. Не храните секреты в Dockerfile

**Зачем:**  
Это отдельное и очень важное правило безопасности.

**Плохо:**

```dockerfile
ARG NPM_TOKEN=supersecret
RUN npm config set //registry.npmjs.org/:_authToken=$NPM_TOKEN
```

**Лучше:**  
Использовать секреты сборки, если доступны, например через BuildKit.

Идея:

```dockerfile
RUN --mount=type=secret,id=npm_token ...
```

Смысл в том, чтобы секрет не оставался в слоях образа.

---

## 20. Используйте конкретные теги и при необходимости digest pinning

**Зачем:**  
Даже тег `python:3.12-slim` со временем может указывать на обновленный образ.

**Лучше для строгой воспроизводимости:**

```dockerfile
FROM python:3.12-slim@sha256:...
```

**Когда полезно:**

- production
- regulated environments
- строгий CI/CD

**Минус:**  
Нужно вручную обновлять digest.

---

## 21. Учитывайте сигнализацию и graceful shutdown

**Зачем:**  
Контейнер должен корректно завершаться.

**Плохо:**

```dockerfile
CMD ["sh", "-c", "python app.py"]
```

**Лучше:**

```dockerfile
CMD ["python", "app.py"]
```

**Еще лучше:**  
Убедиться, что само приложение корректно обрабатывает `SIGTERM`.

---

## 22. Добавляйте `HEALTHCHECK`, если это оправдано

**Зачем:**  
Оркестратор сможет понимать, живо ли приложение.

**Пример:**

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1
```

**Когда полезно:**  
Для веб-приложений, API, сервисов с health endpoint.

**Когда не стоит:**  
Если проверка дорогая, ненадежная или уже есть внешний механизм health monitoring.

---

## 23. Разделяйте dev и prod образы

**Зачем:**  
Разработке нужны hot reload, debug tools и dev dependencies. Продакшену — минимальность и безопасность.

**Хуже:**  
Один образ для всего.

**Лучше:**

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS dev
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS prod
RUN npm ci --omit=dev
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

---

## 24. Старайтесь делать образ immutable

**Зачем:**  
Контейнер должен быть предсказуемым: собрали один раз, запускаем везде одинаково.

**Антипаттерн:**  
При старте контейнера устанавливать пакеты или менять файловую систему.

**Плохо:**

```dockerfile
CMD ["sh", "-c", "apt-get update && apt-get install -y curl && python app.py"]
```

**Лучше:**  
Все необходимое должно быть уже в image на этапе build.

---

## 25. Добавляйте метаданные через `LABEL`

**Зачем:**  
Удобно для поддержки, аудита и CI/CD.

**Пример:**

```dockerfile
LABEL org.opencontainers.image.title="my-app" \
      org.opencontainers.image.description="Example web service" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.source="https://example.com/repo"
```

---

## 26. Проверяйте итоговый размер и содержимое образа

**Зачем:**  
Иногда Dockerfile выглядит хорошо, но образ все равно получился огромным.

**Что полезно проверять:**

- размер образа;
- список слоев;
- наличие лишних файлов;
- наличие секретов.

**Практика:**  
Регулярно смотреть:

- history образа;
- состав файловой системы;
- уязвимости базового образа.

---

## 27. Используйте rootless-совместимый подход, где возможно

**Зачем:**  
Это улучшает безопасность окружения выполнения.

**Пример идеи:**

- не писать в системные каталоги;
- использовать непривилегированный порт внутри контейнера, например `8080`, а не `80`;
- запускать процесс от обычного пользователя.

**Плохо:**

```dockerfile
EXPOSE 80
USER root
```

**Лучше:**

```dockerfile
EXPOSE 8080
USER appuser
```

---

# Общие рекомендации по поддержке и документированию Dockerfile

## 28. Пишите Dockerfile так, чтобы его можно было читать через полгода

**Зачем:**  
Поддержка важнее "хитрости".

**Хорошо:**

- логичный порядок инструкций;
- минимум магии;
- понятные имена стадий.

**Пример:**

```dockerfile
FROM node:20-alpine AS dependencies
...
FROM node:20-alpine AS builder
...
FROM node:20-alpine AS runtime
...
```

Вместо безымянных стадий.

---

## 29. Комментируйте неочевидные решения

**Зачем:**  
Не все нужно комментировать, но нестандартные решения — обязательно.

**Хорошо:**

```dockerfile
# Нужен libc6-compat для совместимости с native module X
RUN apk add --no-cache libc6-compat
```

**Плохо:**  
Оставить странную зависимость без объяснения.

---

## 30. Держите Dockerfile в связке с README

**Зачем:**  
Dockerfile отвечает на вопрос "как собирать", README — "как использовать".

**Что полезно задокументировать:**

- команду сборки;
- команду запуска;
- переменные окружения;
- нужные порты;
- volume, если есть;
- health endpoint.

**Пример в README:**

```bash
docker build -t myapp .
docker run -p 8080:8080 --env-file .env myapp
```

---

## 31. Храните один Dockerfile — одну понятную цель

**Зачем:**  
Если Dockerfile пытается одновременно быть dev, test, prod, debug и builder без структуры, он быстро становится хрупким.

**Лучше:**

- multistage;
- четко названные stages;
- при необходимости отдельные Dockerfile:
  - `Dockerfile`
  - `Dockerfile.dev`
  - `Dockerfile.test`

---

## 32. Проверяйте Dockerfile в CI

**Зачем:**  
Чтобы не узнать о проблемах только на проде.

**Что стоит делать в CI:**

- сборка образа;
- запуск тестов;
- проверка линтером;
- проверка на уязвимости;
- smoke test контейнера.

---

# Краткий шаблон хорошего Dockerfile

Вот универсальный базовый пример:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Устанавливаем только необходимые системные пакеты
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Сначала зависимости для лучшего кеширования
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Затем код приложения
COPY . .

# Создаем непривилегированного пользователя
RUN groupadd -r -g 1001 appgroup && \
    useradd -r -u 1001 -g appgroup -d /app -s /sbin/nologin appuser && \
    chown -R appuser:appgroup /app
USER appuser

EXPOSE 8000

CMD ["python", "app.py"]
```

---

# Практическое правило для запоминания

Хороший Dockerfile обычно:

- **предсказуемый**;
- **минимальный**;
- **кешируемый**;
- **безопасный**;
- **понятный для команды**.
