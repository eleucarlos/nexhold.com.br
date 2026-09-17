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
  Emissão oficial: **até 1h**. Estado em 17/09/2026: **emitido** (`CN = www.nexhold.com.br`)
  e **`Enforce HTTPS` ativo** — HTTP responde `301` para HTTPS.
- Se o certificado travar (API retorna `404 "The certificate does not exist yet"` por
  mais de 1h): remover e re-adicionar o domínio customizado re-dispara a emissão:

```bash
gh api repos/eleucarlos/nexhold.com.br/pages -X PUT --input - <<'EOF'
{"cname": null}
EOF
sleep 5
gh api repos/eleucarlos/nexhold.com.br/pages -X PUT --input - <<'EOF'
{"cname": "www.nexhold.com.br"}
EOF
```

- Conferir cert na borda:

```bash
echo | openssl s_client -connect www.nexhold.com.br:443 -servername www.nexhold.com.br 2>/dev/null \
  | openssl x509 -noout -subject
# ok quando mostra: CN = www.nexhold.com.br  (CN = *.github.io = ainda emitindo)
```

- `Enforce HTTPS` (só funciona com certificado emitido):

```bash
gh api repos/eleucarlos/nexhold.com.br/pages -X PUT --input - <<'EOF'
{"https_enforced": true}
EOF
```

- **Renovação: automática, feita pelo GitHub.** Let's Encrypt emite certificados de
  90 dias; o GitHub Pages renova sozinho semanas antes de expirar. Não existe cron,
  servidor ou script nosso — zero manutenção.
- Condições para a renovação não falhar:
  1. CNAME `www → eleucarlos.github.io` continuar no DNS (DNS only, cinza)
  2. Arquivo `CNAME` continuar na raiz do repo
  3. Se um dia criar CAA no apex, incluir `letsencrypt.org`
- Sintoma de renovação falha: aviso de certificado no navegador perto do vencimento.
  Correção: re-trigger acima (remove/re-adiciona domínio).
- Não existe CAA no apex; o CAA herdado de `eleucarlos.github.io` já permite
  `letsencrypt.org`. Se um dia criar CAA em `nexhold.com.br`, incluir `letsencrypt.org`.

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
