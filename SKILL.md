---
name: rocketify-rules
description: >
  Regras de commit e Trello da Rocketify. SEMPRE usar antes de criar commit, squash,
  rebase ou atualizar cards no Trello em qualquer projeto.
  Use quando o usuário pedir para commitar, gerar mensagem de commit, corrigir autoria
  de commits, ou atualizar/mover cards, checklists ou logs no Trello.
  Aplica-se a todos os projetos Rocketify (VisitaPro, SocialRanking, Olfatto e novos).
---

# Regras Rocketify — commits e Trello

Empresa: **Rocketify**. Cada sistema tem seu próprio domínio de e-mail.

## 1. Autoria de commit

**O autor do git é SEMPRE o humano dono do repositório.** A função do agente vai num trailer
`Agent:` no corpo do commit, não no e-mail.

| Nome (git author) | E-mail |
|---|---|
| `André Buzzo` | `1138225+andrelbuzzo@users.noreply.github.com` |

```powershell
git commit -m "<tipo>(<escopo>): descrição" -m "corpo do commit

Agent: backend"
```

`Agent:` aceita exatamente um destes 9 valores:

| `Agent:` | Quando usar |
|---|---|
| `gerente` | coordenação, escopo, decisão de produto, priorização |
| `product-manager` | requisitos, backlog, critérios de aceite, escopo de card |
| `tech-lead` | arquitetura, **modelagem de dados**, **segurança**, decisões técnicas |
| `backend` | regras de negócio, API, persistência, integrações |
| `frontend` | telas, componentes, estado, chamadas de API |
| `ui-ux-designer` | design system, tokens, acessibilidade, copy |
| `devops` | infra, deploy, CI/CD, variáveis de ambiente, observabilidade |
| `qa` | testes automatizados, matriz de casos, verificação de correção |
| `revisor` | revisão de código, Simplificação, checagem antes do merge |

**Modelagem de dados e segurança vão para `tech-lead`** — não criar `data-arch` nem
`security`. São decisões técnicas, não agentes separados.

### Por que o e-mail do agente não pode ser o autor

Plataformas de hosting (Vercel Hobby) **bloqueiam o deploy** quando o e-mail do autor do commit
não resolve para uma conta do GitHub com acesso ao projeto:

> The deployment was blocked because the commit email `<email>` could not be matched to a
> GitHub account.

Isso vale para **qualquer** commit alcançável, não só os de `master`: o gate roda na criação do
deployment, antes de qualquer build step. Consequências práticas:

- O gate lê o commit que **dispara** o build — inclusive o push de branch, que dispara preview
  deployment. Por isso "só o master está limpo" **não** basta.
- `Ignored Build Step` **não resolve**. Ele pula o *build*, mas o commit já foi rejeitado antes.
  Pior: depois do autor do git corrigido, ele só serve pra branch não deployar, engolindo
  deploys em silêncio. **Não configure** — a raiz (autor do git) já basta. Se algum dia o consumo
  de builds/minutos do Hobby apertar, use Project Settings → Git → Deployments → desmarcar
  branches, que mexe só em preview e é mais explícito.
- `Co-authored-by:` também não resolve: o Vercel lê o campo de autor, não o trailer.

Um e-mail `agent-<função>@<dominio>` que não seja conta do GitHub sempre falha no gate. O trailer
`Agent:` preserva a autoria por função no histórico sem expor o e-mail.

### Nunca dar amend + force-push em branch já pushada

O commit antigo fica órfão (nenhuma branch o referencia) mas continua vivo porque o PR o cita.
O Vercel tenta validá-lo e falha com:

> This commit does not belong to any branch on this repository, and may belong to a fork
> outside of the repository.

Se errar a mensagem depois de pushar, **commit novo**. Nunca `git commit --amend` + `force-push`.

**Antes do primeiro push, `--amend` é o caminho certo** — a proibição acima é sobre reescrever
histórico já publicado, não sobre acertar o commit recém-criado. É comum montar um corpo longo e
esquecer o trailer `Agent:`: confira com `git log -1 --format='%B' | tail -3` **antes** de
`git push`, dê `--amend` com `-c user.name=... -c user.email=...` e só então empurre. Depois do
push, o mesmo erro vira commit novo.

**O trailer entra na primeira tentativa.** Escreva o corpo com o `Agent:` já no fim em vez de
"adicionar depois": quando o commit sai sem ele e a branch já foi publicada, o conserto custa um
commit extra que polui o squash.

### Fluxo de branches: `staging` como integrador

`staging` é a branch de integração contínua e `master` é produção. **`master` nunca
recebe conteúdo direto** — ele só recebe o que já passou por `staging`.

```
feature/<slug>  →  staging  →  master
fix/<slug>      ↗
```

1. `staging` fica **sempre viva**: recebe PR de toda `feature/` e `fix/` nova e é
   deployada (homologação) a cada merge.
2. `master` só recebe o conteúdo de `staging` **depois de os testes passarem em
   homologação**: PR `staging → master`, squash merge.
3. Nunca commitar direto em `master` e nunca abrir PR de `feature/` direto para `master`,
   **exceto** no caso de recuperação do item 3a.
3a. **Quando `master` e `staging` divergiram muito e `master` é a base mais nova, reconstrua
   `staging` a partir de `master` em vez de tentar reconciliar.** É o inverso do fluxo normal e
   precisa de autorização explícita do usuário, mas é o caminho certo quando o histórico já
   furou a regra: se commits foram mergeados direto em `master`, `staging` fica com trabalho
   antigo e o próximo `PR staging → master` traz código velho de volta — regressão silenciosa.
   ```
   git push origin master:staging        # ou --force-with-lease se staging tem commits próprios
   ```
   Antes de descartar `staging`, prove que o conteúdo dele já está em `master` — três checagens,
   porque cada uma sozinha mente:
   - `git cherry -v origin/master origin/staging` marca `+` em quase tudo quando o histórico
     divergiu ("patch não idêntico" ≠ "trabalho perdido"); use só para achar candidatos;
   - `gh pr list --state all --head <branch>` — PR `MERGED` cobre o conteúdo;
   - `git grep -c '<símbolo>'` nas duas pontas para cada feature candidata.
   Na prática a diferença real costuma ser **uma ou duas linhas** (ex: um método que só existe
   em `staging` e que o `master` já chama, por isso não compila). Liste-as ao usuário antes de
   descartar, mesmo quando ele já autorizou: é a última chance de descobrir o que se perde.
4. `staging` **nunca** é deletada nem arquivada — é reutilizada pelo próximo ciclo.
5. Um push em `staging` ou em `master` dispara deploy de homologação/produção; por isso
   o autor do git tem que estar limpo em **todas** as branches que deployam.
6. Quando `feature/` e `staging` divergem (ex: `staging` recebeu squash enquanto
   `feature/` avançou), faça o merge de `staging` na `feature/` para consolidar o
   trabalho antes de abrir/atualizar o PR. Conflitos são normais nesse cenário:
   resolva ficando com o código mais novo e commit a junção.
6a. **Audite a base ANTES de commitar ou abrir PR — nos dois sentidos.** Um PR contra
   `staging` que na verdade parte do `master` leva 40 commits de produção junto, e um
   PR contra `master` que parte de `staging` desfaz homologação. Rode, sempre:
   ```
   git rev-parse --short HEAD
   git log --oneline origin/staging..origin/master | wc -l   # o que staging NAO tem
   git log --oneline origin/master..origin/staging | wc -l   # o que master NAO tem
   git merge-base origin/master origin/staging
   git show origin/<base>:<arquivo-tocado> | head -40          # a base tem isto?
   ```
   Os números contando **nos dois sentidos** são o sinal: 0 e 0 é fast-forward limpo;
   40 e 16 é divergência real, e o PR precisa ser decidido com o usuário, não escolhido
   sozinho. Confira também se o arquivo que você está criando **já existe** na base
   (`git ls-tree -r origin/<base> --name-only | grep <Classe>`): criar um `Service` que
   a base não tem, num PR que também toca essa área, duplica feature por caminho torto.
6b. **Verifique que a BASE compila, não só a sua working tree.** Working tree verde não
   prova nada sobre o destino do PR. Num worktree descartável:
   ```
   git worktree add <scratch>/check origin/<base> --detach
   cd <scratch>/check/backend && <mvn> -q -DskipTests compile
   git worktree remove <scratch>/check --force
   ```
   Base que não compila (ex: `cannot find symbol` num repositório Spring Data cujo método
   o service já chama) é **bloqueio**, não detalhe: commitar em cima dela propaga o erro e
   quebra o próximo deploy. Reporte ao usuário com a saída do compilador e a divergência
   medida, e ofereça corrigir a base primeiro — não escolha entre "melhor em staging" e
   "melhor em master" sozinho, porque essa escolha mexe em produção.
7. Quando `master` está atrás de `staging` e precisa receber o conteúdo, **não use
   `--ff-only`**. Faça `git merge staging --no-edit` local e push; se o remote
   aceitar fast-forward melhor, se não, o merge commit resolve. `git pull
   --ff-only` em `master` falha quando há divergência; use `git pull --rebase=false`
   ou merge direto.
8. Ao verificar deploy, nunca use URLs por adivinhação. Consulte primeiro: usuário,
   `render.yaml`, Vercel project settings ou MCP do provedor. 404 em URL errada não
   significa que o deploy falhou; significa que você está batendo no serviço errado.
9. Se uma configuração de plataforma (SSO do Vercel, IP allowlist, MFA do provedor)
   está ativa e funciona em outros projetos do usuário, **não proponha desativá-la**
   como passo de depuração. Isso é uma mudança de segurança com efeitos colaterais,
   não investigação de causa raiz.
10. **Deploys não dependem de Actions.** Se a quota de E2E estourar e os workflows
   forem cancelados, isso não bloqueia deploy de `staging` ou `master`. O usuário
   pode pedir para não rodar testes via Actions; deploys continuam normais via
   Vercel/Render. Não reative CI nem adie deploy por causa de Actions cancelados
   sem confirmação explícita do usuário.

### Audite o REMOTE antes do primeiro push — não só a branch

O fluxo acima assume que o repositório já tem remote e que ele aponta para a organização.
As duas coisas falham, e falham em silêncio:

```bash
git remote -v          # vazio? não existe upstream — o fluxo inteiro não roda
gh api user/orgs --jq '.[].login'                    # quais orgs o token realmente vê
gh repo view <Org>/<repo> --json name,owner 2>&1     # o repositório existe lá?
```

`gh auth status` verde **não** garante que o `git` use o token: `git fetch`/`git push` falham com
`could not read Username for 'https://github.com'` e o erro parece credencial faltando. Rode
`gh auth setup-git` uma vez (registra o credential helper do `gh` no git) e repita o comando —
não troque o remote para SSH nem invente token em URL.

**`git remote -v` vazio não é erro de configuração local — é repositório que nunca foi
publicado.** Não invente um URL, não empurre para `origin` adivinhado, e não crie sob a conta
pessoal por conveniência: o produto é da organização, e mover depois significa reescrever
webhooks, secrets e permissões. Crie no lugar certo desde o começo:

```bash
gh repo create <Org>/<nome> --private --description "..." --source=. --remote=origin
git push -u origin master          # publica o histórico existente ANTES de ramificar
git checkout -b <tipo>/<slug>      # a branch nova nasce depois que a base existe no remoto
```

`--source=. --remote=origin` numa pasta que já é repositório Git preserva commits e autoria;
`gh repo create` sem isso tenta inicializar do zero. O `push` de `master` publicando o
histórico **já existente** não viola "nunca commitar direto em master" — é a base que o PR
precisa para ter contra o quê ser aberto. Confirme depois: `git remote -v` deve mostrar a
organização, e o merge commit deve trazer `author: <dono>` + `committer: GitHub` (sinal de
squash pelo web).

**`staging` pode não existir.** Quando o repositório só tem `master` (`git ls-remote --heads
origin`), o ciclo `feature → staging → master` não tem onde ser aplicado. Nesse caso siga a
instrução explícita do usuário (branch dedicada → PR → master), **diga no relatório que
`staging` não existe e que o fluxo completo não foi aplicado**, e não crie `staging` por conta
própria no meio da entrega — criar a branch de integração é decisão de processo, não passo de
uma tarefa.

Se o usuário **pedir** para mirar em `staging`, ele autorizou o processo: crie a branch a partir
de `master` **sem trocar de checkout** e retargete o PR que já existe.

```bash
git branch staging origin/master        # NÃO `checkout -b` — veja abaixo
git push -u origin staging
gh pr edit <n> --base staging           # retarget, não um PR novo
```

**`git checkout -b staging origin/master` pode esvaziar a árvore de trabalho.** Em repositório
onde `master` tem um punhado de arquivos e o projeto vive todo na feature, trocar para a branch
nova apaga o código do disco; as leituras seguintes falham com "No such file or directory" e a
depuração mira o problema errado. Crie a branch sem checkout e confirme
`git branch --show-current` antes de retomar o trabalho.

### Dois PRs em paralelo que tocam a mesma fonte de verdade quebram só na composição

Cada PR passa sozinho e o CI pós-merge fica vermelho. Caso real: um PR promoveu `gcp/api/restart`
de `unverified` para `supported` na matriz de capabilities; outro PR, mergeado depois, usava
esse mesmo par como exemplo de capability `unverified` nos testes de recusa. Com a matriz
atualizada o teste passou a receber `201` onde esperava `422`.

Ninguém errou sozinho — **o defeito é o teste emprestar um fato vivo como fixture**. Ao escrever
teste que consulta uma fonte de verdade compartilhada (matriz, registry, enum de permissão):

- escolha um caso que seja **estruturalmente** estável (ausente do modelo, provider inexistente),
  não um que uma mudança de estado legítima possa promover;
- deixe um comentário dizendo por que aquele caso é um gap de propósito, senão a próxima pessoa
  "conserta" promovendo.

Quando rodar subagentes em worktrees paralelos, o segundo a mergear precisa de um `git pull origin
staging` **e rodar a suíte de novo antes do merge** — o CI do próprio PR rodou contra a staging de
antes, então verde ali não prova nada sobre a composição.

### Antes de oferecer "abro o PR?", veja se ele já existe

Um head branch pode ter PR aberto há dias — desde o primeiro marco — com o corpo descrevendo
uma fração do que a branch já contém e a base errada. Cheque **antes** de oferecer:

```bash
gh pr list --state all --limit 10 --json number,title,baseRefName,headRefName,state
gh pr view <n> --json mergeable,mergeStateStatus,baseRefName
```

Se existir, o trabalho não é "abrir PR", é **atualizar** o que está lá: retarget da base,
título e corpo reescritos para o estado atual (`--body-file` com caminho nativo). Abrir um segundo
PR para o mesmo par de branches é recusado pelo GitHub e, se você contornar, duplica a decisão
de merge.

**Mudanças não commitadas na branch errada**

Acontece de você terminar trabalho em `master` ou `staging` (porque estava validando ali, ou
porque o checkout de uma fase anterior deixou a árvore naquela branch). Não commite ali nem
jogue o trabalho fora — transporte para a branch nova:

```bash
git stash push -m "<slug>"
git checkout -b <tipo>/<slug> origin/staging
git stash pop
git branch --show-current          # confirme antes de continuar
```

Verifique `git branch --show-current` **antes** de rodar a suíte e antes de commitar: rodar
meia hora de testes na branch errada e só perceber no push custa uma rodada inteira. Se você
descobrir depois de commitar, não dê amend em branch já pushada (ver acima) — o commit vale
na branch onde está e a promoção resolve.

**`pnpm install`/`pnpm add` escreve no `pnpm-workspace.yaml` — e o `git add -A` leva junto.**
Adicionar uma dependência de workspace gera um bloco `allowBuilds:` com o texto literal
`set this to true or false` (placeholder que o pnpm não resolve sozinho), e ele aparece como
modificação não commitada ao lado do trabalho real. Não é seu código e não pertence ao PR:
rode `git checkout -- pnpm-workspace.yaml` antes do `git add`, e **confira `git status` depois
de cada `pnpm install`**, não só depois de editar fonte. Se o commit já saiu com o bloco, o
conserto é commit novo removendo — não `--amend` em branch publicada.

**`gh pr merge --delete-branch` falha quando você está na própria branch do PR.** O squash
entra, mas a deleção local não pode acontecer com a branch em uso, e o comando sai com erro
depois de já ter mergeado. Faça `git checkout master` antes, ou aceite o merge e apague a
branch local num passo separado.

### Fluxo obrigatório

1. Branch `feature/<slug>` ou `fix/<slug>` a partir de `staging` — **nunca** commit
   direto em `master`.
2. Commitar com o e-mail do dono + trailer `Agent:`.
3. PR da `feature/` para **`staging`** — não para `master`. O corpo do PR descreve a
   mudança e o que foi verificado — **sem footer de atribuição de ferramenta**
   ("Generated with ..."): a autoria do git já é a do dono, e um footer desses credita
   o trabalho à ferramenta errada no histórico do repo.
4. **Squash merge** — o autor do commit final passa a ser quem fez o merge, reforçando o gate.
   `gh pr review --approve` **falha** com `Can not approve your own pull request` sempre
   que o PR foi aberto pela mesma conta do `gh` — que é o caso normal aqui, já que o autor
   do git é o dono. Não é bloqueio: revise por `gh pr view --json mergeable` +
   `gh pr checks <n>` e faça o merge sem a aprovação formal.

   **Quando o usuário perguntar "tem outra forma de passar esse fluxo?":** medido numa conta
   solo, `--approve` e `--request-changes` falham os dois (`Can not ... your own pull request`);
   `--comment` funciona e é o único registro de revisão disponível. E **branch protection /
   ruleset não existem em repositório privado no plano Free** —
   `gh api repos/<org>/<repo>/branches/<b>/protection` responde
   `403 Upgrade to GitHub Pro or make this repository public`. Confira o plano
   (`gh api orgs/<org> --jq .plan.name`) antes de prometer proteção de branch, e ofereça o
   upgrade com preço em vez de depurar escopo de token. Resposta honesta: a trava contra push
   acidental na branch protegida é o upgrade; a aprovação por segundo revisor não tem como
   existir numa conta solo (só conta bot ou segundo membro). Teste num PR descartável antes de
   afirmar — abra, exercite cada verbo, feche e apague a branch.
5. Apagar a branch local e remota.

   **PR com a base errada: feche e use o PR certo, não brigue com o retarget.**
   `gh pr edit <n> --base staging` falha com
   `A pull request already exists for base branch 'staging' and head branch '<branch>'`
   quando já existe outro PR aberto para o mesmo par. Confira com
   `gh pr list --state open --json number,baseRefName,headRefName` antes de tentar o edit:
   se o PR da base correta já existe, atualize o corpo dele (`gh pr edit <n> --body-file`),
   mergeie esse e feche o da base errada apontando para ele. Retargetar o errado duplica o
   PR e deixa dois merges possíveis para o mesmo conteúdo.
   `--body-file` precisa de **caminho nativo** (`C:/...`); `/tmp/x.md` não é traduzido para
   binários nativos no Windows e falha com "The system cannot find the path specified".
5a. **Antes de apagar uma branch divergente, decida por CONTEÚDO, não por contagem.**
   `git cherry -v origin/master origin/<branch>` marca `+` em quase tudo quando o
   histórico divergiu, mesmo que o conteúdo já esteja mergeado — `+` ali significa
   "patch não idêntico", não "trabalho perdido". Confirme nas duas pontas:
   ```
   gh pr list --state all --head <branch> --json number,state
   git grep -c -E '<simbolo-ou-feature>' origin/master  -- <caminho>
   git grep -c -E '<simbolo-ou-feature>' origin/<branch> -- <caminho>
   ```
   PR `MERGED` cobre o conteúdo; símbolo presente no `master` e ausente na branch é
   prova suficiente. Sem essas duas checagens, apagar parece perda de trabalho e
   preservar branch morta acumula divergência — os dois erros custam caro.
6. Quando `staging` passar nos testes em homologação, PR `staging → master` com squash
   merge — é esse merge que promove para produção.

O squash merge resolve o commit final de `master`, mas **não** limpa os commits da branch — por
isso o trailer no commit é obrigatório desde o primeiro push.

### Se o usuário pedir "subir em staging"

Essa solicitação inclui **todo o pipeline**, não só commit local:

- merge da `feature/` em `staging` e push de `staging`;
- merge de `staging` em `master` e push de `master`;
- atualização do Trello com o progresso;
- verificação do deploy em staging.

Deixar as mudanças só na `feature/` enquanto informa "está commitado" é entregar
incompleto. Se `master` rejeitar fast-forward, faça merge não-ff local e push; nunca
pare em "staging está pronta" sem efetivar `master` quando o fluxo exigir.

### Inferir a função quando houver dúvida

Na ordem: 1) a tarefa em andamento no Trello; 2) o escopo da mensagem e os arquivos
tocados (`src/main/java` → `backend`, `frontend/` → `frontend`, `prisma/`,
`migrations/`, schema → `tech-lead`, `render.yaml`/`.github/workflows/` → `devops`,
`*Test.java` → `qa`); 3) quem pediu o trabalho. Se continuar ambíguo, **pergunte** — não chute.

Trabalhos que tocam mais de uma área vão para quem coordena (`tech-lead` ou `gerente`),
não para o agente da área mais visível.

### Exceção: normalizar histórico

Só para repositórios já existentes com commits de agente no histórico — o §2 explica. Todo commit
novo já sai no padrão acima, então não há o que normalizar.

## 2. Histórico anterior

**Só para repositórios com commits de agente já no histórico.** Todo commit novo já sai no padrão
do §1, então na maioria dos casos não há o que fazer.

Quando precisar, use **um único `--commit-callback`** (não `--mailmap` separado): ele troca
author e committer e insere o trailer `Agent:` no mesmo passo, porque o `--message-callback` não
tem acesso ao autor do commit.

Pegadinhas do `git filter-repo` no Windows — todas custaram tempo nesta execução:

- O arquivo do callback é o **corpo** da função, sem `def`, e `message` / `author_name` /
  `author_email` chegam como **bytes**. `return` é obrigatório.
- Precisa rodar **de dentro** do repositório (não existe `-C`).
- Apague `.git/filter-repo/already_ran` antes de rodar, ou responda `n` ao prompt — senão ele
  trava esperando stdin.
- O filter-repo faz **autostash** da working tree e o repack descarta o objeto depois. Stash
 manual antes e **recupere a mudança depois** (`git fsck --unreachable` não acha: o repack já
  eliminou). Alternativa mais segura: deixe a working tree limpa durante o rewrite.
- Sempre valide num **clone de teste** antes de tocar no repo real.

Mensagens de merge (`Merge pull request #N`) são preservadas como estão; já o `merge(...)` de
outra casa vira `chore(merge):`.

Cuidado: reescrever histórico força push e **invalida branches e PRs abertos**. Num repo com
Hobby na Vercel, o commit antigo órfão passa a bloquear deploy — limpe os PRs antes.

## 3. Conventional commits — sempre

```
<tipo>(<escopo>): <descrição em minúsculas, imperativo, sem ponto final>
```

Tipos: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`,
`chore`, `revert`.

Escopos usuais: `backend`, `frontend`, `auth`, `tests`, `infra`, `docs`.
Corpo opcional (linha em branco depois do subject); use-o para o "porquê", não para
descrever o diff. Trailer `Agent:` no final do corpo (ver §1). Nunca coloque trailer
`Author:` — a autoria vem do git, e um e-mail de agente ali bloqueia o deploy.

```powershell
git commit -m "feat(backend): anticonflito por corretor em vez de corretor+imovel"
```

**Corpo com crase vai em arquivo, nunca em `-m`.** Numa linha de `git commit -m "..."` com
aspas duplas, o bash expande `` `texto` `` como substituição de comando: o trecho evapora do
commit e o `tsc`/comando citado chega a executar. O commit sai "com sucesso" com o corpo
mutilado — você só percebe lendo `git log -1 --format=%B` depois. Para qualquer corpo que
mencione `tsc -b`, `pnpm`, nomes de campo entre crases ou código inline, escreva um arquivo
e passe caminho **nativo** (`C:/...`):

```bash
git commit -F 'C:/Users/<user>/AppData/Local/hermes/cache/scratch/commit-msg.txt'
```

Vale o mesmo para strings com `$`, `!` e aspas invertidas. Confira o corpo renderizado
(`git log -1 --format='%B'`) **antes** do push; se a branch ainda não foi publicada, o conserto
é `--amend -F <arquivo>` (ver §1).

## 4. Trello — atualizar sempre

Acompanhe o card **sempre** que uma tarefa for concluída, mudar de estado ou gerar
subtarefas. Checklist novo vai com os itens já preenchidos conforme o andamento.

Ordem canônica das colunas de fluxo (todos os boards):

`🎯 Backlog` → `🏃🏻‍♂️‍➡️ Em desenvolvimento` → `🧪🐞 Testes` → `✅ Pronto`

Listas temáticas/fases existentes (MVP, Fluxos, Sprint N, Backlog Roadmap) continuam
valendo como **fila de origem**: os cards migram para as colunas acima conforme são
trabalhados e finalizados.

### Roteiro

0. **Antes de escrever código, leia a DESCRIÇÃO COMPLETA do card.** O escopo real está
   nela, não no título, e muitas vezes ela lista pré-requisitos que mudam a ordem do
   trabalho (`<dependência> antes de qualquer rota de escrita`). Implementar pelo título e
   descobrir os requisitos depois custa um PR inteiro: a rota pode ficar correta e ainda
   assim não ser a entrega, porque o card pedia auditoria/log/piso junto.
   Do mesmo jeito, procure cards **irmãos** que já tocam a área (`trelloReadCard`
   `list_by_board` e filtrar por palavra-chave): o card novo pode ser a continuação de um
   que já está em `✅ Pronto`, e aí o comentário de complemento vai lá, não num card novo.
1. Ao iniciar: card em `🏃🏻‍♂️‍➡️ Em desenvolvimento`; crie checklist com as subtarefas.
2. Ao concluir item: marque o item do checklist (`update_item` com `checked: true`).
   **Marque item por item a partir do `tasks.md` relido, não de memória.** Antes de fechar o
   lote, releia o `tasks.md` do change e compare com o estado do card: marcar por impressão do
   que "já foi feito" coloca `[x]` num item `[ ]` e reporta progresso que não existe. Se errar,
   reverta com `update_item` + `checked: false` no mesmo turno — não deixe para o próximo.
3. Ao concluir a tarefa: comente o log no card, mova para `🧪🐞 Testes` (se houver
   testes) ou direto para `✅ Pronto`.
4. Comentário de log no card ao concluir, sempre — formato:

```
✅ <tarefa concluída>
- <entrega 1>
- <entrega 2>
Commits: <sha> <sha>
```

**Item do card que não tem dado de suporte não se implementa — recusa com motivo.** Antes de
prometer um item da descrição (custo/uso, mapa de dependências, SLA, capacidade), procure a fonte:
`ls packages/` por um adapter de billing, `grep -n 'dependsOn\|dependency' packages/config/src/schema.ts`
para dependência declarada. Se não existe, implementar exige inventar número — o entregável vira
um documento dizendo o que falta (qual API do provider, qual escopo de credencial, qual campo
adicionar ao registry) e por que inferência por nome/rede não serve. Recusar nomeado é entrega;
preencher com dado plausível é o pior resultado possível, porque parece real.

**O card é atualizado junto com o commit — não é etapa seguinte.** Quando o pedido do usuário é
"commit e deploy", um commit que sai sem o log no card é tarefa incompleta, e o card fica parado na
coluna errada reportando progresso que não existe. Comente o log **antes de informar o sha no
relatório ao usuário**: quem lê o card é quem vê o histórico, e o sha tem que estar disponível no
momento em que o log é escrito. Se a tarefa não tem card, crie um em `✅ Pronto` em vez de pular a
etapa — **mas só se o sistema já tem board**. Quando o board do sistema não existe
(`trelloReadBoard action=list` e nenhum bate com o projeto), não crie um por conta própria:
criar estrutura de board é decisão de processo, igual a criar `staging`. Entregue o trabalho,
diga no relatório que não há board para aquele sistema e pergunte o destino antes de inventar
um.

### Uso do MCP (mcp__trello__*)

**O MCP funciona normalmente.** O erro `missing required argument(s): action` significa que você
esqueceu de passar `action=` — não é bug da ferramenta. Passe `action=` explicitamente em toda
chamada (list, get, list_by_board, create, move, mark_done, add_comment, etc.) e a chamada
sucede. Se a ferramenta responder com `missing required argument(s): action`, reenvie com o
`action=` correto em vez de desistir ou tentar curl.

**Nunca chute o `action`.** O enum é **por ferramenta** e não bate com o nome dela. Chame
`tool_describe` antes do primeiro `tool_call` da sessão e monte os argumentos pelo schema
que ele devolver.

- `trelloReadList` — `list_by_board` | `get` | `list_activity`. **Não existe `list`.**
  O parâmetro da lista é `listId` (**não** `id`); `list_by_board` exige `boardId`.
- `trelloWriteChecklist` — `add_item` **rejeita `checked`** ("does not accept: checked").
  Para criar item já marcado: `add_item` (só `checklistId` + `text`) e depois
  `update_item` com `checked: true` no `itemId` devolvido. Não insista no `add_item`
  com `checked` — o MCP pausa após 3 rejeições e o turno vira loop.
- O MCP do Trello entra em **pausa automática após 3 erros de argumento** na mesma
  sessão. Corrija os argumentos e pare de chamar — repetir a mesma chamada só mantém
  a pausa. Se `tool_describe` mostrou `additionalProperties: false`, remova chaves
  extras antes de reenviar.
- `add_item` aceita `pos` e **sem ele o item entra no fim** da checklist. Quando a
  ordem importa (fases de um plano, passos de um fluxo), passe `pos` explícito em
  progressão (`100`, `200`, `300`, …) em vez de confiar na ordem de criação —
  checklists de várias fases criadas em sequência terminam fora de ordem.
  Use `pos: "top"` só quando a ordem real não importa.
- `trelloReadCard` `get` **não devolve checklists** (`checklists: []` mesmo quando existem).
  Confirme o estado pelo retorno do `update_item` (`state: COMPLETE`) ou use `trelloReadChecklist`.
- `trelloWriteCard` / `trelloWriteChecklist` são **local tools**: o `tool_call` aceita
  **uma única entrada** por chamada. Só nomes `connectors__*` podem ser agrupados.
- `trelloReadCard` — `get` | `list_by_board` | `list_by_list` | `list_comments` |
  `list_activity` | `list_attachments` | `get_comment` | `list_custom_field_items`.
  O parâmetro é `cardIdOrUrl` (para `get`), `boardIdOrUrl` (para `list_by_board`) e
  **`listId`** (para `list_by_list`) — não invente um nome único; cada `action` tem o seu.
- Todas têm `additionalProperties: false`: uma chave a mais derruba a chamada com
  `Additional properties are not allowed`.

`trelloWriteBoard` só expõe `action: "create"` — renomear board, mudar descrição ou
visibilidade pelo MCP é impossível (o schema lista `update` no texto mas o enum rejeita).
No `create` os parâmetros são **`workspaceId`** (ARI do workspace) e **`desc`** — `organizationId`
e `description` são rejeitados com `Additional properties are not allowed`. `visibility` aceita
`ORG` (workspace) ou `PRIVATE`.
Também não existe `trelloWriteCard` para clonar listas em massa. Para isso use a REST API
v1 direto, reaproveitando o token OAuth do MCP já salvo:

```python
TOK = json.load(open(r'C:\Users\<user>\AppData\Local\hermes\mcp-tokens\trello.json'))['access_token']
# PUT https://api.trello.com/1/boards/<id>  com name=, desc=
# headers: Authorization: Bearer <TOK>   (NÃO use ?key=&token= — o token expirado dá "invalid key")
```

Encode o corpo com `urllib.parse.urlencode(...).encode()` + `Content-Type:
application/x-www-form-urlencoded; charset=utf-8`; acentos precisam disso, senão voltam como
`?` e `??`. Para arquivar card/lista: `PUT /cards/<id>` ou `PUT /lists/<id>` com `closed=true`.

**Endpoints REST que o MCP não expõe com a mesma forma.** Marcar item de checklist e comentar
são as duas operações que aparecem em lote ao montar um board. As rotas corretas:

| Operação | Rota |
|---|---|
| marcar item | `PUT /cards/<cardId>/checkItem/<itemId>` com `state=complete` |
| comentar | `POST /cards/<cardId>/actions/comments` com `text=` |

`PUT /checklists/<checklistId>/checkItems/<itemId>` — a forma que espelha a leitura — responde
**404**. Use o caminho do card.

### Criar um board do zero (projeto novo)

`trelloWriteBoard` cria o board e **nada mais** — ele nasce sem nenhuma lista. O workspace sai de
`GET /members/me/boards?fields=name,id,idOrganization,shortLink`, pegando o `idOrganization` de
um board existente do mesmo workspace.

```python
post(f'boards/{BOARD}/lists', name=name, pos=pos)   # 16384, 32768, 49152, 65536
```

As quatro colunas de fluxo para o board novo são criadas com `pos` numa progressão explícita,
nunca deixando o Trello decidir: lista e item sem `pos` entram no fim, e aí a ordem de criação
vira o layout — que quebra no dia em que uma fase precisa ser inserida no meio.

**Antes de criar um board, procure o que já existe.** Se o projeto já tem board com as quatro colunas de fluxo, ele é o board — criar um segundo para "organizar melhor" deixa dois boards do mesmo produto e nenhum dos dois é confiável. Use `trelloReadBoard action=list` e `trelloReadCard list_by_board` para ver o que já está lá; se as colunas batem, atualize os cards existentes em vez de criar estrutura paralela.

**Arquivar card não é grátis: comente o que é insubstituível em outro lugar primeiro.** Um card em Backlog pode carregar anos de decisão em comentários (avaliação de fornecedor, medição de latência, preço comparado) que não existe em nenhum outro artefato. Antes de `closed=true` em lote, leia os comentários e copie cada um que contenha uma **decisão com evidência** para o card que o substitui, citando o card arquivado pela URL. Reorganizar o board não pode apagar o raciocínio que o board guardava.

**Um board novo espelha a realidade, não só o futuro.** Ao criar o board de um projeto que já
tem trabalho feito, a maior parte dos cards é histórica: o concluído vai para `✅ Pronto` com
checklist **todo marcado** e um comentário de log com o artefato que prova (saída do teste,
resultado do validador); a fase em andamento vai para `🏃 Em desenvolvimento` com a proporção
real (`7/8`), nem cheia nem vazia; e só o que falta vai para `🎯 Backlog`. Board que começa vazio,
ou que joga o trabalho feito no Backlog, mente sobre o estado do projeto no primeiro minuto.

**Board criado em script: `post` com `urlencode`, nunca `tool_call` em lote.** `trelloWriteBoard`
cria o board e **nada mais** — listas, labels, cards e checklists saem da REST v1 com
`Authorization: Bearer <token>`. Três armadilhas custam a rodada inteira:

- **`post()` com `**params` não aceita lista em valor.** `urlencode({'idLabels': [a, b]})` serializa a
  representação do Python e a API responde **400 Bad Request** sem dizer qual campo. Para labels,
  mande `idLabels` como **string separada por vírgula** (`','.join([LAB['MVP'], LAB['Backend']])`),
  ou faça `POST /cards/<id>/idLabels` com `value=<id>` por label. Ao depurar um 400 desse helper,
  o primeiro suspeito é um valor de lista, não o token.
- **Labels criados pela REST nascem fora de ordem.** Passe `pos` explícito (`(i+1)*16384`) em
  `PUT /labels/<id>` depois de criar, senão o filtro do board fica com a ordem de criação.
- **`idLabels` espera IDs, não nomes.** Um helper com parâmetro `labels` que sombreia o dict
global de labels estoura `TypeError: list indices must be integers` no meio do lote. Nomeie o
  parâmetro `label_names` e resolva `LAB[name]`.
- **`remove` de memória apaga a ENTRADA inteira**, não o trecho de `old_text` — para encurtar
  uma entrada use `patch`. Perder o registro inteiro de um projeto por causa de 40 caracteres
  é pior que não editar.
- **`workspaceId` é o único caminho para o board cair no workspace certo** (ver acima), e o
  mesmo workspace se descobre lendo `idOrganization` de um board existente — não invente.

**Clonar a estrutura de um board existente é a forma certa de abrir um board de projeto novo.**
Leia o board de referência antes de criar nada: `GET /boards/<id>/lists?fields=name,pos` e
`GET /boards/<id>/labels?fields=name,color`. Isso devolve os emojis exatos das colunas de fluxo
e os 9 labels do padrão (`Backend`, `Frontend`, `Qualidade e operação`, `Segurança`, `Fluxos`,
`DevOps`, `MVP`, `Docs`, `QA`) — copiar a estrutura garante que o filtro por label e a leitura
por coluna funcionem igual nos dois boards. Listas novas vão com `pos` explícito (16384, 32768,
49152, 65536), nunca na ordem de criação.

Confira o resultado com uma leitura por lista contando `checkItems` completos/totais por card:
um board "pronto" no relato e vazio na API é o erro que só aparece aqui.

**O card que depende de decisão do dono fica aberto onde está.** Não force um card para
`✅ Pronto` para o board ficar arrumado: a fase fica em `Em desenvolvimento` com o item em aberto
nomeado e um comentário dizendo que é decisão do dono. É a regra de "entrega parcial não move o
card" aplicada no dia zero.

**Monte o board em `execute_code`, não em `tool_call`.** `trelloWriteCard`/`trelloWriteChecklist`
são local tools que aceitam **uma entrada por chamada**; 17 cards com ~100 itens de checklist
viram mais de 100 turnos. Um script com `urllib.request` + `Authorization: Bearer <token>` faz
tudo em 2–3 chamadas. Grave os ids devolvidos num JSON no workspace entre um chamado e outro.

**Redefina `get`/`put`/`post` no topo de CADA célula `execute_code`.** Elas não sobrevivem de
uma célula para a seguinte na prática (a célula seguinte estoura `NameError: name 'post' is not
defined` no meio do lote, depois de já ter gravado parte das mudanças). Repetir as quatro linhas
de helper custa nada; descobrir no meio de um lote de checklist custa o lote.

**O JSON de ids em scratch é cache, não fonte de verdade — ele é apagado.** Arquivos em
`cache/scratch` somem entre rodadas (expiram por idade), e o sintoma é um
`FileNotFoundError` no meio de um script de board que funcionava. Não tente recuperar o arquivo
nem redescobrir a estrutura card por card: **releia board, listas e cards da API numa chamada**
e siga. Depois de qualquer erro assim, regrave o JSON a partir da leitura nova — e, quando o
board for permanente, registre os ARIs nas tabelas deste §4 além do arquivo temporário, para
que a próxima sessão não dependa de cache nenhum.

**Escrevendo esse script de memória, você vai errar as duas rotas que mais usa.** Antes da
primeira linha, releia a tabela *Endpoints REST que o MCP não expõe com a mesma forma* logo
acima: marcar item é `PUT /cards/<cardId>/checkItem/<itemId>` e a forma simétrica
`PUT /checklists/<id>/checkItems/<id>` responde **404** — o erro aparece no meio de um lote de
30 itens e derruba o script com o checklist pela metade.

**Falha no meio do lote deixa checklist pela metade.** Se um `add_item` estourar, os anteriores
já estão gravados e o card fica com checklist parcial — pior que sem checklist, porque `[1/6]`
parece progresso. Depois de qualquer exceção: `GET /cards/<id>?checklists=all`, conte o que
existe e **acrescente só o restante** com `pos` continuando a progressão. Não recrie o card.

**Identificador é ARI, e nem toda ferramenta aceita URL.** `trelloReadList`
`list_by_board` rejeita `https://trello.com/b/<shortLink>` com `Invalid ARI` — precisa do
ARI completo do board, `ari:cloud:trello::board/workspace/<workspaceId>/<boardId>`, com o id
**interno** do board, não o shortLink. `trelloReadCard` aceita a URL do board e do card.
Para montar o ARI do board sem caçar o id: leia um card qualquer e pegue `board.id`
(ou `boardId`) da resposta.

O mesmo vale para `trelloSearch` `search_cards` com `boardIds`: passar o shortLink devolve
HTTP 500 com `Cast to ObjectId failed for value "<shortLink>" … at path "_id" for model
"Board"` — erro de tipo do GraphQL do Trello, não busca sem resultado. Passe o ARI do board
(pelo `boardId` de um card) ou pule a busca e vá direto em `trelloReadCard list_by_board`,
que já agrupa por lista.

**Resposta grande estoura o contexto.** `list_activity` devolve cada card com as 25
últimas atividades, e `trelloReadCard list_by_board` devolve o board inteiro — os dois
passam de 100 KB e o resultado cai num arquivo com só um preview inline. Processe o
arquivo salvo com `execute_code`/`json`; **não** repita a chamada na API.

**Estar na coluna não é estar pronto.** Um card em `✅ Pronto` pode vir com
`complete: false`. Confirme o estado pelo `activity`/`comments` do card, não pela lista.

**Entrega parcial não move o card.** Se o que foi implementado atende parte do que o card
pede, o card **fica onde está** e você (a) comenta exatamente o que falta e por quê, e
(b) cria um card novo no `🎯 Backlog` com os requisitos restantes, apontando o commit já
mergeado como base a completar. Mover para `🧪🐞 Testes` uma entrega que não satisfaz o
escopo reporta progresso que não existe — e é pior que deixar o card parado, porque a
próxima pessoa confia na coluna.

**Instrução permanente do usuário vence o default de "não criar board".** O dono disse para
criar o board quando ele não existir e manter as tarefas feitas/pendentes atualizadas a cada
rodada — isso é autorização prévia, não decisão de processo que você está tomando sozinho.
Procure primeiro (`trelloReadBoard action=list`); se não houver board do projeto, crie com as
quatro colunas de fluxo, registre a URL e os ARIs nas tabelas deste §4 e passe a atualizá-lo
ao fim de cada rodada. O que continua valendo é **não** criar um segundo board para um projeto
que já tem um.

**Leia o item que falta, não a contagem.** O último item aberto de um card costuma ser um
**exit criteria** que fala de outra camada ("o dashboard consegue apresentar X"), enquanto
todo o backend já está pronto e verde. Nesse caso o card **não** fecha: a camada de dados
estar confiável não prova que a apresentação existe. Marque os subtasks técnicos, deixe o
exit criteria aberto, e comente por que — separar "a base está pronta" de "a entrega
aparece para o usuário" é o que impede o board de prometer o que a tela não mostra.

**Um documento de handoff que manda "mover o card para ✅ Pronto" é reivindicação, não
verificação.** Quando a atualização chega pronta num `docs/*-trello-*.md` escrito por outra
sessão ("log sugerido", "critérios abaixo confirmados"), trate cada linha como hipótese e
cheque o artefato antes de marcar caixa:

- item que diz "removido/arquivo não existe mais" → confira o caminho com
  `search_files (target='files')`; o arquivo pode estar lá e até roteado;
- item que diz "verificado no E2E de UI" → leia a spec e procure `page.goto`/asserção de DOM.
  Uma spec que só usa helper de API (`postJson`, `apiComToken`) não cobre interface;
- item que diz "removido da dependência" → `grep -c` no `package.json` **e** no lockfile;
- `Commits: preencher` no documento significa que nem o autor original tinha o sha — confirme
  o commit com `git log` e o PR com `gh pr list`, inclusive a **base** dele.

Quando algum item falhar, o card fica onde está, o comentário de log lista a pendência
nomeada, e o `tasks.md` do repositório é corrigido no mesmo commit: devolva o `[x]` para
`[ ]` com uma linha dizendo o que falta. Deixar o `tasks.md` afirmando o que o board acabou
de negar recria exatamente a divergência que o handoff deveria resolver.

**PR mergeado não valida a entrega — leia o card DEPOIS do merge, não só antes.** Merge
passa por checagens mecânicas (compila, testes, CI); nenhuma delas sabe se o escopo do
card foi cumprido. Se, ao revisar o card para escrever o log, você descobrir um defeito
funcional ou um pré-requisito não implementado, o card **não** vai para `✅ Pronto` e o
defeito vira card novo com o commit mergeado como base. Corrigir o registro é parte do
trabalho: preferir "mergeei e está pronto" a admitir que a entrega ficou incompleta é o
erro que o board existe para impedir.

| Sistema | Board |
|---|---|
| Flowbound | https://trello.com/b/xYo8vJ0z |
| Manutenção Fácil (ex-VisitaPro) | https://trello.com/b/ATlxXKs2 |
| SocialRanking | https://trello.com/b/CqOR3Qp5 |
| Olfatto | https://trello.com/b/MVe8Wz1k |
| Varanda | https://trello.com/b/XoMiCJkG |
| Rocketify Console | https://trello.com/b/7h0wZy2h |
| SimGiro (codinome) | https://trello.com/b/nSHBL51J |
| Orquídeas e Lazer | https://trello.com/b/TXqKaYUD |
| Sorella (codinome) | https://trello.com/b/pgo7kRK5 |

- **Sorella (codinome)** — board `6aef35ec534de6c9766632d4` — Backlog
  `6aef35fb5637c1b411a6ee24` · Em desenvolvimento `6aef35fc79c8da221e07db72` · Testes
  `6aef35fc22fbee7b55707f75` · Pronto `6aef35fcb767ace9b8ed82a9`

- **Rocketify Console** — board `6aee7fe08e464b57a185cfc9` — as quatro colunas de fluxo são
  `🎯 Backlog` · `🏃🏻‍♂️‍➡️ Em desenvolvimento` · `🧪🐞 Testes` · `✅ Pronto`

Listas de fluxo (ARIs):

- **Flowbound** — prefixo `ari:cloud:trello::list/workspace/6abc3932e4b6b6ead374ebe5/` — Backlog
  `6ae8e0d457b561448ac65c9d` · Em desenvolvimento `6ae8e0d4bdef450be7f85dd7` · Testes
  `6ae8e0d52d58739258e75361` · Pronto `6ae8e0d535a20c157ffd41d6`
- **Manutenção Fácil (ex-VisitaPro)** — Backlog `.../6ac94362daabf22a0c6243b3` · Em desenvolvimento
  `.../6ac9436123083362738eacc4` · Testes `.../6ac943612ddedd7ef091d04d` · Pronto
  `.../6ac94360dc3fc8909aca1f1f`
- **SocialRanking** — Backlog `.../6abc46aa921332ca75a18d89` · Em desenvolvimento
  `.../6ac5719aaf91e03c8331917c` · Testes `.../6ac5719ff4400250d85ea3b5` · Pronto
  `.../6ac571a2af1877a0d2786989`
- **Olfatto** — Backlog `.../6ac943642854a3954c475786` · Em desenvolvimento
  `.../6ac94364abf1ec720c93f05c` · Testes `.../6ac9436359767c34aba48a5c` · Pronto
  `.../6ac94362a1e241e0be0b2863`
- **Varanda** (board `6ae95604a50956cbb8e05598`) — Backlog
  `.../6ae956102d3c065b74b39487` · Em desenvolvimento `.../6ae95611a13803f216d0962f` · Testes
  `.../6ae95611d8b5d6e3a8ed83ce` · Pronto `.../6ae95612bc44c25dfc6462ef`
- **SimGiro (codinome)** — board `6aecb9e1294b871ef9e04e98` — Backlog
  `.../6aecb9e9e9aaa01bd6104d18` · Em desenvolvimento `.../6aecb9eaa69fbf37a2d28ee4` · Testes
  `.../6aecb9eaf785dc8c83faf7cf` · Pronto `.../6aecb9eb90de89d3bf109c9a`
- **Orquídeas e Lazer** — board `6aecc62b03ecf3f7e85c8180` — Backlog
  `.../6aecc638497264c36aebc784` · Em desenvolvimento `.../6aecc6386e5643b0ea6baac0` · Testes
  `.../6aecc638957cf371cd7a49ce` · Pronto `.../6aecc6387072b3b1ee9058d0`

Prefixo `ari:cloud:trello::list/workspace/6abc3932e4b6b6ead374ebe5/`.

## 5. Fluxo de desenvolvimento backend (Java Quarkus)

### Ao criar uma entidade nova (ex: Avaliacao, Notificacao)
1. Poluíra o método `of(JsonObject json)` com `import com.fasterxml.jackson.databind.JsonObject;` — sem isso, `mvn compile` falha no `cannot find symbol class JsonObject`.
2. Se a coluna do banco mudou de nome (ex: `nota` → `estrelas`), atualize **todos** os lugares:
   - campo `public Integer nota;` → `public Integer estrelas;`
   - getter/uso `a.nota` → `a.estrelas`
   - construtor `new Avaliacao(ordem, nota, ...)` → `new Avaliacao(ordem, estrelas, ...)`
3. Se o corpo da entidade é um DTO (Resumo, Filter), não crie construtores que referencia propriedades ausentes — o compilador não dá fallback; `this.id = a.id` falha se `id` for `Long` e `a.id` for `null` no construtor, a menos que use `a.id != null ? a.id : null`.
4. Evite métodos duplicados no mesmo `Entity`: `findByOrdemServico` definido duas vezes causa `method ... is already defined`.

### Verificação antes de commitar backend
- `mvn compile` → `BUILD SUCCESS` (sem erro)
- `mvn test` → 89 pass, 0 fail (após `docker compose down` para liberar porta 8081)
- **Falha em teste que depende de DATA não é regressão sua.** Antes de investigar, leia a
  mensagem: se o próprio teste declara o pressuposto (ex: "a suíte assume que hoje não é
  sábado"), ele falha por calendário e não por código. Isole o resto com
  `-Dtest='!<ClasseDoTesteDeData>'` e confirme `176 pass, 0 fail`; reporte as de data como
  pré-existentes, com o pressuposto citado, em vez de "corrigi o build".
- `npm run build` + `npm run lint` (frontend) → exit 0
- **A branch de destino compila?** Se ela divergiu da sua, rode o compile num worktree do
  `origin/<base>` (ver §1 item 6b). Um método que existe na sua árvore e não na base
  — ou o contrário — passa no build local e falha no merge.

### Antes de criar uma rota de escrita sobre um flag, ache a FONTE DE VERDADE

Um campo derivado (permissão, status, contador) costuma ter **duas** representações: a tabela
que manda e uma cópia desnormalizada que o app lê. Escrever na cópia parece funcionar — a rota
devolve 200 — e some no próximo update se um trigger re-deriva o campo.

```
grep -rn -i '<flag>' <schema.sql>            # tabela de origem + funcao de trigger
# ex.: create trigger t_promove_admin before insert or update on auth.users
```

Se existe `before ... on <tabela>` que reescreve o campo a partir de outra tabela, a rota tem que
escrever **na tabela de origem** e deixar o trigger propagar. Escrever direto no campo derivado
funciona num sentido (ligar) e é revertido no outro (desligar) — o pior tipo de bug, porque o
caminho testado à mão é o que funciona.

Mesma regra para qualquer ação de escrita em console administrativo: **log de auditoria (quem, o
quê, quando), piso mínimo de segurança (não trancar o produto rebaixando o último admin) e
idempotência explícita** vêm ANTES da rota, não depois. Escrita por API sem rastro é pior que a
operação manual que ela substitui, porque parece legítima.

### Conferir o que a base realmente tem antes de "consertar" um diff
Antes de tratar uma diferença como bug do código ou como mudança intencional, compare com as
duas pontas:
```
git diff origin/staging -- <arquivo>
git diff origin/master  -- <arquivo>
git show origin/master:<arquivo> | grep -n '<simbolo>'
```
Diff idêntico contra `master` e contra `staging` significa que a mudança é apenas
**local/não commitada** — não é divergência de branch e não pede merge, pede commit.
Confirmar isso evita "resolver conflito" que não existe.

## 6. Frontend / Clerk

### Roteamento do Clerk

`ClerkProvider` usa **path routing** por padrão. Quando a app monta `SignIn`/`SignUp` dentro da própria SPA, **não** passe `routing="hash"` nesses componentes.

- Misturar `routing="hash"` nos componentes com path routing no provider produz URL mista tipo `/sign-in#/factor-one` e redirecionamentos instáveis após erro de senha.
- Se a app precisa de hash routing, configure no provider; se usa path routing padrão, remova `routing="hash"` de todos os `SignIn`/`SignUp`.

### Estado pós-login

`afterSignInUrl` está **deprecado** no Clerk v5; use `signInFallbackRedirectUrl`/`signInForceRedirectUrl` (e equivalentes de sign-up). `afterSignInUrl` pode deixar o redirect instável e exigir refresh manual.

Quando a app usa **hash routing** (`#/...`), forneça `routerPush` e `routerReplace` ao `ClerkProvider` para manter a navegação dentro da SPA. Sem eles, o Clerk cai em `window.location` e o estado de auth pode travar em "Carregando autenticação..." até refresh manual.

### Recuperação de sessão e token

Na tela de erro de sessão (`Não foi possível carregar a sessão`), exiba ação direta de logout via `signOut()` do Clerk para permitir recuperação sem console. Isso evita usuário preso em estado de auth quebrado quando `/api/sessao` retorna 404/401 inesperado.

`getToken` retornado por `useAuth()` pode mudar de identidade entre renders; não use-o diretamente como dependência de `useEffect` que define o getter de token. Estabilize com `useRef` + effect separado para evitar toggle de `tokenPronto` e 401 na primeira carga.

Se o token for `null` na primeira requisição (Clerk ainda inicializando), faça um retry curto (ex.: 250ms) antes de enviar sem `Authorization`.

### Ambiente de teste (Playwright/Node)

`src/env.ts` (ou equivalente) não deve assumir `import.meta.env` definido. Em Playwright/Node ele é `undefined`; use `(import.meta as { env?: ... }).env ?? {}` para não quebrar os testes E2E.

## 7. Comunicação: regra da concisão

**Todo texto que o usuário lê passa pelo `humanizer` antes de sair.** Isso inclui
mensagem de chat, comentário de commit, corpo de PR, card e comentário no Trello,
README, ADR, GDD e changelog. Vale para todos os projetos Rocketify.

Carregue a skill `humanizer` (`skill_view(name='humanizer')`) e aplique quando o
texto for longo o bastante para ter voz própria: doc, PR, card, mensagem de
entrega. Em resposta curta de chat, o hábito basta (a lista abaixo é o resumo).

### Os vícios a cortar, em ordem de frequência real

1. **Travessão (—) como conector universal.** Use ponto, vírgula, dois-pontos ou
   parêntese. O travessão é o sinal mais reconhecível de texto gerado.
2. **"Não é X, é Y"** e **"não só X, mas Y"**. Diga a coisa direto.
3. **Frase de efeito no fim do parágrafo** ("Isso muda tudo.", "É a diferença entre
   funcionar e parecer que funciona."). Corte. Se o parágrafo precisa de remate,
   ele não terminou de dizer nada.
4. **Abertura de palco** ("Vamos lá", "Aqui está o ponto", "Sinceramente?").
   Comece pelo conteúdo.
5. **Trio forçado.** "Rápido, seguro e simples." Use a quantidade que o conteúdo
   pede, não três.
6. **Palavra inflada:** crucial, robusto, abrangente, cenário (abstrato), jornada,
   explorar, mergulhar, potencializar, alavancar, panorama, testemunho.
7. **Negrito decorativo** e **cabeçalho em Title Case**. Negrito é para o que o
   leitor precisa achar rolando a página, não para enfeitar início de item.
8. **Emoji em título e bullet** de documento técnico.
9. **Significado inflado:** "marca um momento decisivo", "reflete uma tendência
   maior", "desempenha papel fundamental". Fique com o fato.
10. **Pergunta retórica respondida na linha seguinte.**

### O que separa texto com voz de texto limpo e morto

Tirar vício é metade. A outra metade:

- **Opinião.** Relate o fato e reaja a ele. "Não sei o que pensar disso" vale mais
  que listar prós e contras sem se posicionar.
- **Ritmo variado.** Frase curta. Depois uma longa que leva tempo para chegar.
- **Especificidade no lugar de adjetivo.** Não "o desempenho melhorou"; diga
  "o build caiu de 20 minutos para 4".
- **Primeira pessoa quando couber.** "Testei e falhou" é mais honesto que "foi
  verificado e apresentou falha".

### Onde isso mais importa

O usuário já reclamou de texto genérico em entrega de projeto. Documento técnico
com voz de release note de SaaS é o caso que mais incomoda: ADR que fala "esta
decisão representa um marco" em vez de dizer qual problema resolve. Se o texto
pode ser colado em qualquer projeto trocando dois substantivos, ele está errado.

### Antes de entregar

Pergunte a si mesmo: **o que faz este texto parecer escrito por IA?** Responda em
uma linha e corrija. Repita uma vez. Só então envie.

**Nunca imprima status de execução ou prefixos como "Chamando 1 e 3", "Status do Projeto", "✅ Concluído: ..." no início de mensagens.** O usuário não quer narrativa do processo; quer o resultado.

- O corpo da resposta deve conter diretamente: o que foi feito, o estado, e o próximo passo.
- Uso de marcadores de progresso ("Agora vou...", "Vou corrigir...", "Agora testando...") é permitido internamente, mas **não deve aparecer no texto final** que o usuário lê.
- Cabeçalhos de seção são permitidos apenas quando há múltiplas entregas explicadas.
- Não repita o enunciado da tarefa no início da resposta.

### Exemplo correto

**Errado:**
> Chamando 1 e 3. A branch está na branch `feat/controle-acesso-e-perfil`. O PR #1 está aberto.

**Certo:**
> PR #1 foi mesclado em `main`. Branch `feat/controle-acesso-e-perfil` segue ativa para a Fase 4.

### Quando o usuário diz "continue" / "prossiga"

Isso significa **execute o próximo passo**, não repita o que já foi feito. Não re-explique
commits, diff ou estado atual; vá direto à ação seguinte (build, push, merge, deploy,
atualização do Trello). Se o passo seguinte for bloqueado por informação faltando,
peça só o que falta, sem re-expor o histórico.

### Nunca tranquilize sobre custo, cota ou "está tudo certo" sem medir

Quando o usuário manda um print do painel e pergunta "vou ser cobrado?", a resposta exige
medição por produto/SKU, não comparação do total com o saldo. "Está dentro do crédito" é uma
afirmação sobre **saldo**; a pergunta é sobre **free tier**, que é por item e agregado por conta
de faturamento. Responder a segunda com a primeira gera cobrança surpresa e destrói confiança.
e a conta chega com o usuário citando a sua própria frase de volta.

Se a medição desmentir uma previsão sua anterior, **diga isso com todas as letras** na próxima
mensagem ("minha conta estava errada: faltava X") antes de apresentar o número novo. Não emende
o número corrigido como se sempre tivesse sido aquele: o usuário lembra do valor antigo e a
silenciosa troca parece manipulação.

### Quando o usuário diz que vai testar antes de remover algo

Se ele diz "não removi a variável ainda, quero ver se não vai quebrar nada" e depois "adicionei
X, pode continuar": **o estado do painel é dele, não seu. Confirme o efeito real, não o
combinado.** Leia o artefato publicado (bundle JS, env do serviço) para saber qual configuração
está de fato em vigor antes de concluir que a mudança surtiu efeito. Config declarada no painel
≠ deploy concluído ≠ tráfego na versão nova. São três estados e só o último conta.

## 8. Roteiro de status ("Me atualize" / "Status" / "Analise onde paramos e quais os próximos passos")

Uma passada só, comandos reais, sem narrativa. Rode os blocos independentes em paralelo. O pedido
"onde paramos e próximos passos" é a mesma tarefa com ênfase no estado do repositório e na
sonda ao vivo — o resultado é a lista de pendências com dono, não um resumo do histórico.

1. **Repo** — `git fetch origin --prune`, `git status -sb`, `git log --oneline -10`,
   `git branch -a -vv`. Compare `main` local com `origin/main`: um squash merge no GitHub
   deixa o `main` local atrás **mesmo com a árvore limpa**, e a branch já mesclada continua
   listada até alguém apagá-la.
2. **PRs e issues** — `gh pr list --state all --limit 6`, `gh issue list`.
3. **CI** — `gh run list --limit 5`. Workflow configurado para rodar em push/PR mas **sem
   nenhuma execução para os merges recentes** é achado real: reporte a ausência de
   execuções em vez de assumir "CI verde", e não afirme a causa sem checar.
- **Docker** — `docker compose ps`. Container na porta 8081 bloqueia `mvn test`.
- **Diff entre branch local e remota no status**: `git status -sb` mostrando `??
  infra/terraform/**/.terraform*` e `terraform.tfstate` é lixo de execução, não trabalho —
  não commite nem mencione como pendência do usuário. `ahead N` em `master` com árvore
  limpa normalmente é merge local que falta pushar: confirme com
  `git log --oneline origin/master..HEAD` antes de chamar de "divergência".
5. **Trello** — board e as 4 colunas de fluxo (ARIs no §4). Sinalize card parado em
   `🏃🏻‍♂️‍➡️ Em desenvolvimento` cujas tarefas já estão todas `[x]`, e card em `✅ Pronto`
   com `complete: false`.
6. **OpenSpec** — `ls openspec/changes/` e `openspec validate --all`. Se a CLI Node falhar com
   `OpenSSL configuration error` antes de validar, rode com `OPENSSL_CONF=/dev/null` (no Windows o
   `OPENSSL_CONF` global costuma apontar para um diretório, e a CLI tenta abri-lo como arquivo).
   Reporte o resultado da validação, não o erro da CLI.
7. **Deploy** — confirme o alvo real com evidência ao vivo, **nunca** pelos arquivos de config do
   repo: `curl -sI` no domínio de produção (o header `Server:` diz quem serve), probe de
   `/q/health/ready` e `/api/*`, e o MCP do provedor para listar deploys e aliases. Um `render.yaml`
   no repo pode descrever serviços que nunca foram criados **ou que foram abandonados** — mas 404 no
   host do blueprint também pode ser só o nome do serviço diferindo do arquivo. Cruze com a base da
   API embutida no bundle publicado antes de concluir qualquer coisa. Leia o **corpo** do erro: 503
   com `Service Suspended` é serviço suspenso, não queda. Detalhes em
   `references/deploy-verification.md`.
8. **Entregue ≠ promovido.** Conte a divergência nos DOIS sentidos antes de dizer "está no ar":
   `git rev-list --left-right --count origin/master...origin/staging`. Dezenas de commits só em
   `staging` e `master` intocado = entrega pronta e nunca promovida; `scripts/check-sync.mjs`
   verde prova apenas que o invariante master ⊆ staging vale, **não** que a promoção aconteceu.
   Confirme no provedor: deploy de produção mais antigo que a data do último merge em `staging` é
   promoção pendente, não build lento.
8b. **Env var do Vercel vence o rewrite do `vercel.json` — e não aparece no repo.**
   Quando o frontend chama `/api/...` relativo e o `vercel.json` faz o proxy para o backend,
   a intenção do código é `VITE_API_BASE_URL` **vazio**. Se essa var existir no painel, ela vence
   (o default do código só vale para `undefined`), o rewrite nunca é consultado e todo fetch vai
   para o host da var. Depois de migrar região/backend, conferir as env vars mesmo com o repo
   correto:
   ```bash
   vercel link --yes --scope <team> --project <proj>   # na RAIZ do repo, não na subpasta
   vercel env pull <tmp>/env.txt --environment production -y
   grep VITE_ <tmp>/env.txt
   ```
   O sintoma é `Failed to fetch` / "Não foi possível carregar o board", sem erro de CORS e sem
   nada no log do backend — porque o request não chega. Confirmar qual host o bundle publicado
   usa antes de suspeitar de backend: `curl -s <site> | grep -oE 'assets/index-[A-Za-z0-9]+\.js'`
   e grep `https://` no arquivo.

8c. **Conferir o deploy de produção pela API do Vercel, não pelo CLI.** `vercel ls` mostra
   `Ready`/`Error` e mais nada; a causa fica em `errorMessage`/`errorCode`:
   ```bash
   curl -s -H "Authorization: Bearer <token>" \\
     "https://api.vercel.com/v6/deployments?projectId=<id>&teamId=<team>&limit=5"
   curl -s -H "Authorization: Bearer <token>" "https://api.vercel.com/v13/deployments/<uid>"
   ```
   Token em `~/AppData/Roaming/com.vercel.cli/Data/auth.json`. `source: git` com `READY` e
   data antiga = o webhook não está disparando; confira `gh api repos/<org>/<repo>/hooks`.
   **Repo renomeado/transferido mata o webhook em silêncio** (o `git push` continua funcionando
   por redirect, o webhook não): merge em `master` passa a não publicar nada.

   `vercel --prod` na raiz com `rootDirectory=frontend` no painel falha de dois jeitos:
   `File size limit exceeded (100 MB)` (resolver com `.vercelignore`) e
   `Root Directory "frontend" does not exist` (acontece quando o `.vercel` está linkado dentro de
   `frontend/` — rode o `vercel link` na raiz).

9. **Sonde rotas que usam banco E rotas que não usam, nos dois ambientes.** A matriz é o
   diagnóstico: `/health` e `/api/auth/*` em 200 com **todas** as rotas de dado em 500 é processo
   vivo com banco inacessível (credencial, projeto pausado ou pool) — não é bug de código nem
   build velho. Latência constante de alguns segundos em toda rota de dado confirma tentativa de
   conexão. Receita em `references/deploy-verification.md` §"Backend vivo, banco morto".
10. **O documento de estado envelhece.** `.planning/STATE.md`, `PLAN.md` de quick task e runbook
    dizem onde a última sessão achou que estava; `git`, o log do provedor e a sonda HTTP dizem onde
    está. Quando divergirem, o documento é hipótese e a sonda é evidência — relate o que mediu, não
    a data do documento.

Feche com as pendências reais e uma pergunta de próximo passo — não com um resumo do que
foi verificado. Liste as pendências **em ordem de dependência** numa tabela com uma coluna de dono:
separe o que só o dono pode fazer (reativar serviço no painel, rotacionar chave, aplicar migration)
do que o agente faz sozinho (promover, rodar gate, limpar branch). Sem essa separação o relatório
vira lista de bloqueios sem próximo passo executável.

## 9. Merge conflict resolution

Resolver conflitos ficando com o código mais novo em cada arquivo. Depois de
resolver, rodar `git diff --check` para garantir que não sobrou marcador de
conflito. Depois `git add -A` e commit da resolução.

## 10. Edição de arquivos com ferramentas

`read_file` com paginação (offset/limit) deixa o arquivo em estado de **leitura parcial**.
Se tentar `write_file`/`patch` sem reler o arquivo completo primeiro, a ferramenta pode
bloquear a escrita com `stale_write_blocked`. Sempre releia o arquivo completo antes de
reescrevê-lo.

**`write_file` recusa reescrever um arquivo que você só viu parcialmente — e a mensagem
pede "read every page" mesmo quando o arquivo tem 20 linhas.** Ler de novo com `read_file`
sem `offset`/`limit` normalmente destrava. Se a recusa persistir mesmo após a releitura
completa (acontece depois de um `patch` no meio do arquivo), o caminho que funciona é
`rm <arquivo>` e criar de novo com `write_file` — o arquivo é novo, sem histórico de leitura.
Só faça isso em arquivo que você acabou de ler por inteiro, e nunca em arquivo fora do
repositório.

**`patch` em bloco com texto parecido pode INSERIR em vez de substituir**, duplicando uma
função ou um bloco inteiro. O sintoma aparece longe do edit (`Parse error`, função duplicada,
teste falhando em nó que existe). Depois de qualquer `patch` em arquivo com trechos
repetidos, confirme a estrutura — `grep -n "^func " arquivo.gd` ou equivalente — antes de
rodar a suíte.

**Arquivo com header de autenticação não satura a leitura: redação bloqueia `write_file`.**
`read_file` e a saída do terminal substituem `Authorization: Bearer <token>` por marcador
(`***`) antes de te mostrar. Se o arquivo contém essa string — `client.ts` de integração,
test de adapter, `.env` de exemplo — a leitura **nunca** conta como completa, e `write_file`
recusa para sempre com `stale_write_blocked` mesmo depois de reler por inteiro. Não insista
em reler, e não tente reescrever o arquivo por fora para "ver cru": o mascaramento acontece
na saída da ferramenta, não no arquivo em disco, então o texto real nunca chega até você.
O caminho que funciona é apagar e recriar — `rm <arquivo>` seguido de `write_file` com o
conteúdo novo. Use só em arquivo que você já conhece por inteiro (leu antes, ou acabou de
escrever), nunca em arquivo fora do repositório ou que você não viu.

## 11. Onde esta skill vive

`github.com/andrelbuzzo/rocketify-rules` — fonte única da verdade. Instalada globalmente
em `~/.config/opencode/skills/rocketify-rules/`. Atualizar: `git pull` nesse diretório.
Projetos novos herdam automaticamente.

## 12. Web Design Vibrante - Olfatto

### Padrões de design visual para landing pages e admin panels

**Core Principle:** O usuário prefere designs vibrantes com gradientes dinâmicos, inspirados em referências como Droper.app, em vez de designs básicos.

### CSS Custom Properties obrigatórias para web vibrante

```css
:root {
  --vibrant-pink: oklch(70% 0.18 35);
  --vibrant-purple: oklch(68% 0.16 310);
  --vibrant-blue: oklch(65% 0.14 250);
  --gradient-primary: linear-gradient(135deg, var(--vibrant-pink) 0%, var(--vibrant-purple) 70%, var(--vibrant-blue) 100%);
  --radius-sm: 8px;
  --radius-card: 12px;
}
```

### Animações essenciais

```css
@keyframes shimmer { 0% { background-position: -200% 0; } 100% { background-position: 200% 0; } }
@keyframes pulse-glow { 0%, 100% { box-shadow: 0 0 0 0 oklch(70% 0.18 35 / 0.4); } 50% { box-shadow: 0 0 0 20px oklch(70% 0.18 35 / 0); } }
```

### Fluxo de validação visual

1. Verificar em staging antes do push final
2. Build deve passar: `pnpm --filter @olfatto/web build` → exit 0
3. Verificar elementos no DOM com curl ou acesso físico

### Componentes a updatear com classes vibraantes

- **Hero section**: gradientes dinâmicos, shimmer no título
- **Cards de ranking**: hover transform + borda primária
- **Admin cards**: shimmer no hover, border-left animado
- **Tabela**: hover colorido com transform translateX