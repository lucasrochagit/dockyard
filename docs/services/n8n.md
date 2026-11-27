# n8n

Plataforma de automação de fluxos. É o "centro" desta rede: os demais serviços existem para serem usados a partir dele.

| | |
|---|---|
| Compose | [`n8n/docker-compose.yml`](../../n8n/docker-compose.yml) |
| Imagem | `docker.n8n.io/n8nio/n8n:latest` |
| Container / host na rede | `n8n` / `n8n` |
| Porta | `5678` → http://localhost:5678 |
| Credenciais | conta de *owner* criada no primeiro acesso à interface |
| Volume | `n8n_data` (`/home/node/.n8n`) |
| Recursos | **sem `mem_limit`/`cpus` definidos** |

## Subir

```bash
cd n8n && docker compose up -d
```

## Conectar aos outros serviços (credenciais dentro do n8n)

Use sempre o **nome do host na `shared_net`** e a porta interna, nunca `localhost`:

| Node do n8n | Host / URL | Porta | Usuário / senha | Detalhes |
|---|---|---|---|---|
| RabbitMQ / RabbitMQ Trigger | `rabbitmq` | 5672 | `guest` / `guest` | vhost `/` |
| MongoDB | `mongodb` | 27017 | `admin` / `admin` | `authSource=admin`; veja [mongodb](mongodb.md) |
| Postgres / Postgres Trigger | `postgres` | 5432 | `postgres` / `postgres` | banco `postgres`, SSL desativado; veja [postgres](postgres.md) |
| MySQL (serve para MariaDB) | `mariadb` | 3306 | `app` / `app` | banco `app` |
| MySQL | `mysql` | 3306 | `app` / `app` | banco `app`; veja [mysql](mysql.md) |
| Redis | `redis` | 6379 | sem usuário/senha | banco `0` |
| AWS (S3, SQS, SNS) via LocalStack | `http://localstack:4566` | 4566 | `test` / `test` | região `us-east-1`; ligar *Custom Endpoints*; veja [localstack](localstack.md) |
| Elasticsearch | `http://elasticsearch:9200` | 9200 | sem autenticação | veja a nota abaixo |

Também dá para usar o node **HTTP Request** contra `http://elasticsearch:9200`.

## Limitações e pontos de atenção

- **Sem limite de recursos.** É o único serviço do repositório sem `mem_limit`/`cpus`; pode consumir o que a VM do Docker oferecer.
- **Imagem `latest` sem versão fixa.** Um `docker compose pull` pode trazer uma versão nova do n8n, com mudanças de comportamento. Fixe uma tag se precisar de reprodutibilidade.
- **Banco interno é o padrão do n8n** (SQLite dentro do volume). Não há Postgres externo nem *queue mode* (que exigiria Redis + Postgres) configurados.
- **URLs de webhook apontam para `localhost`.** `N8N_HOST=localhost`, `N8N_PORT=5678` e `N8N_PROTOCOL=http` fazem o n8n montar URLs de webhook como `http://localhost:5678/...`. Serviços externos (GitHub, Stripe...) não alcançam isso; seria preciso um túnel e a variável `WEBHOOK_URL`. Chamadas vindas da própria máquina ou de outros containers (`http://n8n:5678/webhook/...`) não dependem disso, mas esse caminho não foi testado aqui.
- **Perder o volume `n8n_data` perde tudo:** workflows, execuções e credenciais. As credenciais são cifradas com uma chave guardada no mesmo volume, então um backup só do banco não as recupera.
- **Fuso horário** `America/Recife` (`GENERIC_TIMEZONE` e `TZ`): os nodes de agenda (Schedule/Cron) usam esse fuso.
- `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` e `N8N_RUNNERS_ENABLED` estão ativos (padrões recomendados pela documentação do n8n).
- Não há `healthcheck` declarado.

## O que foi e o que não foi validado

- Validado: o container do n8n alcança `rabbitmq:5672`, `mongodb:27017` por TCP e tem saída para a internet (DNS e conexões externas), então também conecta em bancos externos (ex.: MongoDB Atlas na nuvem; o IP público da sua máquina precisa estar liberado no firewall do banco).
- **Não testado dentro do n8n:** os nodes em si (Postgres Trigger, MySQL, AWS/LocalStack, Redis, Elasticsearch). Os serviços foram testados com clientes próprios na `shared_net`. O node *Elasticsearch* do n8n pode exigir usuário/senha no cadastro da credencial; como a segurança do Elasticsearch está desativada, o servidor ignora qualquer valor.
