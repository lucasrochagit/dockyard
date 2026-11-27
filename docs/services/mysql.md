# MySQL

Banco relacional MySQL (versão LTS), com parâmetros reduzidos para uso local.

| | |
|---|---|
| Compose | [`mysql/docker-compose.yml`](../../mysql/docker-compose.yml) |
| Imagem | `mysql:8.4` (LTS) |
| Container / host na rede | `mysql-local` / `mysql` |
| Porta | `3307` no host → `3306` no container |
| Credenciais | `root` / `root`; usuário `app` / `app` com acesso ao banco `app` |
| Volume | `mysql_data` (`/var/lib/mysql`) |
| Recursos | `mem_limit: 512m`, `cpus: 1.0`; `innodb-buffer-pool-size=64M`, `max-connections=50`, `performance-schema=OFF`, `skip-log-bin` |
| Healthcheck | `mysqladmin ping -h 127.0.0.1 -uroot -proot` |

## Conectar

| De onde | Endereço |
|---|---|
| Container / n8n | `mysql://app:app@mysql:3306/app` |
| Seu computador | `mysql://app:app@localhost:3307/app` |

No n8n use o node **MySQL**: host `mysql`, porta `3306`, banco `app`, usuário `app`, senha `app`.

## Por que a porta 3307 no host

O [MariaDB](mariadb.md) deste repositório já publica a `3306`. Publicar o MySQL em `3307` permite subir os dois ao mesmo tempo.
Isso só afeta o acesso **do host**: entre containers da `shared_net` o endereço continua `mysql:3306`.

## Limitações e pontos de atenção

- **Não interrompa a primeira subida.** Se o container for parado (ou o Docker Desktop fechado) enquanto o MySQL cria os arquivos, o volume fica incompleto
  e o container entra em loop de reinício com `Cannot create redo log files because data files are corrupt or the database was not shut down cleanly`.
  O script de entrada vê a pasta de dados e pula a inicialização. Correção (apaga os dados): `docker compose down -v` e `docker compose up -d`.
  Aconteceu de fato neste repositório.
- **`MYSQL_*` só valem na primeira inicialização** (volume vazio). Para trocar senhas depois, use SQL (`ALTER USER`) ou `docker compose down -v` (apaga os dados).
- **`root` aceita conexões de qualquer host** (padrão da imagem oficial quando `MYSQL_ROOT_HOST` não é definido). Aceitável aqui por ser ambiente local.
- **Autenticação `caching_sha2_password`** (padrão do MySQL 8.4; o plugin `mysql_native_password` vem desativado). Clientes e drivers
  antigos (MySQL 5.x, conectores desatualizados) podem falhar no login; sem TLS, alguns clientes exigem `allowPublicKeyRetrieval=true` (JDBC) ou `get-server-public-key`.
- **`performance_schema` desligado** para economizar memória: diagnósticos de desempenho (views do schema `sys`, métricas de espera) ficam indisponíveis.
  Para depurar consultas lentas, remova `--performance-schema=OFF` do `command:` (custa cerca de 100 MB+ de RAM).
- **Binlog desligado (`--skip-log-bin`)**: não há replicação nem CDC (Debezium etc.), nem recuperação *point-in-time*. Remova a flag se precisar.
- **`innodb-buffer-pool-size=64M` e `max-connections=50`:** adequados a testes; cargas maiores pedem valores e `mem_limit` maiores.
- **Trocar de versão** (8.4 → 9.x) com o volume existente é uma atualização de dados in-place, sem volta para a versão anterior. Faça `mysqldump` antes.
- Imagem baseada em Oracle Linux (não há variante `alpine`); o download é maior que o do MariaDB.
- Sem TLS configurado e sem scripts de inicialização montados (podem ser adicionados em `/docker-entrypoint-initdb.d`).

## Validação realizada

- `docker compose config -q` sem erros; container `healthy` e na `shared_net`; primeira inicialização completa em ~10 s.
- De um container cliente separado na `shared_net`: login como `app` (criar tabela, inserir 50 mil linhas, ler, remover) e como `root`
  (`SELECT VERSION()` → 8.4.13; conferidos `innodb_buffer_pool_size` = 64 MB, `max_connections` = 50, `performance_schema` e `log_bin` desligados).
- Pela porta publicada `3307` (via `host.docker.internal`): login como `app`.
- Memória: ~186 MB em repouso, ~194 MB após inserir 50 mil linhas (38% do limite); `OOMKilled=false`.
- **Não testado:** o node MySQL do n8n e clientes gráficos no host (Workbench, DBeaver).
