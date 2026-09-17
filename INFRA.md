# Infraestrutura do site — www.nexhold.com.br

> Documento vivo da infra do site institucional. Atualizar junto com qualquer mudança
> de DNS, hospedagem ou domínio. Atualizado em 17/09/2026.

---

## Visão geral

```
visitante
   │
   ▼
registro.br  ── delega DNS para ──▶  Cloudflare (benedict/natasha.ns.cloudflare.com)
                                        │  CNAME www → eleucarlos.github.io (DNS only)
                                        ▼
                                   GitHub Pages (repo eleucarlos/nexhold.com.br)
                                        │  branch main, path /
                                        ▼
                                   CDN Fastly + cert Let's Encrypt
```

- **Site:** página única estática, `index.html` autocontido (HTML + CSS + JS inline).
  Sem framework, sem build, sem dependências. Deploy = `git push` na `main`.
- **Hospedagem:** GitHub Pages, repositório público `eleucarlos/nexhold.com.br`,
  source `main` / `/`, build type `legacy`.
- **Domínio customizado:** `www.nexhold.com.br`, via arquivo `CNAME` na raiz do repo.
- **DNS:** autoritativo na **Cloudflare** (não no registro.br).
- **Registro do domínio:** registro.br — só cuida de titularidade/renovação e da
  delegação dos nameservers. Nenhum registro DNS é criado lá.

## Componentes e responsabilidades

| Componente | Papel | Onde se configura |
|---|---|---|
| registro.br | Registro/renovação de `nexhold.com.br` e delegação NS → Cloudflare | painel registro.br |
| Cloudflare | DNS autoritativo do domínio | dash.cloudflare.com → nexhold.com.br → DNS |
| GitHub Pages | Hospedagem estática + CDN + certificado TLS | repo → Settings → Pages |
| Repo `nexhold.com.br` | Código-fonte do site | github.com/eleucarlos/nexhold.com.br |

## Registros DNS (Cloudflare)

| Tipo | Nome | Conteúdo | Proxy |
|---|---|---|---|
| `CNAME` | `www` | `eleucarlos.github.io` | **DNS only (cinza)** |

Regras:

- **Proxy sempre desabilitado (nuvem cinza).** Proxy laranja impede o GitHub de
  verificar o domínio e emitir/renovar o certificado, e quebra o `Enforce HTTPS`.
- Apex (`nexhold.com.br`) **não configurado** — só `www` responde. Se um dia quiser
  o apex: 4 registros `A` em `@` → `185.199.108.153`, `.109.153`, `.110.153`,
  `.111.153`, todos DNS only.

## TLS / HTTPS

- Certificado Let's Encrypt emitido pelo próprio GitHub após o CNAME propagar.
  Emissão leva de minutos a ~1h após o DNS.
- `Enforce HTTPS`: ativar **somente depois** do certificado existir:

```bash
gh api repos/eleucarlos/nexhold.com.br/pages -X PUT --input - <<'EOF'
{"https_enforced": true}
EOF
```

Se retornar `404 "The certificate does not exist yet"`, o cert ainda não saiu —
aguardar e tentar de novo. Status atual: `gh api repos/eleucarlos/nexhold.com.br/pages`.

## Deploy / como alterar o site

```bash
cd site-nexhold          # clone local do repo
# editar index.html
git add index.html
git commit -m "..."
git push                 # Pages rebuilda sozinho (~30s)
```

Conferir build: `gh api repos/eleucarlos/nexhold.com.br/pages | jq .status`
(deve ficar `built`).

## Como o index.html funciona

- **i18n:** dicionário `I18N` no `<script>` com chaves `pt`/`en`. Elementos usam
  `data-i18n="chave"` (texto puro) ou `data-i18n-html="chave"` (HTML com tags).
  Botões 🇧🇷/🇺🇸 no topo trocam o idioma; escolha persiste em
  `localStorage["nx-lang"]`. Padrão: `pt`.
- **Tema:** 3 estados — `system` (segue `prefers-color-scheme` do SO/navegador),
  `light`, `dark`. Botão 🖥️ cicla system → light → dark. Persiste em
  `localStorage["nx-theme"]`. Cores via CSS custom properties em `:root` e
  `[data-theme="light"]`.
- **Bandeiras/ícones:** SVG inline (emoji de bandeira não renderiza em parte dos
  sistemas — não voltar para emoji).
- **Animação:** `IntersectionObserver` adiciona `.in` nos elementos `.reveal`.
- **Domínio:** arquivo `CNAME` contém `www.nexhold.com.br` — se apagar, o Pages
  perde o domínio customizado.

## Armadilhas conhecidas

1. **Não criar DNS no registro.br** — o domínio está delegado à Cloudflare;
   registros no registro.br não têm efeito.
2. **Não ativar proxy laranja** no CNAME do `www`.
3. **Não apagar o arquivo `CNAME`** do repo.
4. `Enforce HTTPS` só funciona depois que o GitHub emite o certificado.
5. Repo precisa permanecer **público** (Pages no plano Free).

## Comandos úteis

```bash
# DNS resolvendo?
dig +short www.nexhold.com.br        # esperado: eleucarlos.github.io + 4 IPs 185.199.x

# site no ar?
curl -sI https://www.nexhold.com.br | head -3

# estado do Pages
gh api repos/eleucarlos/nexhold.com.br/pages | jq '{status, cname, https_enforced}'
```
