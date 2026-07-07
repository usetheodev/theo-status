# CLAUDE.md — theo-status

Contrato entre Claude e o repositório da **status page do ecossistema Theo** (Upptime). Leia antes de tocar em qualquer arquivo.

Este arquivo é camada sobre `/home/paulo/.claude/CLAUDE.md` (global, regras invioláveis). O que conflitar aqui com o global, **o global vence** — exceto o modelo de branch (§ Git abaixo), que é deliberadamente diferente do padrão develop/main dos outros repos.

---

## Missão

Este repo **é** a infraestrutura de status — não tem código de aplicação. Ele existe para uma coisa: provar com histórico auditável que os serviços do Theo estão no ar (ou admitir publicamente quando não estão). Nasceu em 2026-07-07 para fechar o achado C8 do painel de 18 personas (`theo-website/.claude/persona-validation/2026-07-07-full-production-sr6-gate.md`) e sustentar, com evidência acumulada, o "99.9% SLA target" do site.

- **Página pública:** https://status.usetheo.dev
- **Motor:** [Upptime](https://upptime.js.org) — GitHub Actions (checks a cada 5 min) + Issues (incident log) + Pages (site no branch `gh-pages`).
- **Princípio inegociável:** a status page vive FORA da infraestrutura que monitora. Nada aqui pode depender de TheoCloud, do projeto Cloudflare Pages do site, ou de qualquer serviço monitorado.

## O que Claude PODE fazer aqui

- Editar `.upptimerc.yml` — endpoints, branding do site, navbar, notificações. É o único arquivo de configuração que importa.
- Editar `README.md` — MAS preservando byte a byte os blocos auto-gerados (ver § Regras).
- Editar este `CLAUDE.md`.
- Adicionar/remover endpoints monitorados (com a regra do § Endpoints).

## O que Claude NÃO PODE fazer

- **Editar workflows em `.github/`** — são sincronizados do template upstream (`.templaterc.json` → `files: [".github/**/*"]`); qualquer edição local é sobrescrita no próximo template sync. Comportamento se customiza via `.upptimerc.yml`.
- **Editar `history/`, `api/`, `graphs/`** — são dados escritos pelo bot. O histórico é o produto; editar histórico é fraude de uptime (viola a Regra 3 global e a /honesty do site).
- **Editar a tabela de status ou a linha "Live Status" do README** — o Summary CI as reescreve; edições são perdidas ou geram conflito.
- **Fechar/editar issues de incidente na mão** para "limpar" o log — incidentes abrem e fecham sozinhos; o log público é o ponto.
- **Despróxiar o DNS** de `status.usetheo.dev` na Cloudflare (ver § Infra).

## Git — modelo deste repo (DIFERENTE do padrão da org)

- **Branch única `main`**, sem develop, sem PRs de release. O bot do Upptime commita direto em `main` a cada ciclo (5 min) — um fluxo develop→main travaria o design da ferramenta.
- **SEMPRE `git pull --rebase origin main` antes de qualquer push.** O remoto anda sozinho; push sem rebase = rejeitado. Se o rebase conflitar no README (você e o Summary CI editaram ao mesmo tempo), a resolução é: versão do bot como base, sua prosa enxertada em volta dos marcadores (precedente: commit `605a9eb`).
- Continuam valendo do global: `git switch`/`git restore` (nunca checkout), nunca `push --force`, nunca `reset --hard`.
- Branch `gh-pages` é 100% do bot (deploy do site) — nunca tocar.

## Endpoints (§ regra de adição)

Monitorados hoje (2026-07-07): `usetheo.dev`, `docs.usetheo.dev`, `blog.usetheo.dev`.

- **Antes de adicionar um endpoint, verifique que ele resolve e responde** (`curl -o /dev/null -w "%{http_code}"`). Monitor que nasce vermelho é ruído, não sinal — foi por isso que `get.usetheo.dev` ficou de fora (domínio não resolvia em 2026-07-07; adicionar quando shippar).
- Candidatos futuros óbvios: endpoints públicos do TheoCloud (API, painel) quando houver URL estável; `get.usetheo.dev` quando existir.
- Remover um endpoint monitorado apaga a visibilidade, não o histórico — o histórico fica em `history/` para sempre.

## Infra (como está montado — não redescobrir)

| Peça | Estado |
|---|---|
| DNS | CNAME `status` → `usetheodev.github.io`, **proxied (nuvem laranja)** na zona `usetheo.dev` (id `9f9a41bef0bcb7adf15969c81c303bef`) |
| HTTPS | Servido pelo **edge da Cloudflare** (cert `CN=usetheo.dev`); modo SSL da zona = `full`, que aceita o cert `*.github.io` da origem |
| Cert do GitHub Pages | **Nunca emitiu** (ficou preso em `new` — health check do GitHub vê IPs da CF). Irrelevante enquanto proxied. **Não despróxiar sem plano**: sem a CF na frente, volta `ERR_CERT_COMMON_NAME_INVALID` |
| `https_enforced` no Pages | Off (impossível sem o cert do GitHub) — o enforce de HTTPS pode ser feito na CF ("Always Use HTTPS" da zona) se desejado |
| Actions | **Roda de graça neste repo (público)** apesar do billing quebrado da org — o bloqueio só afeta repos privados. `default_workflow_permissions` do repo foi setado para `write` via API (a org default é read-only) |
| GH_PAT | Não configurado — os workflows usam o fallback `github.token`. Se commits do bot pararem de disparar rebuilds do site, criar PAT fine-grained e salvar como secret `GH_PAT` |

## Incidentes (quando um endpoint cair)

1. **Não apague nem edite a issue** que o bot abriu — ela é o registro público.
2. Comente na issue com contexto humano (o que houve, o que foi feito) — isso vira a timeline do incidente na página.
3. Sev-1 no ecossistema pede post-mortem em até 5 dias úteis (compromisso público do site em `/status` e `/honesty`).
4. Falso positivo recorrente (endpoint saudável marcado down) → ajustar o check no `.upptimerc.yml` (ex.: `expectedStatusCodes`, `maxResponseTime`) em vez de remover o endpoint.

## Relação com o theo-website

- A página `/status` do usetheo.dev ainda é um snapshot manual — há pendência registrada de linká-la (ou redirecioná-la) para cá.
- O texto de marketing do site promete "incident log público no GitHub" — este repo é o cumprimento dessa promessa. Se o site mudar essa promessa, este repo é a fonte da verdade do que é tecnicamente verdadeiro.
- Claims de uptime no site (ex.: evoluir "99.9% target" para número medido) devem citar os dados de `history/`/`api/` deste repo — nunca números soltos.

## Anti-patterns

- Editar workflow para "consertar" comportamento → é sobrescrito pelo template sync; use `.upptimerc.yml`.
- Commitar sem `pull --rebase` → rejeitado; o remoto anda a cada 5 min.
- Adicionar endpoint que não existe ainda "para já ficar pronto" → monitor nasce vermelho, polui o uptime agregado e as issues.
- Reescrever o README inteiro sem preservar os marcadores `<!--live status-->` e `<!--start/end: status pages-->` → o Summary CI reverte ou conflita.
- "Limpar" issues de incidente antigas → o log sujo É o produto.
