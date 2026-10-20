# Deploy do ChatbotX em VPS com EasyPanel

Guia para subir a stack usando `docker-compose.prod.yml`.

---

## 1. Requisitos

| Item | Mínimo | Recomendado |
|---|---|---|
| RAM | 4 GB | 8 GB |
| vCPU | 2 | 4 |
| Disco | 40 GB SSD | 80 GB SSD |
| SO | Ubuntu 22.04/24.04 | Ubuntu 24.04 |

> O README oficial cita 2 GB como baseline, mas com `builder` + `worker` +
> `realtime` + Postgres + Redis + RustFS no mesmo host, 2 GB causa OOM kill.
> Comece com 4 GB.

EasyPanel já instalado na VPS. Se ainda não estiver:

```bash
curl -sSL https://get.easypanel.io | sh
```

---

## 2. DNS

Crie três registros `A` apontando para o IP da VPS:

| Subdomínio | Aponta para | Serviço | Porta interna |
|---|---|---|---|
| `app.seudominio.com` | IP da VPS | `builder` | 3000 |
| `rt.seudominio.com` | IP da VPS | `realtime` | 1999 |
| `s3.seudominio.com` | IP da VPS | `filesystem` | 9000 |

Espere a propagação antes de emitir os certificados (`dig app.seudominio.com`).

---

## 3. Criar o projeto no EasyPanel

1. **Projects → Create Project** → nome: `chatbotx`.
2. Dentro do projeto: **+ Service → Compose**.
3. Nome do serviço: `stack`.

### 3.1 Fonte do código

Na aba **Source**, aponte para este repositório Git (ou cole o compose
manualmente na aba **Compose**).

Se usar Git, defina o **Compose File** como:

```
docker-compose.prod.yml
```

### 3.2 Variáveis de ambiente

Na aba **Environment**, cole o conteúdo do `.env.example` já com os valores
reais preenchidos. Gere os segredos assim:

```bash
echo "BETTER_AUTH_SECRET=$(openssl rand -base64 32)"
echo "ENCRYPTION_KEY=$(openssl rand -hex 32)"
echo "REALTIME_BROADCAST_SECRET=$(openssl rand -hex 32)"
echo "REALTIME_API_KEY=$(openssl rand -hex 24)"
echo "JAVASCRIPT_EXECUTOR_TOKEN=$(openssl rand -hex 24)"
echo "POSTGRES_PASSWORD=$(openssl rand -hex 24)"
echo "RUSTFS_SECRET_KEY=$(openssl rand -hex 24)"
```

> O compose falha o deploy com mensagem explícita se algum segredo
> obrigatório estiver faltando — isso é proposital.

**No primeiro deploy apenas**, use `RUN_DB_SEED=true`. Depois que o app
subir, volte para `false` e faça redeploy.

### 3.3 Deploy

Clique em **Deploy**. O primeiro boot baixa ~2 GB de imagens e roda as
migrações; pode levar 5–10 minutos.

---

## 4. Expor os domínios

Para cada serviço, abra **Service → Domains → Add Domain**:

| Serviço | Host | Porta | HTTPS | WebSocket |
|---|---|---|---|---|
| `builder` | `app.seudominio.com` | `3000` | ✅ | ✅ |
| `realtime` | `rt.seudominio.com` | `1999` | ✅ | ✅ **obrigatório** |
| `filesystem` | `s3.seudominio.com` | `9000` | ✅ | — |

O EasyPanel emite os certificados Let's Encrypt automaticamente.

**Não exponha** `postgres`, `redis`, `worker`, `javascript-executor` nem o
console do RustFS (porta 9001). Eles só precisam da rede interna.

---

## 5. Verificação

```bash
# Health do app
curl -I https://app.seudominio.com/api/health

# Bucket público do S3 criado
curl -I https://s3.seudominio.com/chatbotx/public/

# Logs
docker compose -p chatbotx logs -f builder worker
```

Depois abra `https://app.seudominio.com` e crie a conta de administrador.

---

## 6. Operação

### Atualizar para a última versão

As imagens usam a tag `latest`:

```bash
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

Ou clique em **Deploy** no EasyPanel (ele faz pull automático).

> Para produção séria, troque `:latest` por uma tag fixa de versão no
> `docker-compose.prod.yml` e atualize de forma deliberada.

### Backup do banco

```bash
docker compose -f docker-compose.prod.yml exec postgres \
  pg_dump -U chatbotx chatbotx | gzip > backup-$(date +%F).sql.gz
```

Agende com cron e envie para fora da VPS (S3, Backblaze, etc.).

Volumes a preservar: `db-data`, `filesystem-data`, `redis-data`.

### Adminer (debug de banco)

Fica no perfil `tools` e não sobe por padrão:

```bash
docker compose -f docker-compose.prod.yml --profile tools up -d adminer
```

Acesse via túnel SSH, nunca por domínio público:

```bash
ssh -L 8080:localhost:8080 usuario@ip-da-vps
```

Desligue quando terminar:

```bash
docker compose -f docker-compose.prod.yml --profile tools stop adminer
```

---

## 7. Problemas comuns

| Sintoma | Causa provável | Correção |
|---|---|---|
| `builder` reinicia em loop | OOM (pouca RAM) | Aumentar RAM ou reduzir `NODE_MAX_OLD_SPACE` |
| Interface não atualiza em tempo real | WebSocket desligado no domínio `rt` | Ativar WebSocket Support no domínio |
| Imagens/uploads quebrados (404) | `S3_DOMAIN` errado ou bucket sem permissão pública | Conferir `NEXT_PUBLIC_ASSET_URL` e rodar `filesystem-init` de novo |
| `password authentication failed` | `POSTGRES_PASSWORD` alterado após o volume existir | Resetar a senha dentro do container ou recriar o volume |
| E-mails não chegam | `SMTP_SERVER` inválido | Testar as credenciais fora da stack |
| Deploy falha citando variável | Segredo obrigatório ausente | Preencher a variável indicada no `.env` |

### Senha do Postgres dessincronizada

Se trocar `POSTGRES_PASSWORD` depois que o volume já existe, a variável é
ignorada (o Postgres só a usa na inicialização). Sincronize manualmente:

```bash
docker compose -f docker-compose.prod.yml exec postgres \
  psql -U chatbotx -c "ALTER USER chatbotx WITH PASSWORD 'nova-senha';"
```

---

## 8. Segurança

- [ ] Todos os segredos gerados com `openssl` — nenhum valor de tutorial reaproveitado
- [ ] `.env` fora do Git (já coberto pelo `.gitignore`)
- [ ] Firewall liberando apenas 22, 80 e 443 (`ufw allow 22,80,443/tcp`)
- [ ] `POSTGRES_LOG_STATEMENT=none`
- [ ] Adminer desligado quando não estiver em uso
- [ ] Backup automatizado e testado com restore real
- [ ] `javascript-executor` mantido na rede `internal: true`, sem porta publicada
