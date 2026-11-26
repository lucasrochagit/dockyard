# MinIO (simulador de AWS S3)

Armazenamento de objetos compatível com a API do S3, para testar fluxos que gravam/leem arquivos em buckets (por exemplo, a partir do n8n) sem usar a AWS.

| | |
|---|---|
| Compose | [`minio/docker-compose.yml`](../../minio/docker-compose.yml) |
| Imagem | `quay.io/minio/minio` (tag `latest`) |
| Container / host na rede | `minio_s3_simulator` / `minio` |
| Portas | `9000` (API S3), `9001` (console web) → http://localhost:9001 |
| Credenciais | `admin` / `admin1234` |
| Volume | `minio_data` (`/data`) |
| Recursos | `mem_limit: 512m`, `cpus: 1.0` |

## Conectar

| De onde | Endereço |
|---|---|
| Container / n8n | endpoint `http://minio:9000` |
| Seu computador | endpoint `http://localhost:9000`; console em http://localhost:9001 |

**No node S3 do n8n** (mesma API da AWS): Endpoint `http://minio:9000`, Access Key `admin`, Secret `admin1234`,
**Force Path Style** ligado, região qualquer (ex.: `us-east-1`; o MinIO ignora, mas o node exige).

Os buckets **não são criados automaticamente**: crie pelo console ou com o cliente `mc`:

```bash
docker run --rm --network shared_net --entrypoint sh quay.io/minio/mc:latest -c "
  mc alias set local http://minio:9000 admin admin1234 &&
  mc mb local/meu-bucket &&
  mc ls local"
```

(o `--entrypoint sh` é necessário: o entrypoint padrão da imagem do `mc` é o próprio `mc`.)

## Limitações e pontos de atenção

- **Imagem vem do Quay.io.** O `minio/minio` do Docker Hub não pode mais ser baixado (`pull access denied`); o compose usa `quay.io/minio/minio`.
- **Senha com no mínimo 8 caracteres** (usuário, no mínimo 3). Com `admin/admin` o container entra em loop de reinício com `Unable to validate credentials`.
  Por isso a senha é `admin1234`, e não `admin`.
- **Tag `latest`:** sem versão fixa, um `pull` pode trazer outra versão.
- **Nó único, disco único:** sem redundância (erasure coding) nem alta disponibilidade. Serve para teste de integração, não de desempenho ou durabilidade.
- **Sem TLS:** o endpoint é `http://`. Se alguma biblioteca exigir HTTPS, a conexão falha.
- **Path-style obrigatório na prática:** endereços do tipo `bucket.minio:9000` (virtual-hosted) não funcionam nesta rede; use `http://minio:9000/bucket/chave`.
- **URLs pré-assinadas** geradas dentro da rede Docker carregam o host usado na assinatura (`minio:9000`), que **não resolve no navegador do seu computador**.
  Se precisar abri-las no host, assine usando `http://localhost:9000` como endpoint.
- Leve: ~96 MB de memória mesmo após operações de bucket; 512 MB é folgado.

## Validação realizada

- Console (9001) e API (9000) respondendo; container na `shared_net`.
- Com o `mc` rodando em outro container da `shared_net`: criar bucket, enviar, listar, ler e remover objeto, e remover o bucket.
- **Não testado:** o node S3 do n8n em si.
