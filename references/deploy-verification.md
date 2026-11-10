# Deploy verification — pitfalls

## URLs: never guess

Do not infer frontend/backend URLs from repo names or patterns. Use one of:
- explicit user-provided URLs
- hosting config: `render.yaml` for Render services, Vercel project settings for web
- MCP provider tooling to list actual deploys/aliases
- **the published frontend bundle**: the API base is baked into the JS at build time
  (`grep -oE 'https://[A-Za-z0-9._-]+\.onrender\.com' bundle.js`, or grep for `VITE_API_URL`).
  This is the only source that reflects what production actually calls.

Guessing produces false negatives and wasted debugging time.

## Distinguish 404 / 503 / suspended / cold start

The status code alone does not tell you what is wrong. Read the body and the provider headers.

- **404 + `NOT_FOUND` page** = the route is not served on that host. Not "service down".
- **404 + `x-render-routing: no-server`** = the Render service is not answering as an app.
- **503 + `Service Suspended`** = the provider suspended the service (billing, quota, manual).
  No redeploy fixes it; it has to be reactivated in the provider panel. Do not report this as a
  cold start or a broken build. The header form of the same state is
  **`x-render-routing: suspend-by-user`**, and it **survives a Vercel rewrite**: a 503 from
  `<dominio>/api/...` carrying that header is proof the backend is suspended even when you have
  no way to reach the Render host directly. Read the response headers of the proxied call, not
  only of the health URL.
- **timeout / 504** = the usual cold start on free tiers.

Also **do not assume a 404 on `<name>.onrender.com` means the service does not exist** — the
provider's hostname may differ from the name in the blueprint (`name-api` vs `nameapi`). Try both
separators and the `-staging`/`-prod` suffixes, or read the real host out of the published bundle.

When the production domain serves 200 while every host of the blueprint returns 404 or
`Service Suspended`, production has **moved to another provider**: report that, and stop describing
the retired host as the deployment target. Repo files (`render.yaml`, RUNBOOK, E2E specs) keep
describing the old target until someone updates them.

## Backend vivo, banco morto

A API responde e ainda assim toda página de catálogo dá 500. Separe processo de dependência
antes de abrir log ou mexer em código — o padrão é sempre o mesmo:

```bash
for p in /health /api/auth/get-session /api/<recurso-com-banco> /api/<outro-com-banco>; do
  printf "%-28s " "$p"
  curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" -m 45 "https://<api>$p"
done
```

- **`/health` 200 + toda rota de dado 500** = processo de pé, banco inacessível: credencial
  expirada, projeto do banco pausado ou pool esgotado. Nenhum redeploy resolve; o log da API é o
  único que nomeia a causa (`P1001`/`P2021`/`FATAL: password authentication failed`).
- **Latência constante de 2–4 s em toda rota de dado** (e não só na primeira) é a assinatura de
  tentativa de conexão falhando — cold start seria um atraso único na primeira chamada e depois
  respostas rápidas.
- **Só as rotas de dado caem e a home responde 200** porque a home da web não faz fetch no
  servidor. Não conclua "só o catálogo quebrou": é a mesma API.
- `get-session` respondendo 200 **não** prova que o banco está bom em toda arquitetura — confirme
  com uma rota que comprovadamente lê tabela de domínio antes de descartar o banco.

O mesmo raciocínio vale para o banco de dados "de mentira" no relatório: se você não rodou uma
consulta, diga que a suspeita é suspeita e **peça o log**, em vez de escolher uma causa.

## SSO / platform security features

If a platform security feature (Vercel SSO, Render private services, IP allowlist) is enabled and works in other projects, do not propose disabling it as a debugging step. That is not a root-cause investigation; it is a security change with side effects.

## Health checks first

Always probe backend health before debugging frontend auth errors. A 404/401 from `/api/sessao` can be caused by:
- frontend calling wrong base URL
- backend down
- auth token missing/invalid

Check backend health (`/q/health/ready`) and API root first, then inspect frontend proxy/base URL config.

## Frontend on Vercel talking to Render backend

When the frontend is hosted on Vercel and the backend is on Render (or any external host), **Vercel does not automatically proxy `/api` to the backend**. The SPA must use an absolute API base URL in production builds.

Procedure:
1. Add `VITE_API_URL` to the frontend build config (Vercel env vars), pointing to the backend public URL.
2. In the frontend code, prefix all API calls with `VITE_API_URL` when it is defined; otherwise keep relative paths for dev/Docker.
3. Redeploy the frontend.

Without this, `/api/*` calls return 404 because Vercel serves the SPA static files and has no reverse proxy rule for the API path.

## Merge não gerou build — prove pelo artefato, não pelo painel

"Está mergeado" não é "está publicado". Um merge em `staging` que não dispara build
deixa a homologação servindo código antigo, e o PR verde esconde isso.

Duas evidências, independentes:

1. **O bundle publicado diz de qual commit ele é.** Builds Vite costumam carregar
   `<meta name="build-commit" content="<sha7>">` no `index.html` (injetado no
   `transformIndexHtml`). `curl -s <url> | grep -o 'build-commit" content="[^"]*'`
   devolve o sha servido agora. Compare com `git rev-parse --short origin/<branch>`.
2. **A lista de deployments do projeto.** Na Vercel, `GET
   https://api.vercel.com/v6/deployments?projectId=<id>&branch=<branch>` com o token do
   CLI (`%APPDATA%/com.vercel.cli/Data/auth.json`) devolve `meta.githubCommitSha`,
   `githubCommitRef` e `created` por deployment. Se o mais recente é de dias atrás, o
   push não chegou.

Antes de culpar o código, olhe o webhook: `gh api repos/<org>/<repo>/hooks` mostra para
onde os pushes vão. Um hook `push` apontando para um endpoint de deploy próprio (ex.:
`/deploy` de uma API em EC2) **não** é a integração Git do provedor de frontend — se não
existe hook da Vercel, ela nunca vê o push. Corrigir isso é painel/infra: reporte com as
duas evidências e não anuncie "publicado".

## Quarkus CORS configuration

**403 com corpo vazio = origem fora do CORS, não falta de permissão.** O sinal que separa
os dois: a mesma chamada feita pelo **navegador** devolve `403` sem corpo, enquanto o
mesmo método pelo `curl` (sem header `Origin`) devolve `201/200`. O filtro CORS do
Quarkus rejeita o preflight antes de a autorização rodar.

Antes de investigar token/permissão, leia a lista de origens:
`grep -n 'quarkus.http.cors' backend/src/main/resources/application.properties`. O
default costuma cobrir `localhost:5173` (vite dev) e **não** `localhost:8081` (nginx do
compose), que é justamente a URL que a stack E2E serve — então a suíte de API fica verde
e toda escrita da tela falha.

Corrija apontando a origem no compose de E2E via a env var que a property já lê
(`FRONTEND_CORS_ORIGINS=http://localhost:8081,...`). Não abra para `*` com
`access-control-allow-credentials=true` — a combinação é inválida e some do lado do
navegador.

Quarkus `quarkus.http.cors.*` properties may not take effect in some deployments, especially behind CDNs like Cloudflare/Render. If CORS headers are missing in preflight responses despite correct properties, use a programmatic `ContainerResponseFilter` instead.

Procedure:
1. Create a JAX-RS filter implementing `ContainerResponseFilter`.
2. In `filter()`, read the `Origin` header and set `Access-Control-Allow-Origin` to that value.
3. Also set `Vary: Origin`, allowed methods, allowed headers, exposed headers, and max age.
4. Register it with `@Provider`.

Properties-only CORS is not reliable on Render+Cloudfront; the programmatic filter is the working fallback.
