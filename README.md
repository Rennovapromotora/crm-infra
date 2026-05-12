# crm-infra

Orquestração Docker para subir juntos o **backend** ([crm-api](../crm-api), Laravel) e o **frontend** ([crm-rennova](../crm-rennova), React + Nginx estático).

Organização esperada (três repositórios irmãos):

```text
rennova/
  crm-api/
  crm-rennova/
  crm-infra/          ← este diretório
```

Os caminhos padrão são `../crm-api` e `../crm-rennova`; dá para sobrescrever com `CRM_API_PATH` e `CRM_WEB_PATH` no arquivo de ambiente.

## Desenvolvimento

1. Copie o exemplo de variáveis: `cp .env.development.example .env`
2. Suba os serviços: `docker compose --env-file .env up -d --build`
3. Rode as migrações uma vez: `docker compose exec api php artisan migrate`
4. Acesse:
   - API: `http://localhost:8000` (rotas JSON em `/api/...`)
   - Front: `http://localhost:8082` (porta configurável por `WEB_HTTP_PORT`)

O build do React usa `REACT_APP_API_URL`; em dev o padrão é `http://localhost:8000/api`, alinhado ao `BASE_URL` do axios em `crm-rennova`.

## Produção (`crm.rennovafgts.com.br` / `api.rennovafgts.com.br`)

1. Copie `cp .env.prod.example .env.prod` e preencha senhas reais (`APP_KEY`, MySQL, Redis, SMTP, Facta).
2. Gere/obtenha certificados TLS e copie os arquivos para `certs/` conforme `certs/LEIA-ME.txt`:
   - `crm-fullchain.pem` / `crm-privkey.pem`
   - `api-fullchain.pem` / `api-privkey.pem`
3. Aponte DNS dos dois hosts para o IP do servidor.
4. Suba:  
   `docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --build`
5. Migrações:  
   `docker compose --env-file .env.prod -f docker-compose.prod.yml exec api php artisan migrate --force`

### O que o stack de produção faz

- **MySQL** e **Redis** (com senha) só na rede interna do Compose.
- **API** e **web** também só na rede interna; apenas o **edge** (Nginx) publica portas **80** e **443**.
- **`TRUST_PROXY=true`** na API para que URLs e HTTPS atrás do proxy estejam corretos.
- **`FRONTEND_URL`** e **`APP_URL`** alinhados aos domínios públicos para CORS (config `FRONTEND_URL` no Laravel) e consistência geral.
- **`REACT_APP_API_URL=https://api.rennovafgts.com.br/api`** no build do front (valor injetado no bundle em tempo de build).

Renovação de certificados (Certbot): use o webroot em `/.well-known/acme-challenge/` (volume `certbot_webroot` no serviço `edge`) ou pause o edge, rode o modo standalone do Certbot e volte a copiar os PEMs para `certs/`.

## Inicializar repositório Git

```bash
cd crm-infra
git init
git add .
git commit -m "Infra Docker dev e produção para CRM"
```

Não versionar `.env`, `.env.prod` nem arquivos `*.pem` em `certs/` (já ignorados no `.gitignore`).
