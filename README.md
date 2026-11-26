# gnc
site de noticias
#!/usr/bin/env bash
# =============================================================================
#  deploy.sh - Publica o portal GNC no GitHub Pages
#  Rode dentro da pasta que contém o index.html.   Ajuda: bash deploy.sh --help
# =============================================================================
set -Eeuo pipefail

# ------------------------------- configuração --------------------------------
REPO="gnc"
BRANCH="main"
MESSAGE=""
TAG=""
VISIBILITY="public"
DRY_RUN=0
WAIT=1
RETRIES=3
WAIT_SECONDS=180

# --------------------------------- aparência ---------------------------------
if [ -t 1 ] && command -v tput >/dev/null 2>&1 && [ "$(tput colors 2>/dev/null || echo 0)" -ge 8 ]; then
  C_RED=$'\e[31m'; C_GRN=$'\e[32m'; C_YLW=$'\e[33m'; C_BLU=$'\e[34m'; C_DIM=$'\e[2m'; C_RST=$'\e[0m'
else
  C_RED=""; C_GRN=""; C_YLW=""; C_BLU=""; C_DIM=""; C_RST=""
fi
STEP=0
step() { STEP=$((STEP + 1)); printf '\n%s[%s]%s %s\n' "$C_BLU" "$STEP" "$C_RST" "$*"; }
info() { printf '    %s\n' "$*"; }
ok()   { printf '    %s✔%s %s\n' "$C_GRN" "$C_RST" "$*"; }
warn() { printf '    %s!%s %s\n' "$C_YLW" "$C_RST" "$*" >&2; }
die()  { printf '\n%s✘ Erro:%s %s\n' "$C_RED" "$C_RST" "$*" >&2; exit 1; }
trap 'printf "\n%s✘ Falhou na linha %s: %s%s\n" "$C_RED" "$LINENO" "$BASH_COMMAND" "$C_RST" >&2' ERR

# Executa um comando; em --dry-run só mostra o que seria feito.
run() {
  if [ "$DRY_RUN" -eq 1 ]; then
    printf '    %s[simulação] %s%s\n' "$C_DIM" "$*" "$C_RST"
  else
    "$@"
  fi
}

usage() {
  cat <<USAGE
Uso: bash deploy.sh [opções]

Opções:
  -r, --repo NOME       Nome do repositório (padrão: $REPO)
  -b, --branch NOME     Branch de publicação (padrão: $BRANCH)
  -m, --message TEXTO   Mensagem do commit (padrão: data e hora)
  -t, --tag VERSAO      Cria e envia uma tag, ex.: v1.0.0
      --private         Cria o repositório como privado (Pages em repositório
                        privado exige plano pago do GitHub)
      --no-wait         Não espera o site ficar no ar
  -n, --dry-run         Mostra o que seria feito, sem alterar nada
  -h, --help            Mostra esta ajuda

Exemplos:
  bash deploy.sh
  bash deploy.sh -m "Nova notícia de esportes"
  bash deploy.sh --repo gnc --tag v1.0.0
  bash deploy.sh --dry-run
USAGE
}

# --------------------------------- argumentos --------------------------------
while [ $# -gt 0 ]; do
  case "$1" in
    -r|--repo)    REPO="${2:?Falta o valor de $1}"; shift 2 ;;
    -b|--branch)  BRANCH="${2:?Falta o valor de $1}"; shift 2 ;;
    -m|--message) MESSAGE="${2:?Falta o valor de $1}"; shift 2 ;;
    -t|--tag)     TAG="${2:?Falta o valor de $1}"; shift 2 ;;
    --private)    VISIBILITY="private"; shift ;;
    --no-wait)    WAIT=0; shift ;;
    -n|--dry-run) DRY_RUN=1; shift ;;
    -h|--help)    usage; exit 0 ;;
    *)            usage; die "Opção desconhecida: $1" ;;
  esac
done

[ -n "$MESSAGE" ] || MESSAGE="Atualiza GNC - $(date '+%Y-%m-%d %H:%M')"
[ "$DRY_RUN" -eq 1 ] && printf '%s*** MODO SIMULAÇÃO: nada será alterado ***%s\n' "$C_YLW" "$C_RST"

# ------------------------------ verificações ---------------------------------
step "Verificando o ambiente"

command -v git >/dev/null 2>&1 || die "Git não encontrado. Instale em https://git-scm.com e rode de novo."
ok "git $(git --version | awk '{print $3}')"

[ -f index.html ] || die "index.html não encontrado em $(pwd). Rode o script dentro da pasta do site."
ok "index.html encontrado"

[[ "$REPO" =~ ^[A-Za-z0-9._-]+$ ]] || die "Nome de repositório inválido: '$REPO' (use letras, números, ponto, hífen ou sublinhado)."
[[ "$BRANCH" =~ ^[A-Za-z0-9._/-]+$ ]] || die "Nome de branch inválido: '$BRANCH'."
[ -z "$TAG" ] || [[ "$TAG" =~ ^[A-Za-z0-9._-]+$ ]] || die "Nome de tag inválido: '$TAG'."

grep -qi "<title>" index.html || warn "O index.html não tem <title>."
grep -qi 'name="viewport"' index.html || warn "O index.html não tem a meta viewport (o site pode ficar ruim no celular)."
grep -q "Texto de exemplo" index.html && warn "A nota 'Texto de exemplo' ainda está na página. Remova antes de divulgar."

USE_GH=0
if command -v gh >/dev/null 2>&1 && gh auth status >/dev/null 2>&1; then
  USE_GH=1
  GH_USER="$(gh api user --jq .login)"
  ok "GitHub CLI logado como $GH_USER"
else
  warn "GitHub CLI (gh) ausente ou sem login. Vou usar o modo manual."
fi

# --------------------------- arquivos de apoio -------------------------------
step "Preparando os arquivos do site"

if [ ! -f .nojekyll ]; then
  run touch .nojekyll
  ok "Criado .nojekyll (o Pages publica os arquivos como estão)"
fi

if [ ! -f .gitignore ]; then
  if [ "$DRY_RUN" -eq 0 ]; then
    printf '.DS_Store\nThumbs.db\n*.log\n*.swp\nnode_modules/\n' > .gitignore
  fi
  ok "Criado .gitignore"
fi

if [ ! -f 404.html ]; then
  if [ "$DRY_RUN" -eq 0 ]; then
    cat > 404.html <<'HTML'
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Página não encontrada - GNC</title>
<style>
body{margin:0;min-height:100vh;display:grid;place-items:center;font-family:system-ui,sans-serif;background:#F3F5F8;color:#12161D;text-align:center;padding:24px}
h1{font-size:4rem;margin:0;color:#0A3CC9;letter-spacing:-.04em}
a{color:#0A3CC9;font-weight:600}
</style>
</head>
<body>
<main>
<h1>404</h1>
<p>Não encontramos essa página.</p>
<p><a href="./">Voltar para a GNC</a></p>
</main>
</body>
</html>
HTML
  fi
  ok "Criado 404.html"
fi

# ------------------------------ repositório git ------------------------------
step "Preparando o repositório local"

if [ ! -d .git ]; then
  run git init -b "$BRANCH"
  ok "Repositório iniciado na branch $BRANCH"
fi

if [ "$DRY_RUN" -eq 0 ]; then
  CURRENT="$(git symbolic-ref --short -q HEAD || echo "$BRANCH")"
  if [ "$CURRENT" != "$BRANCH" ]; then
    git checkout -B "$BRANCH" >/dev/null 2>&1
    ok "Trocado para a branch $BRANCH"
  fi

  if [ -z "$(git config user.name || true)" ] || [ -z "$(git config user.email || true)" ]; then
    warn "Seu nome/e-mail do Git não está configurado."
    read -r -p "    Seu nome: " GIT_NAME
    read -r -p "    Seu e-mail: " GIT_EMAIL
    [ -n "$GIT_NAME" ] && [ -n "$GIT_EMAIL" ] || die "Nome e e-mail são obrigatórios para o commit."
    git config user.name "$GIT_NAME"
    git config user.email "$GIT_EMAIL"
  fi
fi

# ---------------------------------- commit -----------------------------------
step "Registrando as alterações"

run git add -A
if [ "$DRY_RUN" -eq 1 ] || ! git diff --cached --quiet || ! git rev-parse --verify HEAD >/dev/null 2>&1; then
  run git commit -m "$MESSAGE"
  ok "Commit: $MESSAGE"
else
  info "Nenhuma alteração nova para commitar."
fi

# ------------------------------ remoto / GitHub ------------------------------
step "Configurando o repositório no GitHub"

HAS_ORIGIN=0
git remote get-url origin >/dev/null 2>&1 && HAS_ORIGIN=1

if [ "$HAS_ORIGIN" -eq 0 ]; then
  if [ "$USE_GH" -eq 1 ]; then
    if [ "$DRY_RUN" -eq 0 ] && gh repo view "$GH_USER/$REPO" >/dev/null 2>&1; then
      git remote add origin "https://github.com/$GH_USER/$REPO.git"
      ok "Repositório $GH_USER/$REPO já existe; conectado."
    else
      run gh repo create "$REPO" "--$VISIBILITY" --source=. --remote=origin
      ok "Repositório $REPO criado ($VISIBILITY)"
    fi
  else
    printf '    Crie um repositório VAZIO chamado "%s" em https://github.com/new\n' "$REPO"
    if [ "$DRY_RUN" -eq 1 ]; then
      GH_USER="SEU-USUARIO"
    else
      read -r -p "    Digite seu usuário do GitHub: " GH_USER
      [ -n "$GH_USER" ] || die "Usuário do GitHub é obrigatório."
      git remote add origin "https://github.com/$GH_USER/$REPO.git"
    fi
    ok "Remoto configurado"
  fi
else
  ok "Remoto já configurado: $(git remote get-url origin)"
  # Descobre o usuário a partir da URL do remoto, se ainda não souber.
  if [ -z "${GH_USER:-}" ]; then
    GH_USER="$(git remote get-url origin | sed -nE 's#.*github\.com[:/]([^/]+)/.*#\1#p')"
  fi
fi

# ---------------------------------- envio ------------------------------------
step "Enviando para o GitHub"

push_with_retry() {
  local attempt=1
  while [ "$attempt" -le "$RETRIES" ]; do
    if git push -u origin "$BRANCH"; then
      return 0
    fi
    warn "Envio falhou (tentativa $attempt de $RETRIES)."
    if [ "$attempt" -eq 1 ] && git ls-remote --exit-code --heads origin "$BRANCH" >/dev/null 2>&1; then
      info "Sincronizando com as mudanças que já estão no GitHub..."
      git pull --rebase origin "$BRANCH" || die "Conflito ao sincronizar. Resolva manualmente com 'git status'."
    fi
    attempt=$((attempt + 1))
    sleep $((attempt * 2))
  done
  return 1
}

if [ "$DRY_RUN" -eq 1 ]; then
  run git push -u origin "$BRANCH"
else
  push_with_retry || die "Não consegui enviar o código. Confira sua conexão e suas permissões no GitHub."
  ok "Código enviado"
fi

if [ -n "$TAG" ]; then
  if [ "$DRY_RUN" -eq 0 ] && git rev-parse -q --verify "refs/tags/$TAG" >/dev/null; then
    warn "A tag $TAG já existe; não foi recriada."
  else
    run git tag -a "$TAG" -m "Versão $TAG"
    run git push origin "$TAG"
    ok "Tag $TAG enviada"
  fi
fi

# --------------------------------- GitHub Pages ------------------------------
step "Ativando o GitHub Pages"

SITE_URL=""
[ -n "${GH_USER:-}" ] && SITE_URL="https://${GH_USER}.github.io/${REPO}/"

if [ "$USE_GH" -eq 1 ]; then
  if [ "$DRY_RUN" -eq 1 ]; then
    run gh api -X POST "repos/$GH_USER/$REPO/pages"
  elif gh api "repos/$GH_USER/$REPO/pages" >/dev/null 2>&1; then
    ok "Pages já estava ativo"
  elif gh api -X POST "repos/$GH_USER/$REPO/pages" \
        -f "source[branch]=$BRANCH" -f "source[path]=/" >/dev/null 2>&1; then
    ok "Pages ativado (branch $BRANCH, pasta raiz)"
  else
    warn "Não consegui ativar o Pages automaticamente."
    warn "Ative em: Settings > Pages > Branch '$BRANCH' e pasta '/ (root)' > Save."
  fi
else
  warn "Ative manualmente: Settings > Pages > Branch '$BRANCH' e pasta '/ (root)' > Save."
fi

# ------------------------------ espera o site --------------------------------
if [ "$WAIT" -eq 1 ] && [ "$DRY_RUN" -eq 0 ] && [ -n "$SITE_URL" ] && command -v curl >/dev/null 2>&1; then
  step "Aguardando o site ficar no ar (até ${WAIT_SECONDS}s)"
  elapsed=0
  LIVE=0
  while [ "$elapsed" -lt "$WAIT_SECONDS" ]; do
    code="$(curl -s -o /dev/null -w '%{http_code}' -L "$SITE_URL" || echo 000)"
    if [ "$code" = "200" ]; then LIVE=1; break; fi
    printf '    %s... ainda publicando (HTTP %s)%s\r' "$C_DIM" "$code" "$C_RST"
    sleep 6
    elapsed=$((elapsed + 6))
  done
  printf '\n'
  if [ "$LIVE" -eq 1 ]; then
    ok "Site no ar!"
  else
    warn "Ainda não respondeu. O primeiro deploy pode levar alguns minutos; tente abrir o link daqui a pouco."
  fi
fi

# ---------------------------------- resumo -----------------------------------
printf '\n%s================ Resumo ================%s\n' "$C_GRN" "$C_RST"
printf '  Repositório : %s\n' "$(git remote get-url origin 2>/dev/null || echo "$REPO")"
printf '  Branch      : %s\n' "$BRANCH"
[ -n "$TAG" ] && printf '  Tag         : %s\n' "$TAG"
[ -n "$SITE_URL" ] && printf '  Site        : %s\n' "$SITE_URL"
[ "$DRY_RUN" -eq 1 ] && printf '  %s(simulação: nada foi alterado)%s\n' "$C_YLW" "$C_RST"
printf '\n'
