# MongoDB com Atlas Search

Instância local do MongoDB com **Atlas Search / Vector Search**, para testar buscas do Atlas sem a nuvem e,
ao mesmo tempo, servir de banco local para uso geral. A imagem `mongodb-atlas-local` empacota o `mongod`
(configurado como replica set de 1 nó) e o `mongot` (o motor de busca, que roda em JVM).

| | |
|---|---|
| Compose | [`mongodb/docker-compose.yml`](../../mongodb/docker-compose.yml) |
| Imagem | `mongodb/mongodb-atlas-local` (tag `latest`, ~1.4 GB) |
| Container / host na rede | `mongodb-search-local` / `mongodb` |
| Porta | `27017` |
| Credenciais | `admin` / `admin` (usuário root, `authSource=admin`) |
| Volumes | `db` (`/data/db`), `configdb` (`/data/configdb`), `mongot` (`/data/mongot`) |
| Recursos | `mem_limit: 2g`, `cpus: 2.0` (mínimo oficial para o Atlas Local) |

## Conectar

| De onde | String de conexão |
|---|---|
| Container / n8n / sua API | `mongodb://admin:admin@mongodb:27017/?directConnection=true&authSource=admin` |
| Seu computador (Compass, mongosh, código) | `mongodb://admin:admin@localhost:27017/?directConnection=true&authSource=admin` |

No MongoDB Compass, cole a segunda string no campo de URI.

**Sempre inclua `directConnection=true`.** O único membro do replica set está registrado como `mongodb:27017`
(o hostname do container). Esse nome só resolve dentro da rede Docker; do seu computador ele não existe. Sem a
flag, o driver tenta descobrir o replica set e roteia para esse nome, e clientes como o Compass podem travar em
"Connecting…" mesmo com a porta aberta. Com `directConnection=true` o cliente fala apenas com o endereço informado.

## Atlas Search

Os índices de busca são criados pelo próprio mongosh/driver (não há painel do Atlas):

```js
use meubanco
db.produtos.createSearchIndex("default", { mappings: { dynamic: true } })
db.produtos.getSearchIndexes()
```

Depois use `$search` / `$vectorSearch` em pipelines de agregação. A indexação é assíncrona: logo após criar o
índice ele ainda não está pronto para consulta.

## Esperar o banco ficar saudável

O `healthcheck` já vem embutido na imagem (cobre `mongod` + `mongot`); um serviço seu pode usar
`depends_on: { mongodb: { condition: service_healthy } }` (há um exemplo comentado no compose).

## Limitações e pontos de atenção

- **Não interrompa a primeira subida.** Se o container for parado antes de o replica set ser iniciado, as subidas seguintes não o iniciam:
  o log mostra `Collection [local.oplog.rs] not found` repetidamente e, após ~2 min, `panic: error checking mongod ... ReplicaSetNoPrimary ... RSGhost`
  (o container sai com código 2). Correção (apaga os dados): `docker compose down -v` e `docker compose up -d`.
- **Consumo alto comparado aos outros serviços.** Mínimo oficial: 2 GB de RAM e 2 CPUs livres. Parado, o container usa
  ~380 MB, e sobe para ~660 MB com um índice de Search criado. Com limite menor que 2 GB o `mongot` tende a ser morto por OOM
  quando há índices reais.
- **Imagem grande e sem versão fixa** (`latest`): o primeiro `up` baixa ~1.4 GB, e um `pull` pode mudar a versão.
- **Somente desenvolvimento/testes.** É um ambiente local do Atlas, sem garantia de paridade total com o Atlas na nuvem.
- **Não altere o `hostname: mongodb`.** O nome fica gravado na configuração do replica set (`rs.conf()`); mudá-lo depois de
  inicializado pode quebrar o replica set até que os volumes sejam recriados.
- **Replica set de 1 nó**, sem sharding e sem TLS. O usuário `admin` é root: crie usuários dedicados se quiser simular permissões.
- **Scripts de inicialização:** arquivos `.js`/`.sh` em `mongodb/init/` rodam só na **primeira** subida (volumes vazios). A pasta
  `init/` é montada mas está vazia e não é versionada pelo Git; o Docker a cria sozinho. Crie-a você mesmo se quiser seed de dados.
- Não há `restart: unless-stopped` neste compose (diferente dos demais): depois de reiniciar o Docker/máquina o container não volta sozinho.
- `docker compose down -v` apaga os três volumes (dados e índices de busca).

## Validação realizada

- Container `healthy`; login `admin/admin` e escrita de documento funcionando; `createSearchIndex` executado com sucesso;
  conexão do host com `directConnection=true`; n8n alcança `mongodb:27017` por TCP.
- Windows 11 (26/11/2026), após recriar os volumes: replica set `PRIMARY`; 1000 documentos inseridos e lidos; índice de Search `READY`
  e consulta `$search` retornando os 500 documentos esperados; banco de teste removido. Login pelo host (`host.docker.internal:27017`).
  Memória: ~656 MB parado, ~810 MB após criar o índice de Search (40% do limite).
- **Não exercitado:** `$vectorSearch` e o node MongoDB do n8n.
