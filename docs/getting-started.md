# Primeiros passos

## Pré-requisitos

- Docker com **Compose v2** (comando `docker compose`, sem hífen). Validado com Docker Desktop no macOS.
- RAM suficiente para o que você pretende subir (veja [Recursos](#recursos)).
- Acesso à internet na primeira subida: as imagens são baixadas (a do MongoDB Atlas Local tem ~1.4 GB).

## 1. Criar a rede compartilhada

Todos os composes declaram a rede como `external: true`, então ela precisa existir **antes** do primeiro
`docker compose up`:

```bash
docker network inspect shared_net >/dev/null 2>&1 || docker network create shared_net
```

Sem ela, o Compose falha com `network shared_net declared as external, but could not be found`.

## 2. Operação do dia a dia

Cada serviço é independente: entre na pasta e use o Compose normalmente.

```bash
cd postgres
docker compose up -d        # sobe (e baixa a imagem, se preciso)
docker compose ps           # estado e saúde
docker compose logs -f      # logs
docker compose stop         # para, mantendo container e dados
docker compose down         # remove o container, MANTÉM os volumes (dados)
docker compose down -v      # remove o container E os volumes (apaga os dados)
```

- **Ordem de subida:** livre. Não há dependência entre composes de pastas diferentes; só a `shared_net` precisa existir.
- **Dados:** ficam em volumes nomeados do Docker, **na sua máquina**. O repositório guarda apenas a configuração, não os dados.
- **Nome do volume** = `<pasta>_<volume>` (ex.: `postgres_postgres_data`). Por isso não se renomeiam as pastas.
- Credenciais e parâmetros de inicialização (ex.: `POSTGRES_PASSWORD`, `MARIADB_ROOT_PASSWORD`, `RABBITMQ_DEFAULT_USER`)
  **só valem na primeira inicialização**, com o volume vazio. Para mudá-los depois é preciso `docker compose down -v` (apaga os dados) ou alterar pelo próprio serviço.

## 3. Como os serviços se enxergam

| De onde | Como acessar o serviço | Exemplo |
|---|---|---|
| Outro container na `shared_net` (n8n, sua API) | **nome do host** + porta **interna** | `postgres:5432`, `rabbitmq:5672`, `minio:9000` |
| Seu computador (psql, Compass, navegador) | `localhost` + porta **publicada** | `localhost:5432`, `http://localhost:15672` |

Regras que evitam a maioria dos problemas:

- Dentro de um container, `localhost` é **o próprio container**. No n8n, nunca use `localhost` para falar com outro serviço.
- O nome DNS é o **nome do serviço/hostname** (coluna "Host" do README), não o `container_name`. Os dois resolvem na rede, mas a documentação usa sempre o nome curto.
- Se publicar a porta em outro número (`"3307:3306"`), isso só muda o acesso **do host**; entre containers continua a porta interna.
- Para sua própria API entrar na rede, adicione `networks: [shared_net]` (com `external: true`) ao compose dela.

## Recursos

Limites definidos em cada compose (`mem_limit` / `cpus`) e consumo **medido** com `docker stats --no-stream`:

| Serviço | Limite RAM | CPU | Consumo medido | Observação |
|---|---|---|---|---|
| n8n | **sem limite** | sem limite | não medido | único serviço sem limite, veja [n8n](services/n8n.md) |
| RabbitMQ | 512 MB | 1.0 | não medido | alarme de memória em 40% do limite |
| MongoDB Atlas Local | 2 GB | 2.0 | ~380 MB logo após subir; ~660 MB com índice de Search criado | 2 GB/2 CPUs é o mínimo oficial |
| Elasticsearch | 1.5 GB | 1.0 | ~850 MB parado; ~850 MB após indexar 51 docs | heap fixo em 512 MB |
| MinIO | 512 MB | 1.0 | ~96 MB | |
| PostgreSQL | 512 MB | 1.0 | ~19 MB | |
| MariaDB | 512 MB | 1.0 | ~110 MB | |
| Redis | 256 MB | 0.5 | ~12 MB | `maxmemory` de 128 MB |

- A soma dos **limites** (sem o n8n) é ~5.8 GB. O consumo real em repouso é bem menor, mas com tudo ligado
  ao mesmo tempo e sob carga você pode encostar na memória da VM do Docker Desktop. **Suba só o que for usar.**
- Ajuste a memória da VM em Docker Desktop → Settings → Resources. MongoDB Atlas Local + Elasticsearch juntos pedem, na prática, ≥ 4 GB livres.
- Se um container é morto por falta de memória, ele sai com código **137**. Confirme com
  `docker inspect <container> --format '{{.State.OOMKilled}}'`.
- Espaço em disco: nenhum volume tem quota. Eles crescem com os dados dentro do disco virtual do Docker Desktop
  (Settings → Resources → Disk image size). O volume do MongoDB já ocupa ~210 MB vazio.

## Portas publicadas (conflitos)

| Porta | Serviço |
|---|---|
| 5678 | n8n |
| 5672 / 15672 | RabbitMQ (AMQP / UI) |
| 27017 | MongoDB |
| 9200 | Elasticsearch |
| 9000 / 9001 | MinIO (API / console) |
| 5432 | PostgreSQL |
| 3306 | MariaDB (conflita com um MySQL local) |
| 6379 | Redis (conflita com um Redis local) |

Erro `port is already allocated`: algo na sua máquina já usa a porta. Troque só o lado esquerdo do mapeamento
(`"3307:3306"`) e use `localhost:3307` do host.

## Segurança

Estas configurações são para **máquina de desenvolvimento**:

- Credenciais fixas e triviais estão no próprio repositório. Nunca coloque segredos reais aqui.
- Elasticsearch e Redis **não têm autenticação**; MinIO, Postgres, MariaDB, MongoDB e RabbitMQ usam senhas óbvias
  (o `guest` do RabbitMQ foi liberado para conexões fora de `localhost`).
- As portas são publicadas como `"porta:porta"`, ou seja, **em todas as interfaces do seu computador**: qualquer
  máquina da mesma rede (Wi-Fi do escritório, café) pode alcançá-las. Em redes que você não controla, publique só no
  loopback (`"127.0.0.1:9200:9200"`) ou pare os serviços.

## Solução de problemas

| Sintoma | Causa provável e solução |
|---|---|
| `network shared_net declared as external, but could not be found` | Rede não criada. Rode o comando da seção 1. |
| `port is already allocated` | Conflito de porta, veja acima. |
| Container reinicia em loop | `docker compose logs <servico>`. No MinIO, senha com menos de 8 caracteres; no Elasticsearch, `vm.max_map_count` baixo. |
| Container some com código 137 | Falta de memória (OOM). Aumente `mem_limit` e/ou a RAM do Docker Desktop. |
| Elasticsearch não sobe (`max virtual memory areas`) | O serviço auxiliar `sysctl-fix` já tenta corrigir; veja [Elasticsearch](services/elasticsearch.md). |
| Compass/cliente não conecta no MongoDB | Faltou `directConnection=true&authSource=admin`, veja [MongoDB](services/mongodb.md). |
| `pull access denied for minio/minio` | A imagem saiu do Docker Hub; o compose já usa `quay.io/minio/minio`. |
| Serviço subiu "vazio" depois de renomear a pasta | O Compose criou volumes novos com o nome novo; os dados antigos estão em `<pastaantiga>_<volume>`. |
| `docker compose ls` mostra caminho antigo | Os rótulos do container guardam onde ele foi criado; atualizam no próximo `docker compose up -d`. |

## Usando em outro computador

```bash
git clone <url-do-repositorio> dockyard && cd dockyard
docker network create shared_net
cd <servico> && docker compose up -d
```

Os dados **não** vão junto (ficam em volumes locais). Para levar dados, exporte/importe pelo próprio serviço
(`pg_dump`, `mongodump`, `mc mirror`, etc.).
