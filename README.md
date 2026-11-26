# dockyard

Coleção de `docker-compose` prontos para **testes e desenvolvimento local**. Cada serviço fica em
uma pasta própria, usa o mínimo de recursos que ainda permite funcionamento pleno e entra na mesma
rede Docker (`shared_net`), para que os serviços conversem entre si (por exemplo, fluxos do n8n
usando RabbitMQ, MongoDB, Postgres, MinIO etc.).

> **Uso exclusivo local.** Credenciais triviais (`admin/admin`, `guest/guest`...), alguns serviços sem
> autenticação e portas publicadas em todas as interfaces. Não exponha na internet nem use em produção.

## Serviços

| Serviço | Imagem | Host na `shared_net` | Portas (host) | Credenciais | Limites (RAM / CPU) |
|---|---|---|---|---|---|
| [n8n](docs/services/n8n.md) | `docker.n8n.io/n8nio/n8n:latest` | `n8n` | 5678 | conta criada no 1º acesso | **sem limite definido** |
| [RabbitMQ](docs/services/rabbitmq.md) | `rabbitmq:4.1-management` | `rabbitmq` | 5672 (AMQP), 15672 (UI) | `guest` / `guest` | 512 MB / 1.0 |
| [MongoDB + Atlas Search](docs/services/mongodb.md) | `mongodb/mongodb-atlas-local` | `mongodb` | 27017 | `admin` / `admin` | 2 GB / 2.0 |
| [Elasticsearch](docs/services/elasticsearch.md) | `docker.elastic.co/elasticsearch/elasticsearch:9.5.2` | `elasticsearch` | 9200 | nenhuma | 1.5 GB / 1.0 |
| [MinIO (S3)](docs/services/minio.md) | `quay.io/minio/minio` | `minio` | 9000 (API), 9001 (console) | `admin` / `admin1234` | 512 MB / 1.0 |
| [PostgreSQL](docs/services/postgres.md) | `postgres:17-alpine` | `postgres` | 5432 | `postgres` / `postgres` | 512 MB / 1.0 |
| [MariaDB](docs/services/mariadb.md) | `mariadb:11.8` | `mariadb` | 3306 | `root`/`root` e `app`/`app` (banco `app`) | 512 MB / 1.0 |
| [Redis](docs/services/redis.md) | `redis:8-alpine` | `redis` | 6379 | nenhuma | 256 MB / 0.5 |

Cada página de serviço traz como conectar (do host, de outro container e do n8n), os recursos medidos
e, principalmente, as **limitações** daquela configuração.

## Começando

```bash
# 1. Criar a rede compartilhada (uma única vez por máquina)
docker network create shared_net

# 2. Subir o(s) serviço(s) desejado(s)
cd rabbitmq && docker compose up -d
cd ../n8n && docker compose up -d
```

Dentro da rede, um serviço acessa o outro **pelo nome do host da tabela acima** (ex.: `rabbitmq:5672`),
nunca por `localhost`. Do seu computador, use `localhost:<porta>`.

Não é necessário subir tudo: suba só o que for usar. Detalhes de pré-requisitos, consumo de memória
com vários serviços ao mesmo tempo, conflitos de porta e solução de problemas em
[docs/getting-started.md](docs/getting-started.md).

## Documentação

- [docs/getting-started.md](docs/getting-started.md): pré-requisitos, rede, operação do dia a dia, recursos, segurança e troubleshooting.
- [docs/services/](docs/services/): uma página por serviço (conexão, recursos, limitações).
- [docs/creating-new-services.md](docs/creating-new-services.md): guia para **agentes e pessoas** criarem novos serviços seguindo as mesmas premissas.

## Estrutura

```
dockyard/
├── <servico>/docker-compose.yml   # um diretório por serviço (o nome da pasta é o nome do projeto Compose)
├── rabbitmq/rabbitmq.conf         # arquivos de configuração extras ficam ao lado do compose
└── docs/
    ├── getting-started.md
    ├── creating-new-services.md
    └── services/<servico>.md
```

> **Não renomeie as pastas dos serviços** depois de criar volumes: o Compose usa o nome da pasta como
> prefixo dos volumes, e renomear faz o serviço subir "vazio" (os dados antigos continuam no volume antigo).

## Validado em

macOS com Docker Desktop (VM com ~7.75 GB de RAM). Em Linux nativo ou Windows o conjunto deve funcionar,
mas **não foi testado**; os pontos que dependem da plataforma estão sinalizados nas páginas dos serviços
(em especial o Elasticsearch).
