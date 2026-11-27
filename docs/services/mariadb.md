# MariaDB

Banco relacional compatível com MySQL, com parâmetros reduzidos para uso local.

| | |
|---|---|
| Compose | [`mariadb/docker-compose.yml`](../../mariadb/docker-compose.yml) |
| Imagem | `mariadb:11.8` (LTS) |
| Container / host na rede | `mariadb-local` / `mariadb` |
| Porta | `3306` |
| Credenciais | `root` / `root`; usuário `app` / `app` com acesso ao banco `app` |
| Volume | `mariadb_data` (`/var/lib/mysql`) |
| Recursos | `mem_limit: 512m`, `cpus: 1.0`; `innodb-buffer-pool-size=64M`, `max-connections=50`, `performance-schema=OFF` |
| Healthcheck | `healthcheck.sh --connect --innodb_initialized` (script da própria imagem) |

## Conectar

| De onde | Endereço |
|---|---|
| Container / n8n | `mysql://app:app@mariadb:3306/app` |
| Seu computador | `mysql://app:app@localhost:3306/app` |

No n8n use o node **MySQL** (funciona com MariaDB): host `mariadb`, porta `3306`, banco `app`, usuário `app`, senha `app`.

## Limitações e pontos de atenção

- **A porta `3306` conflita com um MySQL/MariaDB instalado na sua máquina.** Se for o caso, publique em outra porta (`"3307:3306"`);
  entre containers continua `mariadb:3306`.
- **Não interrompa a primeira subida.** Se o container for parado no meio dela, o volume fica sem a senha do `root` e sem o usuário/banco `app`,
  mas o container sobe `healthy` mesmo assim: o sintoma é `Host '172.x.x.x' is not allowed to connect to this MariaDB server` para qualquer login de fora.
  O script de entrada vê a pasta de dados e não repete a inicialização. Correção (apaga os dados): `docker compose down -v` e `docker compose up -d`.
- **`MARIADB_*` só valem na primeira inicialização** (volume vazio). Para trocar senhas depois, use SQL (`ALTER USER`) ou `docker compose down -v` (apaga os dados).
- **`root` aceita conexões de qualquer host** (validado de outro container). Aceitável aqui por ser ambiente local.
- **`performance_schema` desligado** para economizar memória: diagnósticos de desempenho (views do schema `sys`, métricas de espera) ficam indisponíveis.
  Para depurar consultas lentas, remova `--performance-schema=OFF` do `command:` (custa cerca de 100 MB+ de RAM).
- **`innodb-buffer-pool-size=64M` e `max-connections=50`:** adequados a testes; cargas maiores pedem valores e `mem_limit` maiores.
- Sem TLS e sem scripts de inicialização montados (podem ser adicionados em `/docker-entrypoint-initdb.d`).
- Leve: ~110 MB de memória parado (21% do limite).

## Validação realizada

- Container `healthy` e na `shared_net`.
- De um container cliente separado: login como `app` (criar tabela, inserir, ler, remover) e como `root` (`SELECT VERSION()` → 11.8.x).
- Windows 11 (26/11/2026), após recriar o volume: logins `app` e `root` pela `shared_net` e pela porta 3306; 50 mil linhas
  inseridas e lidas; `innodb_buffer_pool_size` = 64 MB, `max_connections` = 50 e `performance_schema` desligado conferidos;
  tabelas de fuso horário carregadas. Memória: ~134 MB após a carga (26% do limite).
- **Não testado:** o node MySQL do n8n em si.
