# Self-hosting no VPS (uso interno, single-workspace)

Guia para rodar este fork do Dub numa VPS barata, para uso interno de uma
empresa/pessoa só — sem as features de programa de parceiros, payouts,
bounties e billing (Stripe), que foram deixadas de fora da configuração.

Arquivos relevantes criados para isso:

- `apps/web/Dockerfile` — build de produção do app
- `docker-compose.production.yml` — app + MySQL + proxy PlanetScale (raiz do repo)
- `.env.production.example` — variáveis necessárias (copie para `.env.production`)
- `deploy/crontab.production` — crons essenciais, sem as features não usadas
- `deploy/Caddyfile` — proxy reverso com HTTPS automático e bloqueio do `/api/cron/*`

## 1. Domínio

Compre um domínio (Namecheap, Registro.br) e deixe pronto pra apontar o DNS.

## 2. Criar a VPS

Na Hetzner (https://www.hetzner.com/cloud): servidor **CX22** (2vCPU/4GB,
~€4/mês) é suficiente para uso interno de baixo tráfego. Ubuntu 24.04,
autenticação por SSH key.

Aponte o DNS do domínio (registro `A`) para o IP da VPS.

## 3. Preparar a VPS

```bash
ssh root@SEU_IP

# Docker
curl -fsSL https://get.docker.com | sh

# Caddy (proxy reverso com HTTPS automático)
apt update && apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install -y caddy
```

## 4. Clonar o repositório e configurar

```bash
git clone https://github.com/mayconmop-maker/dubfmg.git
cd dubfmg
cp .env.production.example .env.production
nano .env.production   # preencher os valores (ver seção abaixo)
```

### Contas gratuitas necessárias (free tier cobre uso interno)

- Upstash (Redis + QStash): https://upstash.com
- Resend (email): https://resend.com
- Tinybird (analytics): https://tinybird.co

Preencha as chaves geradas por esses serviços no `.env.production`.

`MYSQL_ROOT_PASSWORD` — gere uma senha forte qualquer, é usada só
internamente entre os containers.

## 5. Subir a aplicação

```bash
docker compose -f docker-compose.production.yml --env-file .env.production up -d --build
```

Isso builda a imagem (pode levar alguns minutos na primeira vez) e sobe:
app (porta 3000, só em `127.0.0.1`), MySQL, e o proxy PlanetScale interno.

Depois de subir, popular o schema do banco (só na primeira vez):

```bash
docker compose -f docker-compose.production.yml exec app sh -c \
  "cd /app/apps/web && pnpm prisma:push"
```

## 6. Configurar o proxy reverso (HTTPS)

```bash
cp deploy/Caddyfile /etc/caddy/Caddyfile
nano /etc/caddy/Caddyfile   # trocar pelo seu domínio
systemctl reload caddy
```

O Caddy emite certificado Let's Encrypt automaticamente assim que o DNS
estiver apontado. Depois disso, `https://seudominio` já funciona.

## 7. Configurar os crons

```bash
crontab deploy/crontab.production
```

⚠️ **Importante**: o código do Dub só valida o `CRON_SECRET` quando a
variável de ambiente `VERCEL=1` está presente (`apps/web/lib/cron/verify-vercel.ts`).
Fora da Vercel, essas rotas `/api/cron/*` ficam **sem autenticação**. Por
isso o `deploy/Caddyfile` bloqueia acesso externo a `/api/cron/*` — os
crons só rodam via `curl` local (`127.0.0.1`), nunca pelo domínio público.
Não pule essa parte do proxy reverso.

## 8. Backup do banco

O cron de backup interno da Dub Inc. (`trigger-backup`) foi removido do
`crontab.production` porque depende da infra deles. Configure o seu:

```cron
0 3 * * * docker compose -f /root/dubfmg/docker-compose.production.yml exec -T mysql \
  mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" dub > /root/backups/dub-$(date +\%F).sql
```

## Manutenção contínua (o que realmente precisa de atenção)

- `apt update && apt upgrade` periodicamente (ou ativar `unattended-upgrades`)
- Puxar atualizações do Dub de tempos em tempos: `git pull && docker compose -f docker-compose.production.yml up -d --build`
- Testar de vez em quando se o backup do passo 8 realmente restaura
- Um monitor externo grátis (UptimeRobot) avisando se o site cair

## O que foi deixado de fora de propósito

Como é uso interno de uma pessoa só, os seguintes recursos do Dub **não**
foram configurados: Stripe (billing), programa de parceiros/afiliados,
payouts, bounties, API de domínios da Vercel (domínio customizado é
configurado manualmente via DNS + Caddy, não pelo botão da UI).
