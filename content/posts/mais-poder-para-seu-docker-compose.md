+++ 
draft = false
date = 2022-09-26T11:45:17-03:00
title = "Mais Poder no Seu Docker Compose"
description = "Deixando seu docker-compose.yaml mais inteligente"
slug = "mais-poder-no-seu-docker-compose"
authors = ["Marcelo Lino"]
tags = ["docker", "docker-compose", "devops"]
categories = ["devops", "docker", "programação"]
externalLink = ""
series = ["Docker"]
+++

Já podemos dar como certo que o docker está em um momento ou outro na vida de um programador, e muitas vezes nos utilizamos de arquivos `docker-compose.yaml` para executarmos localmente nossas aplicações e suas dependências e algumas vezes até em produção.

Por vezes temos que executar coisas mais complexas como executar uma migração de banco de dados antes da nossa aplicação executar, garantir que o banco de dados já esteja pronto para receber conexões, executar um subconjunto de dependências e assim por diante. Em muitas das minhas experiências vi esses requisitos serem executados através de scripts shell que nem sempre são familiares a todos os membros da equipe o que sempre gera uma estranheza e medo de extender tais funcionalidades, pensando nisso resolvi deixar mais evidentes algumas funcionalidades do `Docker Compose` que por muitas vezes são ignoradas.

## [Healthcheck](https://docs.docker.com/compose/compose-file/#healthcheck)

A especificação do Docker Compose conta com uma forma de _healthcheck_ que podemos utilizar para garantir que nossa aplicação não só está rodando, mas como ela está operacional, essa checagem é útil quando queremos garantir que um serviço de ser executado somente após outro serviço estar totalmente operacional.

```yaml
version: '3'

services:
  db:
    container_name: db
    image: postgres:alpine
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_DB=postgres
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Definindo o banco de dados com esse _healthcheck_ quando adicionarmos a definição da nossa aplicação podemos dizer que ao depender do banco ela só inicie após o serviço dependente estar de pé e saudável.

```yaml
version: '3'

services:
  app:
    container_name: app
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    command: "gunicorn -b 0.0.0.0:8000 -k gevent sample.main:create_app() --access-logfile=- --preload -w 4"
    depends_on:
      db:
        condition: service_health
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthcheck"]
      interval: 5s
      timeout: 30s
      retries: 5

  db:
    container_name: db
    image: postgres:alpine
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_DB=postgres
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

## [Profiles](https://docs.docker.com/compose/compose-file/#profiles-1)

As vezes nos deparamos com arquivos `docker-compose.yaml` extensos e com várias definições de serviço, porém ao desenvolver alguma funcionalidade necessitamos somente de um subconjunto desses serviços afim de testarmos nosso código, para isso temos os _profiles_

```yaml
version: '3'

services:
  app:
    container_name: app
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    command: "gunicorn -b 0.0.0.0:8000 -k gevent sample.main:create_app() --access-logfile=- --preload -w 4"
    depends_on:
      db:
        condition: service_healthy
      profiles:
      - web
      - full
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthcheck"]
      interval: 5s
      timeout: 30s
      retries: 5

  db:
    container_name: db
    image: postgres:alpine
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_DB=postgres
    profiles:
      - dependencies
      - web
      - full
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Com essa definição podemos utilizar o comando `docker-compose --profile dependencies up` para iniciar somente a nossa base de dados.

## Extensão

A mesma base de código muitas vezes pode ser utilizada para executar serviços em diferentes contextos, uma aplicação web, um serviço assíncrono de tarefas, uma execução única de linha de comando. Sendo assim é interessante para nós desenvolvedores definirmos uma base comum e somente personalizarmos para cada caso de uso.

Para atingir tal objetivo no `Docker Compose` temos a opção de usar um feature da própria

### Ancora YAML

A especificação do YAML permite que criemos [ancoras](https://yaml.org/spec/1.2.2/#3222-anchors-and-aliases) para reaproveitamento de definições, assim podemos criar uma base comum para um serviço para depois especializarmos.

```yaml
version: '3'

x-app: &app
  build:
      context: .
      dockerfile: Dockerfile
  depends_on:
    db:
      condition: service_healthy
    queue:
      condition: service_healthy

services:
  app:
    <<: *app
    container_name: app
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    command: "gunicorn -b 0.0.0.0:8000 -k gevent sample.main:create_app() --access-logfile=- --preload -w 4"
    profiles:
      - web
      - full
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthcheck"]
      interval: 5s
      timeout: 30s
      retries: 5
  
  background:
    <<: *app
    container_name: background
    command: "celery -A sample.main:celery_app worker"
    environment:
      - CELERY_RESULT_BACKEND=redis://cache:6379/0
      - CELERY_BROKER_URL=amqp://guest:guest@queue:5672//
    profiles:
      - background
      - full

  db:
    container_name: db
    image: postgres:alpine
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_DB=postgres
    profiles:
      - dependencies
      - full
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  queue:
    container_name: queue
    image: rabbitmq:3-management-alpine
    profiles:
      - dependencies
      - full
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 30s
      timeout: 30s
      retries: 3
  
  cache:
    container_name: cache
    image: redis:alpine
    profiles:
      - dependencies
      - full
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 30s
      retries: 3

volumes:
  db_data:
```

Essas são algumas formas de deixar seu `Docker Compose` um pouco mais dinâmico e inteligente.

[Fonte](https://github.com/Mdslino/advanced-docker-compose-usage)
