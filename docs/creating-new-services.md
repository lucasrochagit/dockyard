# Guia para criar novos serviços

Referência para **agentes (e pessoas)** que forem adicionar uma nova imagem a este repositório. Siga o padrão dos serviços existentes
(`postgres/`, `redis/`, `mariadb/` são os mais recentes e limpos; `elasticsearch/` e `mongodb/` mostram os casos difíceis).

## Objetivo do repositório

Ambiente **local de testes**: o usuário quer rodar o serviço na própria máquina, com o **mínimo de recursos que ainda permita funcionamento pleno**,
na mesma rede (`shared_net`) dos demais, para integrar tudo a partir do n8n ou de APIs próprias. Não é produção, não é alta disponibilidade, não é segurança.

## Premissas (o "porquê" de cada decisão)

1. **Uma pasta por serviço:** `<servico>/docker-compose.yml`, nome curto em minúsculas. O nome da pasta é o nome do projeto Compose e o prefixo dos volumes: **nunca renomeie** pastas existentes.
2. **Rede compartilhada:** o serviço entra em `shared_net`, declarada `external: true` (ela é criada uma vez com `docker network create shared_net`). Nunca crie redes novas, nem use `network_mode: host`.
3. **Nomes:** o **nome do serviço = `hostname` = nome DNS documentado** (`postgres`, `redis`...). É o que outros containers usam para conectar.
   `container_name: <servico>-local` (exceções históricas: `mongodb-search-local`, `minio_s3_simulator`, `n8n`).
4. **Recursos mínimos para uso pleno** (a premissa principal): defina `mem_limit` e `cpus`, e **meça** (veja a seção de validação). "Mínimo" significa o menor valor que **não** dá risco de OOM sob uso real de teste.
   - Pesquise o mínimo oficial da imagem. Se existir (MongoDB Atlas Local: 2 GB/2 CPUs), ele é o piso.
   - Serviços em JVM (Elasticsearch, `mongot`) e bancos têm configurações de memória próprias: limite-as explicitamente (heap `-Xms=-Xmx`, `shared_buffers`, `innodb_buffer_pool_size`, `maxmemory`...) para a imagem não dimensionar pela RAM da VM inteira.
   - O uso em repouso medido deve ficar bem abaixo do limite: **referência ≤ ~60% do limite**. Lição do Elasticsearch: com `1g`, o processo parado já usava 84%, então o limite subiu para `1.5g` (~55% sob carga leve).
   - Valores usados hoje: serviços leves 256–512 MB e 0.5–1.0 CPU; pesados 1.5–2 GB e 1–2 CPUs.
5. **Sem fricção de credenciais, mas nunca segredos reais.** Prefira **sem autenticação** quando o serviço permitir e o uso for só local (Elasticsearch, Redis); caso contrário use credenciais triviais (`admin/admin`, `postgres/postgres`),
   respeitando o mínimo que a imagem exige (MinIO: senha ≥ 8 caracteres). Tudo **inline no compose**, sem `.env`, para o clone funcionar sem passos extras.
6. **Persistência em volume nomeado** (`<servico>_data`, ou o nome que a imagem recomendar). Volumes antigos usam nomes variados (`rabbitmq-data`, `es-data`); novos seguem `<servico>_data`.
7. **`restart: unless-stopped`** em todo serviço (o `mongodb` é uma exceção histórica, não copie).
8. **Healthcheck:** use o que a imagem já traz (`mongodb-atlas-local`, `healthcheck.sh` do MariaDB) ou declare um com a CLI da própria imagem (`pg_isready`, `redis-cli ping`). Permite `depends_on: condition: service_healthy`.
9. **Porta padrão do serviço**, no formato `"host:container"`. Publique só as portas necessárias (no Elasticsearch a 9300 de transporte ficou de fora). Registre a porta na tabela de conflitos de [getting-started.md](getting-started.md).
10. **Versão da imagem:** prefira **tag fixa** (major ou exata: `17-alpine`, `11.8`, `9.5.2`) e variantes `alpine`/`slim` quando existirem. `latest` só quando não houver alternativa (n8n, mongodb-atlas-local, minio hoje usam `latest`: dívida, não padrão).
11. **Confirme que a imagem pode ser baixada** do registry escolhido antes de escrever o compose. Caso real: `minio/minio` no Docker Hub falha com `pull access denied`; a imagem oficial passou para `quay.io/minio/minio`. Prefira o registry oficial do fornecedor.
12. **Configuração extra:** por variáveis de ambiente ou `command:` quando possível; se precisar de arquivo, coloque-o **ao lado do compose** e monte `:ro` (exemplo: `rabbitmq/rabbitmq.conf`).
13. **Comentários curtos em português**, só para o **porquê** de valores não óbvios (limites, flags, workarounds). Os arquivos novos usam ASCII (sem acentos) nos comentários.
14. **Fique no escopo pedido.** Não adicione UIs extras (Kibana, pgAdmin...), réplicas, TLS, nem mexa nos outros composes, a menos que o usuário peça. Não faça commit nem push sem pedido explícito.
15. **Helpers privilegiados são último recurso.** O `sysctl-fix` do Elasticsearch (`privileged` + `pid: host`) só existe porque o serviço não sobe sem ele. Se algo parecido for inevitável, isole em um serviço auxiliar com `restart: "no"` e `network_mode: "none"`, e documente os riscos.

## Template

```yaml
services:
  <servico>:
    image: <registry>/<imagem>:<tag-fixa>
    container_name: <servico>-local
    hostname: <servico>

    ports:
      - "<porta>:<porta>"

    environment:
      # credenciais triviais, validas apenas para uso local
      VAR: valor

    # parametros reduzidos para uso local (explique o PORQUE dos valores)
    command:
      - ...

    mem_limit: 512m
    cpus: 1.0

    volumes:
      - <servico>_data:/caminho/de/dados

    healthcheck:
      test: ["CMD", "..."]
      interval: 10s
      timeout: 5s
      retries: 5

    restart: unless-stopped

    networks:
      - shared_net

volumes:
  <servico>_data:

networks:
  shared_net:
    external: true
```

## Procedimento

1. **Pesquisar** (documentação oficial): imagem e registry, porta padrão, mínimo de recursos, como desativar autenticação/TLS, caminho de dados, variáveis que só valem na primeira inicialização, requisitos de kernel/plataforma.
2. **Criar** `<servico>/docker-compose.yml` pelo template.
3. **Validar a sintaxe:** `docker compose config -q` dentro da pasta.
4. **Subir e esperar ficar saudável:** `docker compose up -d`, depois conferir `docker inspect --format='{{.State.Health.Status}}' <container>` em loop curto.
5. **Verificar a rede:** `docker network inspect shared_net --format '{{range .Containers}}{{.Name}} - {{.IPv4Address}}{{"\n"}}{{end}}'`.
6. **Testar a partir de outro container na `shared_net`**, que é como o n8n vai usar, com o cliente da própria imagem:
   `docker run --rm --network shared_net <imagem-cliente> <cli> -h <servico> ...` (por exemplo `redis-cli -h redis`, `psql -h postgres`, `mariadb -h mariadb`). Teste escrita e leitura, não só conexão TCP, e teste a **autenticação** (conexão aberta não prova que o login funciona).
7. **Testar do host** (`localhost:<porta>`), como o usuário fará em ferramentas gráficas.
8. **Medir sob uso real:** `docker stats --no-stream <container>` em repouso e **depois de carga representativa** (criar índice/tabela, inserir dados, gerar tráfego). Ajuste `mem_limit` se passar de ~60%. Confira também `docker inspect <container> --format '{{.State.OOMKilled}}'`.
9. **Limpar** os dados de teste (tabelas, índices, buckets) e as redes/containers auxiliares que você criou.
10. **Documentar** (próxima seção).
11. **Relatar ao usuário** o que foi criado, os ajustes feitos em relação ao pedido (e por quê) e **o que não foi testado**.

## Armadilhas já encontradas

| Situação | O que aconteceu | Como tratar |
|---|---|---|
| Imagem removida do Docker Hub | `pull access denied for minio/minio` | Procurar o registry oficial atual (Quay.io, `docker.elastic.co`...) e testar `docker pull` antes |
| Credencial abaixo do mínimo da imagem | MinIO em loop de reinício (`MINIO_ROOT_PASSWORD` precisa de ≥ 8 caracteres) | Ler os logs (`docker logs`) quando o container reinicia; ajustar e avisar o usuário que difere do pedido |
| Requisito de kernel na VM do Docker Desktop | Elasticsearch exige `vm.max_map_count >= 262144`; `sysctls:` no compose é recusado (`not in a separate kernel namespace`) e o valor reseta a cada reinício | Serviço auxiliar `sysctl-fix` (veja `elasticsearch/`) |
| Replica set anunciando o hostname do container | MongoDB Atlas Local registra `mongodb:27017`, que não resolve no host | Documentar `directConnection=true`; não mudar o `hostname` |
| Variáveis só valem no primeiro boot | `POSTGRES_*`, `MARIADB_*`, `RABBITMQ_DEFAULT_*` não mudam com o volume já criado | Avisar; `docker compose down -v` apaga os dados |
| Segurança padrão bloqueia o uso em rede | RabbitMQ recusa `guest` fora de `localhost` | `loopback_users.guest = false` no `rabbitmq.conf` |
| Limite de memória apertado | Elasticsearch com 1 GB já em 84% parado | Medir e dar folga; regra de ≤ ~60% |
| Porta padrão ocupada na máquina do usuário | `3306`/`6379` conflitam com instalações locais | Documentar como trocar o lado esquerdo do mapeamento |
| Entrypoint da imagem cliente | `quay.io/minio/mc` tem `mc` como entrypoint; `sh -c` falha | Usar `--entrypoint sh` |
| Shell do agente (zsh/macOS) | URL com `?` sem aspas vira glob; o comando `timeout` não existe; `status` é variável somente leitura | Aspas em URLs, laço com `sleep`, outro nome de variável |

## Documentação obrigatória

Ao terminar, **no mesmo trabalho**:

1. Criar `docs/services/<servico>.md` com: tabela de resumo (compose, imagem, container/host, portas, credenciais, volume, recursos, healthcheck), seção **Conectar** (container/n8n e host), explicação de qualquer configuração não óbvia,
   **Limitações e pontos de atenção** e **Validação realizada** (incluindo o que **não** foi testado). Use [postgres.md](services/postgres.md) ou [redis.md](services/redis.md) como modelo.
2. Atualizar a **tabela de serviços do `README.md`** (imagem, host, portas, credenciais, limites).
3. Atualizar em [getting-started.md](getting-started.md) a tabela de **Recursos** (com o consumo medido) e a de **Portas**.
4. Se o n8n puder usar o serviço, acrescentar a linha na tabela de [n8n.md](services/n8n.md).

Regras de redação das limitações: seja específico e honesto. Diferencie o que é limitação **da configuração** (ex.: `noeviction`, sem autenticação) do que é limitação **do software** (ex.: tamanho do payload do `NOTIFY`), e marque como não testado o que não foi verificado de fato.

## Checklist de conclusão

- [ ] `docker compose config -q` passa; sobe saudável; está na `shared_net`
- [ ] Escrita/leitura e login testados de **outro container** da `shared_net` e do **host**
- [ ] `mem_limit`/`cpus` definidos, consumo medido em repouso e sob carga, `OOMKilled` = `false`
- [ ] Sem segredos reais; credenciais respeitam o mínimo da imagem
- [ ] Tag de imagem fixa (ou `latest` justificado na documentação)
- [ ] Dados de teste removidos; nada sobrando fora do que foi pedido
- [ ] `docs/services/<servico>.md`, README e getting-started atualizados
- [ ] Resumo ao usuário com desvios do pedido e itens não testados
