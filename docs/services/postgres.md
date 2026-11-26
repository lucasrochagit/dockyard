# PostgreSQL

Banco relacional com os parâmetros reduzidos para uso local. Suporta `LISTEN`/`NOTIFY` sem nenhuma configuração extra.

| | |
|---|---|
| Compose | [`postgres/docker-compose.yml`](../../postgres/docker-compose.yml) |
| Imagem | `postgres:17-alpine` |
| Container / host na rede | `postgres-local` / `postgres` |
| Porta | `5432` |
| Credenciais | usuário `postgres`, senha `postgres`, banco `postgres` |
| Volume | `postgres_data` (`/var/lib/postgresql/data`) |
| Recursos | `mem_limit: 512m`, `cpus: 1.0`; `shared_buffers=64MB`, `max_connections=50` |
| Healthcheck | `pg_isready -U postgres -d postgres` |

## Conectar

| De onde | Endereço |
|---|---|
| Container / n8n | `postgresql://postgres:postgres@postgres:5432/postgres` |
| Seu computador | `postgresql://postgres:postgres@localhost:5432/postgres` |

No n8n (credencial Postgres): host `postgres`, porta `5432`, banco `postgres`, usuário/senha `postgres`, SSL desativado.

## LISTEN / NOTIFY

É um recurso nativo do Postgres e funciona na imagem oficial. O node **Postgres Trigger** do n8n usa esse mecanismo: ele cria sozinho a trigger e a função
no banco para disparar o fluxo em `INSERT`/`UPDATE`/`DELETE`; também é possível escutar um canal próprio e emitir com `NOTIFY canal, 'payload'` ou `pg_notify()`.

Limitações do próprio recurso (valem em qualquer Postgres):

- A notificação **não é persistida**: se ninguém estiver escutando no momento, ela se perde.
- O listener precisa de uma **conexão dedicada e de longa duração**. Um pooler em modo *transaction* (PgBouncer, por exemplo) quebra isso.
- O payload é limitado a ~8000 bytes; o padrão é enviar só um identificador e buscar o resto com `SELECT`.
- A notificação só é entregue após o `COMMIT` da transação que a emitiu.

## Limitações e pontos de atenção

- **Parâmetros enxutos:** `shared_buffers=64MB` e `max_connections=50` (o padrão do Postgres é 128 MB e 100). Se vários clientes
  (n8n, APIs, ferramentas de SQL) abrirem muitas conexões, aumente `max_connections` na seção `command:` do compose.
- **Só existe o superusuário `postgres`.** Crie papéis e bancos dedicados para suas aplicações.
- **`POSTGRES_USER/PASSWORD/DB` só valem na primeira inicialização** (volume vazio).
- **Replicação lógica / CDC não está habilitada** (`wal_level` no padrão `replica`). Ferramentas como Debezium exigiriam `-c wal_level=logical`.
- **Trocar de versão major** (17 → 18, por exemplo) com o volume existente não funciona: o formato dos dados muda. É preciso `pg_dump`/restore ou recriar o volume.
  A partir do Postgres 18 a imagem também mudou o caminho de dados do volume (`/var/lib/postgresql`); ajuste o mount ao atualizar.
- Sem TLS e sem scripts de inicialização montados (poderiam ser adicionados em `/docker-entrypoint-initdb.d`).
- Leve: ~19 MB de memória parado.

## Validação realizada

- Container `healthy` e na `shared_net`.
- De um container cliente separado: tabela com trigger `AFTER INSERT` chamando `pg_notify`; o listener recebeu a notificação do `INSERT` (payload com o id) e a de um `NOTIFY` manual. Tabela e função de teste removidas.
- **Não testado:** o node Postgres Trigger do n8n em si.
