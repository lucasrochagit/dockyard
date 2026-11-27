# RabbitMQ

Broker de mensageria (AMQP) com a interface de management.

| | |
|---|---|
| Compose | [`rabbitmq/docker-compose.yml`](../../rabbitmq/docker-compose.yml) + [`rabbitmq.conf`](../../rabbitmq/rabbitmq.conf) |
| Imagem | `rabbitmq:4.1-management` |
| Container / host na rede | `rabbitmq-local` / `rabbitmq` |
| Portas | `5672` (AMQP), `15672` (UI/API de management) → http://localhost:15672 |
| Credenciais | `guest` / `guest` (vhost `/`) |
| Volume | `rabbitmq-data` (`/var/lib/rabbitmq`) |
| Recursos | `mem_limit: 512m`, `cpus: 1.0` |

## Conectar

| De onde | Endereço |
|---|---|
| Container / n8n | `amqp://guest:guest@rabbitmq:5672/` |
| Seu computador | `amqp://guest:guest@localhost:5672/` e http://localhost:15672 |

No n8n, credencial RabbitMQ: host `rabbitmq`, porta `5672`, usuário/senha `guest`, vhost `/`, SSL desligado.

## Configuração (`rabbitmq.conf`, montado como somente leitura)

| Parâmetro | Valor | Efeito |
|---|---|---|
| `total_memory_available_override_value` | `512MiB` | Memória total que o RabbitMQ considera. **Necessário:** ele não lê o `mem_limit` do container e usaria a RAM da VM inteira (com 16 GB, alarme em ~6.4 GB; o container seria morto por OOM em 512 MB antes). Mantenha igual ao `mem_limit` |
| `vm_memory_high_watermark.relative` | `0.4` | Alarme de memória a 40% do total acima (~204 MB); acima disso o broker bloqueia publicadores. Confira no log: `Memory high watermark set to 204 MiB ... of 512 MiB` |
| `disk_free_limit.absolute` | `500MB` | Alarme de disco: bloqueia publicadores se o espaço livre cair abaixo disso |
| `listeners.tcp.default` / `management.tcp.port` | `5672` / `15672` | Portas padrão |
| `loopback_users.guest` | `false` | Permite o login de `guest` vindo de fora de `localhost` |

## Limitações e pontos de atenção

- **`loopback_users.guest = false` é uma flexibilização de segurança.** Por padrão o RabbitMQ só aceita `guest` vindo de `localhost`; sem essa linha, o n8n (que conecta pela rede Docker) seria recusado. Qualquer cliente que alcance as portas entra como `guest/guest`.
- **Limite de 512 MB:** com filas grandes, o alarme de memória (~200 MB) dispara cedo e os publicadores ficam bloqueados. É um sinal para drenar filas ou aumentar `mem_limit`.
- **`RABBITMQ_DEFAULT_USER/PASS` só valem na primeira inicialização** (volume vazio). Para trocar o usuário depois: `docker compose down -v` (apaga mensagens e definições) ou use a UI/`rabbitmqctl`.
- Sem TLS, sem clustering, sem `healthcheck` declarado no compose.
- O `rabbitmq.conf` é lido no start; alterações exigem `docker compose restart`.

## Validação realizada

- Subida limpa com a configuração; volume existente reaproveitado.
- Login `guest/guest` a partir de **outro container na `shared_net`**: API de management (HTTP 200; senha errada retorna 401) e AMQP (declarar fila, publicar e consumir uma mensagem).
- Consumo de memória não foi medido.
