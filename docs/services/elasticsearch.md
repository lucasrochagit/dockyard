# Elasticsearch

Nó único, **sem autenticação e sem TLS**, com o mínimo de recursos que ainda indexa e consulta.

| | |
|---|---|
| Compose | [`elasticsearch/docker-compose.yml`](../../elasticsearch/docker-compose.yml) |
| Imagem | `docker.elastic.co/elasticsearch/elasticsearch:9.5.2` (versão fixa) |
| Container / host na rede | `elasticsearch-local` / `elasticsearch` |
| Porta | `9200` (HTTP/REST). A `9300` (transporte entre nós) não é publicada |
| Credenciais | nenhuma |
| Volume | `es-data` (`/usr/share/elasticsearch/data`) |
| Recursos | heap `-Xms512m -Xmx512m`, `mem_limit: 1.5g`, `cpus: 1.0` |

## Conectar

| De onde | Endereço |
|---|---|
| Container / n8n | `http://elasticsearch:9200` |
| Seu computador | `http://localhost:9200` |

```bash
curl http://localhost:9200
curl "http://localhost:9200/_cluster/health?pretty"   # aspas: no zsh o "?" sem aspas é interpretado como glob
```

A subida leva em torno de 15 segundos. Não há `healthcheck` no compose; use `/_cluster/health`.

## O serviço auxiliar `sysctl-fix`

O Elasticsearch exige `vm.max_map_count >= 262144` no kernel. No Docker Desktop esse valor da VM é menor, e o container não sobe
(`max virtual memory areas vm.max_map_count [...] is too low`). Não dá para resolver com `sysctls:` no compose (neste kernel a
chave não é isolada por container e o Docker recusa criar o container), e o valor da VM **volta ao padrão a cada reinício do Docker Desktop**.

Por isso o compose tem um segundo serviço, `sysctl-fix`: um container `privileged` com `pid: host` que roda
`nsenter -t 1 ... sysctl -w vm.max_map_count=262144` e termina. O Elasticsearch só inicia depois dele
(`depends_on: service_completed_successfully`), então o ajuste é reaplicado em todo `docker compose up`.

- No **Windows (Docker Desktop + WSL 2)** o ajuste foi validado: `sysctl-fix` sai com código 0 e o Elasticsearch sobe `green`.
  Nessa máquina o consumo ficou em ~990 MB (64% do limite), acima dos ~850 MB medidos no macOS.
- Em **Linux nativo**, esse container altera o `vm.max_map_count` do **kernel do próprio host** (até o próximo reboot).
  Isso não foi testado aqui; se preferir, configure em `/etc/sysctl.d/` e remova o serviço auxiliar.
- Exige `privileged: true`. É uma escolha consciente para ambiente local; não replique em outros contextos.
- `docker compose down` remove o container auxiliar junto com os demais; ele é recriado no próximo `up`.

## Limitações e pontos de atenção

- **Sem segurança alguma** (`xpack.security.*` desativado): qualquer cliente que alcance a porta lê, grava e apaga índices.
  Como `9200` é publicada em todas as interfaces, na prática qualquer máquina da sua rede local alcança.
- **Memória apertada por desenho.** Heap de 512 MB; o processo inteiro (JVM + memória fora do heap) usa ~850 MB mesmo parado.
  Com `mem_limit: 1g` o uso chegava a 84% do limite (risco de OOM ao indexar), por isso o limite é 1.5 GB (~55% sob carga leve).
  Para volumes de dados maiores, aumente o heap **e** o `mem_limit` juntos (heap ≤ 50% do limite).
- **Cluster de 1 nó:** índices criados com réplica (padrão do Elasticsearch) ficam `yellow`, porque a réplica não tem onde ser alocada.
  Crie índices com `"number_of_replicas": 0` para ficarem `green`.
- **Limites de disco do Elasticsearch** (watermarks): ele para de alocar shards quando o disco passa de ~85% e bloqueia escrita acima de ~95%.
  O que conta é o disco da VM do Docker; se o disco virtual encher, os índices ficam somente leitura.
- Sem Kibana (só a API REST). Sem plugins adicionais.
- Versão **fixa** (9.5.2): trocar de versão major com o volume existente pode ser incompatível; para atualizar, recrie o volume.
- Não validado fora do macOS/Docker Desktop.

## Validação realizada

- Cluster `green` em 1 nó, sem credenciais, respondendo em `localhost:9200`; presente na `shared_net`.
- Criação de índice e indexação de 51 documentos, com memória estável (~847 MB de 1.5 GB); índice de teste removido depois.
- Reaplicação automática do `vm.max_map_count`: o valor foi derrubado manualmente para 65530 (simulando um reinício do Docker Desktop) e o `docker compose up -d` o corrigiu e subiu o cluster.
