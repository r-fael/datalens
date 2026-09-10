# Publicar o DataLens pelo Traefik existente (VPS)

Expõe o frontend do DataLens em `https://datalens.r-fael.com.br` usando o
Traefik + Let's Encrypt que já rodam na VPS, sem tocar nas outras apps.

O DataLens continua um `docker compose` normal; o arquivo
`deploy/docker-compose.traefik.yml` é um override aplicado só na VPS.

## Por que uma rede nova

O Traefik só roteia para containers na rede que ele enxerga. A rede das apps
existentes (`rafaelnet`) é `attachable=false` — um container de compose não
entra nela. Em vez de recriar a `rafaelnet` (o que derrubaria n8n/portainer),
criamos uma rede overlay `attachable` dedicada (`datalens-proxy`) e ligamos o
Traefik nela. Nada da `rafaelnet` muda.

## Passo 1 — DNS no Cloudflare (você faz)

Registro `A`:
- Nome: `datalens`
- Conteúdo: IP da VPS
- Proxy: **desligado (nuvem cinza)** na primeira emissão do certificado.
  Depois de o HTTPS funcionar, pode religar (nuvem laranja) se quiser.

Motivo: o Let's Encrypt do Traefik valida por HTTP-01 na porta 80; a nuvem
laranja pode atrapalhar essa primeira validação.

## Passo 2 — rede overlay attachable (uma vez, na VPS)

```bash
docker network create --driver overlay --attachable datalens-proxy
```

## Passo 3 — conectar o Traefik à rede nova (uma vez)

Adicionar a rede ao serviço é um rolling update; não derruba as outras apps.

```bash
docker service update --network-add datalens-proxy traefik_traefik
```

Confirme que o Traefik voltou:

```bash
docker service ps traefik_traefik --format '{{.CurrentState}}' | head -1
```

## Passo 4 — configurar o `.env` da VPS (uma vez)

No `.env` da VPS, defina estas linhas (ver `.env.example`, seção "SERVIDOR COM
TRAEFIK"):

```bash
COMPOSE_FILE=docker-compose.yml:deploy/docker-compose.traefik.yml
ENVIRONMENT=production
FRONTEND_ORIGIN=https://datalens.r-fael.com.br
```

O `COMPOSE_FILE` faz o `docker compose` incluir o override do Traefik
automaticamente. **É o que evita o 404**: sem ele, um `docker compose up` puro
recria o frontend sem as labels/rede do Traefik e a rota some.

## Passo 5 — subir

```bash
cd /home/datalens/datalens
docker compose up -d --build
```

Com o `COMPOSE_FILE` no `.env`, não precisa passar `-f` — o override entra
sozinho. O frontend serve `/api` na mesma origem (proxy do nginx), então o CORS
quase nunca é acionado, mas manter `FRONTEND_ORIGIN` correto evita surpresa.

> Sem o `COMPOSE_FILE` no `.env`, use sempre os dois arquivos explicitamente:
> `docker compose -f docker-compose.yml -f deploy/docker-compose.traefik.yml up -d`

## Verificação

```bash
curl -I https://datalens.r-fael.com.br            # 200, certificado valido
docker logs $(docker ps -q -f name=datalens-frontend) --tail 5
```

## Reverter

```bash
cd /home/datalens/datalens
docker compose up -d                               # sobe sem as labels do Traefik
docker service update --network-rm datalens-proxy traefik_traefik
docker network rm datalens-proxy
```
