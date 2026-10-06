# Evolution API + Evolution Manager (Docker Desktop)

Stack local com Evolution API v2 (que já traz o Evolution Manager embutido), PostgreSQL e Redis.

---

## 🚀 Como Executar

Abra o terminal (PowerShell ou CMD) na pasta do projeto e execute:

```bash
docker compose up -d
```

Para acompanhar os logs em tempo real:

```bash
docker compose logs -f
```

Para parar todos os containers (os dados continuam nos volumes):

```bash
docker compose down
```

Para parar **e apagar todos os dados** (instâncias, mensagens e sessões do WhatsApp):

```bash
docker compose down -v
```

---

## 🌐 Serviços e Acessos

| Serviço | URL | Descrição |
| :--- | :--- | :--- |
| **Evolution Manager** | [http://localhost:8080/manager/](http://localhost:8080/manager/) | Dashboard visual: criar instâncias, ler QR Code, configurar integrações. Servido pela própria imagem da API (sem a barra final, `/manager` responde `301` para `/manager/`) |
| **Evolution API** | [http://localhost:8080](http://localhost:8080) | Servidor da API REST (a raiz responde um JSON de boas-vindas) |
| **Documentação dos endpoints** | [doc.evolution-api.com](https://doc.evolution-api.com) | Documentação oficial. **Não existe `/docs` nem Swagger nesta versão** — a API responde `404` em `/docs`, `/swagger` e `/openapi.json` |

---

## ✅ Verificação rápida

Com a stack no ar, estes são os resultados esperados. Uso `curl.exe` (já vem no Windows) em
vez de `Invoke-WebRequest` de propósito: este último **lança exceção** em respostas 4xx, o que
atrapalha justamente o teste do `401`.

```powershell
# A API está de pé? Deve devolver um JSON de boas-vindas
curl.exe -s -w '\n' http://localhost:8080
# {"status":200,"message":"Welcome to the Evolution API, it is working!","version":"2.3.7", ...}

# O Manager está sendo servido pela API? (note a barra no final)
curl.exe -s -o NUL -w '%{http_code}\n' http://localhost:8080/manager/                # 200

# A autenticação está sendo exigida? (401 sem a chave, 200 com ela)
curl.exe -s -o NUL -w '%{http_code}\n' http://localhost:8080/instance/fetchInstances # 401
$k = ((Select-String .env -Pattern '^AUTHENTICATION_API_KEY=').Line -split '=',2)[1]
curl.exe -s -o NUL -w '%{http_code}\n' -H "apikey: $k" `
  http://localhost:8080/instance/fetchInstances                                      # 200

# Todos os containers de pé?
docker compose ps
```

---

## 🔑 Como entrar no Manager

O Manager é um SPA compilado com Vite: variáveis `VITE_*` são resolvidas em **tempo de
build**, não em tempo de execução. Por isso o `docker-compose.yaml` não define
`VITE_EVOLUTION_API_URL` / `VITE_EVOLUTION_API_KEY` — elas não teriam efeito nenhum.

Abra [http://localhost:8080/manager/](http://localhost:8080/manager/). Se ele pedir a URL da
API, informe:

* **URL da API**: `http://localhost:8080` (a mesma origem que serve o `/manager`)
* **Global API Key**: o valor de `AUTHENTICATION_API_KEY` definido no arquivo `.env`

Esses dados ficam salvos no `localStorage` do navegador, então você só digita uma vez — e é
por isso que uma URL ou chave antiga salva ali pode fazer o Manager falhar mesmo com a stack
saudável (veja "Problemas comuns").

---

## 🗄️ Credenciais do banco de dados

| Item | Valor |
| :--- | :--- |
| Banco | `evolution` |
| Usuário | `evolution` |
| Senha | `evolution_pass` |
| Host (de dentro do Docker) | `evolution-postgres:5432` |
| Volume | `postgres_data` |

O Postgres e o Redis **não** publicam portas no host — só são alcançáveis pela rede
`evolution-net`. Para abrir o banco num cliente gráfico (DBeaver, pgAdmin), descomente o
bloco `ports` do serviço `evolution-postgres` no `docker-compose.yaml`.

---

## 📦 Versões das imagens

As versões estão fixadas de propósito. Com a tag `latest`, um `docker compose pull` pode
pular para uma major nova e quebrar a stack sem aviso.

| Serviço | Imagem |
| :--- | :--- |
| `api` | `evoapicloud/evolution-api:v2.3.7` |
| `evolution-postgres` | `postgres:15-alpine` |
| `redis` | `redis:8-alpine` |

Para trocar a versão da API, defina `EVOLUTION_API_IMAGE` no `.env` (a linha já existe
comentada) e rode `docker compose up -d`. Não é preciso editar o compose.

> [!WARNING]
> **Cuidado com downgrade.** A tag `latest` do `evoapicloud/evolution-api` é um build
> **posterior** à `v2.3.7` (linha `2.4.0-rc`). O Prisma não desfaz migrações: se o volume
> `postgres_data` já foi migrado por esse build, voltar para `v2.3.7` pode fazer a API
> falhar na subida, e você teria que recriar o ambiente com `docker compose down -v` e
> reescanear o QR Code de todas as instâncias. Nesse caso, mantenha a versão anterior em
> `EVOLUTION_API_IMAGE`.

---

## 🔒 Segurança

* **Troque a `AUTHENTICATION_API_KEY`** antes de expor esta stack para qualquer rede. Ela é
  a chave mestra: quem a tiver controla todas as instâncias de WhatsApp.
* O `.env` está no `.gitignore`. Não versione esse arquivo nem cole a chave em README,
  prints ou issues.
* `CORS_ORIGIN` fica em `*` de propósito. Restringir a uma origem faz o middleware de
  CORS recusar toda requisição **sem** header `Origin` (curl, webhooks, chamadas do n8n)
  com `500 Not allowed by CORS`. Quem protege a API é a `AUTHENTICATION_API_KEY`,
  não o CORS.
* `AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES=true` devolve a API Key nas respostas de
  `/instance/fetchInstances`. Aceitável em local; use `false` se for expor a API.
* Postgres e Redis não publicam portas no host. O Redis sobe sem senha, mas só é alcançável
  pela rede interna `evolution-net`.

---

## 🔧 Problemas comuns

| Sintoma | Causa provável / solução |
| :--- | :--- |
| Manager funciona em aba anônima mas não na aba normal | `localStorage` com URL ou API Key antigas. `F12` → *Application* → *Local Storage* → `http://localhost:8080` → apague as entradas, ou limpe os dados do site. Se não resolver, `Ctrl+Shift+R` para o cache HTTP |
| Manager abre, mas não lista instâncias | Falta informar a URL e a API Key na tela de login |
| Manager mostra erro logo após o `up` | A API ainda está subindo e migrando o banco; aguarde ~30s e recarregue |
| `500 {"message":"Not allowed by CORS"}` em `curl` ou chamadas do n8n | O `CORS_ORIGIN` foi restrito a uma origem. Volte para `CORS_ORIGIN=*` — sem wildcard, requisições sem header `Origin` são recusadas |
| `evolution_redis` sai com `Error dependency redis failed to start` | A imagem do Redis é mais **antiga** que o formato de dados gravado no volume. Use a major que criou o volume (hoje `redis:8-alpine`) ou apague só esse volume: `docker compose down`, depois `docker volume ls` para achar o nome (`<pasta>_evolution_redis`) e `docker volume rm <nome>`. O Redis aqui é cache descartável (`CACHE_REDIS_SAVE_INSTANCES=false`), então não há perda relevante |
| QR Code não aparece | Veja os logs com `docker compose logs -f api` e confira `LOG_BAILEYS`. A causa mais comum é sessão Baileys corrompida — apague a instância e crie de novo |
| `api` reinicia em loop | Verifique se `postgres_data` foi migrado por uma versão mais nova que a fixada em `EVOLUTION_API_IMAGE` |

---

## 📄 Variáveis de ambiente

A referência completa fica em [`.env.example`](.env.example) (template versionável) e a
lista oficial em
[docs.evolutionfoundation.com.br](https://docs.evolutionfoundation.com.br/evolution-api/configuration/env).

---

## 📎 Apêndice: usar o Manager como container separado

O `/manager` embutido na imagem da API dispensa o container `frontend`, que por isso foi
removido do compose. Se algum dia você precisar do Manager standalone (por exemplo para
apontá-lo para uma API em outro host), o serviço seria:

```yaml
  frontend:
    container_name: evolution_frontend
    image: evoapicloud/evolution-manager:v2.3.5
    restart: always
    depends_on:
      - api
    ports:
      - "3000:80"
    # OBRIGATÓRIO na imagem 2.3.5: o nginx.conf dela traz, na linha 11,
    #   gzip_proxied expired no-cache no-store private must-revalidate auth;
    # e `must-revalidate` NÃO é valor válido para `gzip_proxied` (os válidos são
    # off, expired, no-cache, no-store, private, no_last_modified, no_etag, auth
    # e any). Sem remover o token, o nginx aborta com
    #   [emerg] invalid value "must-revalidate" in /etc/nginx/conf.d/nginx.conf:11
    # e o container entra em loop de reinício. O upstream já corrigiu isso no
    # branch main (a linha virou "... private auth;"), então este override pode
    # sair quando existir uma imagem com a correção.
    entrypoint: ["/bin/sh", "-c", "sed -i 's/ must-revalidate//g' /etc/nginx/conf.d/nginx.conf && exec /start.sh"]
    networks:
      - evolution-net
```

Acesse em [http://localhost:3000](http://localhost:3000) — o `localStorage` dele é de outra
origem, então a URL e a chave precisam ser informadas de novo.

> [!NOTE]
> **Ao remover esse serviço de novo**, rode `docker compose up -d --remove-orphans` uma vez
> para o Docker apagar o container `evolution_frontend`, que passa a ficar órfão. Se você
> chegou neste projeto depois de o `frontend` já ter sido removido, ignore esta nota.
