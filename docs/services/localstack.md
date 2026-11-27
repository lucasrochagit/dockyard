# LocalStack (emulador de AWS)

Emula serviços da AWS localmente, para testar fluxos que usam S3, SQS, SNS e DynamoDB (por exemplo, a partir do n8n ou de uma API) sem conta na AWS.

| | |
|---|---|
| Compose | [`localstack/docker-compose.yml`](../../localstack/docker-compose.yml) |
| Imagem | `localstack/localstack:4.14` (última Community sem token, **congelada**) |
| Container / host na rede | `localstack-local` / `localstack` |
| Porta | `4566` (todos os serviços AWS na mesma porta) |
| Credenciais | qualquer valor (ex.: `test` / `test`), região `us-east-1`; conta `000000000000` |
| Serviços ligados | `s3`, `sqs`, `sns`, `dynamodb`, `sts`, `iam` |
| Volume | `localstack_data` (`/var/lib/localstack`: cache e logs, **não** os dados dos serviços) |
| Recursos | `mem_limit: 1g`, `cpus: 1.0` |
| Healthcheck | `curl -sf http://localhost:4566/_localstack/health` |

## Conectar

| De onde | Endpoint |
|---|---|
| Container / n8n | `http://localstack:4566` |
| Seu computador | `http://localhost:4566` |

Em qualquer SDK/CLI, aponte o *endpoint* para o endereço acima, use credenciais quaisquer e região `us-east-1`. Exemplo com a AWS CLI no host:

```bash
aws --endpoint-url http://localhost:4566 --region us-east-1 s3 ls
```

(A CLI exige credenciais configuradas; `aws configure` com `test`/`test` basta.)

- **S3:** use *path-style* (`http://localstack:4566/<bucket>/<chave>`). No boto3, `Config(s3={"addressing_style": "path"})`; em outros SDKs, `forcePathStyle: true`.
- **SQS:** as `QueueUrl` vêm como `http://localstack:4566/queue/us-east-1/000000000000/<fila>`.
- **Status dos serviços:** http://localhost:4566/_localstack/health.

No n8n, crie uma credencial **AWS** com Access Key/Secret `test`, região `us-east-1` e ligue **Custom Endpoints**, preenchendo `http://localstack:4566` nos serviços usados (S3, SQS, SNS). Para S3 também dá para usar a credencial S3 genérica com *Force Path Style*.

## Por que a versão 4.14

Desde a `2026.03.0` o LocalStack publica uma imagem única que **só sobe com `LOCALSTACK_AUTH_TOKEN`**, ligado a uma conta em app.localstack.cloud.
O plano gratuito (Hobby) é **só para uso não comercial**, e o token é um segredo real, o que fugiria da regra do repositório de manter tudo inline no compose.
A `4.14.0` (build de 26/02/2026) é a última imagem Community da série 4.x e sobe sem token; ao iniciar ela mostra um aviso sobre a mudança, que pode ser ignorado.

## Limitações e pontos de atenção

- **Os dados somem ao reiniciar o container** (validado: buckets, filas, tópicos e tabelas desaparecem após `docker restart`). Persistência é recurso pago.
  Recrie os recursos com um script de inicialização: arquivos `.sh` montados em `/etc/localstack/init/ready.d/` rodam quando o LocalStack fica pronto (não configurado aqui).
- **Versão congelada:** a 4.14 não recebe correções de bugs nem de segurança. Serviços e APIs novos da AWS não chegam. Não use em nada exposto.
- **Sem autenticação**, e a porta `4566` é publicada em todas as interfaces: qualquer pessoa na sua rede pode ler e gravar nos buckets, filas e tabelas.
  Para restringir à sua máquina, use `"127.0.0.1:4566:4566"`.
- **Sem Lambda, de propósito.** Ela exigiria montar `/var/run/docker.sock`, o que dá ao LocalStack controle total do Docker da máquina
  (e, via a porta sem autenticação, permitiria a qualquer um na rede executar código na sua máquina). Os containers das funções também ficariam fora do `mem_limit`.
- **Só os serviços de `SERVICES` respondem.** Para usar outro (ex.: `secretsmanager`, `ssm`, `events`), acrescente-o à lista e rode `docker compose up -d`.
  Nem todo serviço existe na edição Community; veja o status em `/_localstack/health`.
- **`QueueUrl` usa o host `localstack`**, inclusive quando a fila é criada a partir do seu computador. SDKs recentes enviam a requisição para o *endpoint* configurado e não têm problema;
  um cliente que chame a `QueueUrl` diretamente do host falha (`localstack` não resolve fora da rede Docker). Nesse caso, troque o host da URL por `localhost`.
- **URLs pré-assinadas do S3** carregam o host usado na assinatura: geradas dentro da rede (`localstack:4566`) não abrem no navegador do host; gere-as com o endpoint `http://localhost:4566`.
- **HTTPS não configurado:** o LocalStack tenta baixar um certificado de teste de `api.localstack.cloud` e, sem acesso, usa um autoassinado. Use `http://`.
- **Telemetria desligada** (`DISABLE_EVENTS=1`).
- **DynamoDB roda numa JVM** (DynamoDB Local) dentro do container: é o que mais consome memória. Tabelas grandes pedem `mem_limit` maior.
- Fidelidade à AWS não é total: permissões IAM não são aplicadas por padrão, limites e erros podem diferir. Valide o comportamento crítico na AWS de verdade.

## Validação realizada

- `docker compose config -q` sem erros; container `healthy` e na `shared_net`.
- De um container separado na `shared_net` (boto3): S3 (criar bucket, gravar, ler, URL pré-assinada), SQS (enviar/receber), SNS → SQS (assinatura e entrega),
  DynamoDB (criar tabela, gravar, ler) e STS (`GetCallerIdentity`). Recursos de teste removidos.
- Do host via `localhost:4566`: health, S3 (criar bucket, gravar, ler, apagar) e SQS (criar/apagar fila).
- Memória: ~95 MB em repouso; ~310 MB com os quatro serviços em uso (30% do limite); `OOMKilled=false`.
- Reinício: dados perdidos, como esperado.
- **Não testado:** a credencial AWS com Custom Endpoints do n8n, AWS CLI e Terraform no host, e cargas grandes no DynamoDB.
