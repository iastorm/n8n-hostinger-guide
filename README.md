# Guia de Implantação do n8n em Modo Fila no Hostinger com Docploy

Este guia fornece instruções detalhadas para implantar o n8n em modo fila (queue mode) em um servidor Hostinger usando a ferramenta Docploy, conectando a um domínio gerenciado pela Cloudflare. O Traefik será usado como proxy reverso e gerenciará os certificados SSL automaticamente.

## Pré-requisitos

- Conta ativa no Hostinger com acesso ao painel de controle.
- Servidor dedicado ou VPS no Hostinger com Docker instalado.
- Domínio registrado e gerenciado pela Cloudflare.
- Acesso ao Docploy no painel do Hostinger.
- Conhecimento básico de Docker e Docker Compose.

## Passo 1: Configuração do Domínio na Cloudflare

1. Acesse o painel da Cloudflare e selecione o domínio desejado.
2. Vá para a seção **DNS** e adicione um registro A apontando para o IP do seu servidor Hostinger:
   - **Tipo**: A
   - **Nome**: seu-subdominio (ex: n8n)
   - **Conteúdo**: IP do servidor Hostinger
   - **Proxy status**: DNS only (não use proxy da Cloudflare para evitar conflitos com Traefik)
3. Adicione outro registro para o dashboard do Traefik se desejar:
   - **Tipo**: A
   - **Nome**: traefik
   - **Conteúdo**: Mesmo IP
   - **Proxy status**: DNS only

Aguarde a propagação do DNS (pode levar até 24 horas, mas geralmente é rápido).

## Passo 2: Preparação do Ambiente no Hostinger

1. Acesse o painel do Hostinger e navegue até o Docploy.
2. Crie um novo projeto no Docploy.
3. Configure o repositório ou faça upload dos arquivos necessários (docker-compose.yml e .env).

## Passo 3: Configuração das Variáveis de Ambiente

Crie um arquivo `.env` no diretório do projeto com as seguintes variáveis. Este arquivo contém todas as configurações necessárias para o n8n, PostgreSQL, Redis e Traefik.

```env
# =======================================
# Configurações de Domínio
# =======================================
DOMAIN_NAME=seu-dominio.com
SUBDOMAIN=n8n
# Não é necessário SSL_EMAIL pois o Traefik gerencia automaticamente

# =======================================
# Credenciais do Banco de Dados PostgreSQL
# Use senhas fortes em produção!
# =======================================
POSTGRES_USER=admin
POSTGRES_PASSWORD=sua_senha_forte_postgres
POSTGRES_DB=n8n

# =======================================
# Chave de Criptografia do n8n (CRÍTICO)
# Gere uma nova chave segura para produção
# =======================================
N8N_ENCRYPTION_KEY=gere_uma_chave_segura_aqui

# =======================================
# Configurações do n8n e Modo Fila
# =======================================
EXECUTIONS_MODE=queue
GENERIC_TIMEZONE=America/Sao_Paulo

# =======================================
# Limpeza Automática de Dados de Execuções
# =======================================
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=4320
EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL=1440
EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL=1440

# =======================================
# Configuração do Redis
# Use uma senha forte em produção!
# =======================================
QUEUE_BULL_REDIS_PASSWORD=sua_senha_forte_redis

# =======================================
# Configurações do Node.js
# =======================================
NODE_FUNCTION_ALLOW_BUILTIN=crypto

# =======================================
# Autenticação do Traefik (opcional, para dashboard)
# Gere com: echo $(htpasswd -nb usuario senha)
# =======================================
TRAEFIK_AUTH_USERS=usuario:$2y$10$hash_gerado
```

**Importante**: 
- Substitua `seu-dominio.com` pelo seu domínio real.
- Gere uma nova `N8N_ENCRYPTION_KEY` segura (pode usar `openssl rand -hex 32`).
- Use senhas fortes para PostgreSQL e Redis.

## Passo 4: Arquivo docker-compose.yml

Use o seguinte arquivo `docker-compose.yml` adaptado para o ambiente Hostinger. Este arquivo configura o Traefik, PostgreSQL, Redis e os serviços do n8n em modo fila.

```yaml
services:
  traefik:
    image: "traefik:v3.5"
    restart: always
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.websecure.address=:443"
      - "--api.dashboard=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=websecure"
      - "--certificatesresolvers.letsencrypt.acme.email=${SSL_EMAIL:-admin@exemplo.com}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
    ports:
      - "80:80"
      - "443:443"
    networks:
      - n8n-network
    volumes:
      - traefik_data:/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`traefik.${DOMAIN_NAME}`)"
      - "traefik.http.routers.api.service=api@internal"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"
      - "traefik.http.routers.api.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=${TRAEFIK_AUTH_USERS}"

  postgres:
    image: postgres:17
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:8
    restart: always
    command: redis-server --requirepass ${QUEUE_BULL_REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - n8n-network

  n8n-main:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - EXECUTIONS_MODE=${EXECUTIONS_MODE}
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${QUEUE_BULL_REDIS_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      - N8N_PROXY_HOPS=1
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - NODE_ENV=production
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - NODE_FUNCTION_ALLOW_BUILTIN=${NODE_FUNCTION_ALLOW_BUILTIN}
      - OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true
      - EXECUTIONS_DATA_PRUNE=${EXECUTIONS_DATA_PRUNE}
      - EXECUTIONS_DATA_MAX_AGE=${EXECUTIONS_DATA_MAX_AGE}
      - EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL=${EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL}
      - EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL=${EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL}
    networks:
      - n8n-network
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

  n8n-worker:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    command: worker
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - EXECUTIONS_MODE=${EXECUTIONS_MODE}
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${QUEUE_BULL_REDIS_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      - N8N_PROXY_HOPS=1
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - NODE_ENV=production
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - NODE_FUNCTION_ALLOW_BUILTIN=${NODE_FUNCTION_ALLOW_BUILTIN}
      - OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true
      - EXECUTIONS_DATA_PRUNE=${EXECUTIONS_DATA_PRUNE}
      - EXECUTIONS_DATA_MAX_AGE=${EXECUTIONS_DATA_MAX_AGE}
      - EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL=${EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL}
      - EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL=${EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL}
    networks:
      - n8n-network
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - n8n-main

  n8n-webhook:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    command: webhook
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - EXECUTIONS_MODE=${EXECUTIONS_MODE}
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_BULL_REDIS_PASSWORD=${QUEUE_BULL_REDIS_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      - N8N_PROXY_HOPS=1
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - NODE_ENV=production
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - NODE_FUNCTION_ALLOW_BUILTIN=${NODE_FUNCTION_ALLOW_BUILTIN}
      - OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true
      - EXECUTIONS_DATA_PRUNE=${EXECUTIONS_DATA_PRUNE}
      - EXECUTIONS_DATA_MAX_AGE=${EXECUTIONS_DATA_MAX_AGE}
      - EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL=${EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL}
      - EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL=${EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL}
    networks:
      - n8n-network
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - n8n-main

networks:
  n8n-network:
    driver: bridge

volumes:
  n8n_data:
  postgres_data:
  redis_data:
  traefik_data:
```

**Notas sobre o docker-compose.yml**:
- Removido o serviço `cloudflare-ddns` pois não é necessário no Hostinger.
- Configurado o Traefik para usar Let's Encrypt automaticamente para certificados SSL.
- Adicionado `certificatesresolvers` para gerenciar certificados.

## Passo 5: Implantação via Docploy

1. No painel do Docploy, faça upload ou conecte o repositório com os arquivos `docker-compose.yml` e `.env`.
2. Configure as variáveis de ambiente no Docploy (ou use o arquivo .env).
3. Inicie a implantação.
4. Monitore os logs para garantir que todos os serviços iniciem corretamente.

## Passo 6: Verificação da Implantação

1. Acesse `https://n8n.seu-dominio.com` para o n8n.
2. Acesse `https://traefik.seu-dominio.com` para o dashboard do Traefik (se configurado).
3. Verifique se os certificados SSL estão funcionando (ícone de cadeado no navegador).

## Troubleshooting

### Problemas Comuns

1. **Certificados SSL não são emitidos**:
   - Verifique se as portas 80 e 443 estão abertas no firewall do Hostinger.
   - Certifique-se de que o domínio aponta corretamente para o IP do servidor.

2. **n8n não consegue conectar ao banco de dados**:
   - Verifique as credenciais no arquivo `.env`.
   - Aguarde o healthcheck do PostgreSQL.

3. **Erro de autenticação no Redis**:
   - Confirme a senha `QUEUE_BULL_REDIS_PASSWORD`.

4. **Traefik não roteia corretamente**:
   - Verifique os labels no docker-compose.yml.
   - Certifique-se de que o domínio está resolvendo para o servidor.

### Logs Úteis

- Para ver logs do Traefik: `docker logs traefik`
- Para ver logs do n8n: `docker logs n8n-main`
- Para ver logs do PostgreSQL: `docker logs postgres`

### Comandos Úteis

- Reiniciar serviços: `docker-compose restart`
- Ver status: `docker-compose ps`
- Atualizar imagens: `docker-compose pull`

Se encontrar problemas específicos, consulte a documentação oficial do n8n, Traefik e Hostinger.