# Redis

Armazenamento chave-valor em memória, sem senha, com persistência e teto de memória.

| | |
|---|---|
| Compose | [`redis/docker-compose.yml`](../../redis/docker-compose.yml) |
| Imagem | `redis:8-alpine` |
| Container / host na rede | `redis-local` / `redis` |
| Porta | `6379` |
| Credenciais | nenhuma |
| Volume | `redis_data` (`/data`) |
| Recursos | `mem_limit: 256m`, `cpus: 0.5`; `maxmemory 128mb`, `maxmemory-policy noeviction`, `appendonly yes` |
| Healthcheck | `redis-cli ping` |

## Conectar

| De onde | Endereço |
|---|---|
| Container / n8n | `redis://redis:6379` |
| Seu computador | `redis://localhost:6379` ou `redis-cli -h localhost -p 6379` |

No n8n (credencial Redis): host `redis`, porta `6379`, sem usuário e sem senha, banco `0`.

## Configuração

| Parâmetro | Valor | Por quê |
|---|---|---|
| `--appendonly yes` | AOF ligado | os dados sobrevivem a reinícios do container (gravados no volume) |
| `--maxmemory 128mb` | teto de memória de dados | evita crescer sem limite; fica abaixo do `mem_limit` de 256 MB, deixando folga para o processo |
| `--maxmemory-policy noeviction` | sem remoção automática | ao encher, as **escritas retornam erro** em vez de apagar chaves em silêncio |

## Limitações e pontos de atenção

- **Sem autenticação.** Qualquer cliente que alcance a porta tem controle total. Como `6379` é publicada em todas as interfaces, máquinas da sua rede local também alcançam.
- **`noeviction` não serve como cache puro.** Quando os 128 MB enchem, novas escritas falham (`OOM command not allowed`). Para usar como cache, troque para `--maxmemory-policy allkeys-lru`.
  A escolha atual protege usos em que perder chaves seria pior (filas, locks, estado de fluxos).
- **Teto de 128 MB de dados** e **0.5 CPU**: suficiente para testes e filas leves; não serve para benchmarks.
- **Instância única, sem réplicas, sem TLS, sem ACLs.** O n8n em *queue mode* (que usa Redis) não está configurado neste repositório.
- **Conflito de porta** com um Redis local: publique em outra porta (`"6380:6379"`).
- Persistência AOF não substitui backup; `docker compose down -v` apaga os dados.
- Leve: ~12 MB de memória parado.

## Validação realizada

- Container `healthy` e na `shared_net`; versão 8.12.0.
- De um container cliente separado: `SET`, `GET` e `DEL`.
- **Não testado:** o node Redis do n8n em si, nem a persistência AOF após reinício do container.
