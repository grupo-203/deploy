# mv2img-deploy

Repositório de configuração de deploy para a plataforma **mv2img** — sistema distribuído de conversão de vídeos em frames/imagens com geração de arquivos ZIP para download.

## Visão Geral

O **mv2img** é uma plataforma de microserviços que permite o upload de vídeos, extração de frames em intervalos configuráveis e entrega dos resultados empacotados em ZIP. Este repositório contém as configurações Docker Compose para orquestrar todos os serviços em ambiente de produção.

### Arquitetura

```
Usuário → Load API (upload) → RabbitMQ → Workers de Processamento
                                              ├── request-job
                                              ├── extract-frames-job
                                              ├── generate-zip-job
                                              ├── publish-job
                                              └── dead-letter-job
                                                         ↓
                                              Delivery API (download)
```

### Serviços

| Serviço | Tecnologia | Porta | Descrição |
|---|---|---|---|
| `auth-api` | Spring Boot | 8080 | Autenticação e geração de tokens JWT |
| `load-api` | Spring Boot | 8081 | Recebimento e enfileiramento de vídeos |
| `delivery-api` | Python / FastAPI | 8000 | Entrega dos ZIPs processados |
| `rabbitmq` | RabbitMQ 3 | 5672 / 15672 | Broker de mensagens entre serviços |
| `redis` | Redis 7 | 6379 | Cache e gerenciamento de sessões |
| `auth-db` | PostgreSQL 16 | — | Banco de dados do serviço de autenticação |
| `load-db` | PostgreSQL 15 | — | Banco de dados do serviço de carga |
| `delivery-db` | PostgreSQL 15 | — | Banco de dados do serviço de entrega |
| `watchtower` | Watchtower | — | Atualização automática de imagens Docker |

### Workers de Processamento

Todos utilizam a imagem `jefferson233/mv2img-process:latest` e se comunicam via RabbitMQ:

| Worker | Fila | Função |
|---|---|---|
| `process-request-job` | `carregamentos.video.request` | Processa requisições de upload |
| `process-extract-frames-job` | `extract-frames` | Extrai frames dos vídeos |
| `process-generate-zip-job` | `generate-zip` | Gera o arquivo ZIP com os frames |
| `process-publish-job` | `publish` | Publica o resultado para entrega |
| `process-dead-letter-job` | `dead-letter` | Trata jobs com falha |

## Estrutura do Repositório

```
mv2img-deploy/
├── LICENSE
└── compose/
    ├── .env                      # Define os arquivos Compose ativos
    ├── docker-compose.yml        # Watchtower + rede compartilhada
    ├── docker-compose.auth.yml   # Serviço de autenticação + banco
    ├── docker-compose.delivery.yml # Serviço de entrega + Redis + banco
    ├── docker-compose.load.yml   # Serviço de carga + RabbitMQ + banco
    ├── docker-compose.process.yml # Workers de processamento (5 jobs)
    └── mv2img-process.env        # Variáveis de ambiente dos workers
```

## Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) 20.10+
- [Docker Compose](https://docs.docker.com/compose/install/) 1.29+ (ou Docker Compose V2)
- Credenciais AWS S3 (opcional — para armazenamento em nuvem)
- Servidor SMTP (opcional — para notificações por e-mail)

## Configuração

### 1. Variáveis dos Workers (`mv2img-process.env`)

Edite o arquivo `compose/mv2img-process.env` e preencha os valores obrigatórios:

```env
# URLs dos serviços internos (obrigatório)
AUTH_SERVICE_URL=http://auth-api:8080
LOAD_SERVICE_URL=http://load-api:8081

# Credenciais do worker no serviço de autenticação
CLIENT_ID=video-processor
CLIENT_SECRET=password

# Intervalo de extração de frames (padrão: 1 frame por segundo)
SECONDS_PER_FRAME=1

# Caminhos de armazenamento interno (não alterar em geral)
VIDEO_PATH=/data/videos
FRAMES_PATH=/data/frames
ZIP_PATH=/data/zip

# Tentativas máximas em caso de falha
ERROR_MAX_RETRIES=3

# RabbitMQ
RABBITMQ_CONNECTION=hostname=rabbitmq;port=5672;username=rabbit;password=rabbit

# E-mail (opcional)
MAIL_HOST=
MAIL_PORT=25
MAIL_FROM=
MAIL_USERNAME=
MAIL_PASSWORD=
```

### 2. Credenciais de Produção

Antes de subir em produção, altere os seguintes valores nos arquivos Compose:

| Arquivo | Variável | Valor padrão (mudar!) |
|---|---|---|
| `docker-compose.auth.yml` | `JWT_SECRET` | `your-secret-key-change-this-in-production-minimum-256-bits` |
| `docker-compose.auth.yml` | `POSTGRES_PASSWORD` | `auth_password` |
| `docker-compose.load.yml` | `POSTGRES_PASSWORD` | `postgres` |
| `docker-compose.delivery.yml` | `POSTGRES_PASSWORD` | `delivery_pass` |
| `docker-compose.load.yml` | `RABBITMQ_DEFAULT_PASS` | `rabbit` |

### 3. Armazenamento S3 (opcional)

No arquivo `docker-compose.delivery.yml`, configure as variáveis de ambiente do serviço `delivery-api`:

```yaml
AWS_ACCESS_KEY_ID: sua-access-key
AWS_SECRET_ACCESS_KEY: sua-secret-key
S3_BUCKET_NAME: nome-do-bucket
```

## Deploy

### Iniciar todos os serviços

```bash
cd compose
docker compose up -d
```

### Verificar status

```bash
docker compose ps
```

### Acompanhar logs

```bash
# Todos os serviços
docker compose logs -f

# Serviço específico
docker compose logs -f delivery-api
```

### Parar os serviços

```bash
# Mantém os volumes (dados persistidos)
docker compose down

# Remove também os volumes e dados
docker compose down -v
```

### Gerenciar serviços individualmente

```bash
# Iniciar apenas um serviço
docker compose up -d auth-api

# Reiniciar um serviço
docker compose restart load-api
```

## Acessos

| Serviço | URL | Credenciais padrão |
|---|---|---|
| Auth API | http://localhost:8080 | — |
| Load API | http://localhost:8081 | — |
| Delivery API | http://localhost:8000 | — |
| RabbitMQ Management | http://localhost:15672 | `rabbit` / `rabbit` |

## Imagens Docker

Todas as imagens são obtidas do Docker Hub sob a organização `lerrana`:

- `jefferson233/mv2img-auth:latest`
- `jefferson233/mv2img-load:latest`
- `jefferson233/mv2img-delivery:latest`
- `jefferson233/mv2img-process:latest`

O **Watchtower** monitora e atualiza automaticamente essas imagens a cada 30 segundos.

## Rede

Todos os serviços se comunicam através da rede bridge `mv2img-network`. Apenas as portas explicitamente mapeadas ficam expostas ao host.

## Volumes Persistentes

| Volume | Usado por | Conteúdo |
|---|---|---|
| `auth-db-data` | `auth-db` | Dados do PostgreSQL de autenticação |
| `load-db-data` | `load-db` | Dados do PostgreSQL de carga |
| `delivery-db-data` | `delivery-db` | Dados do PostgreSQL de entrega |
| `redis-data` | `redis` | Dados do Redis |
| `load-data` | `load-api`, `delivery-api`, workers | Vídeos, frames e ZIPs gerados |

## Licença

MIT License — Copyright (c) 2026 [grupo-203](https://github.com/grupo-203)
