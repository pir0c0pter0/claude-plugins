# pir0c0pter0/claude-plugins

Marketplace pessoal de plugins [Claude Code](https://claude.ai/code) mantidos por [@pir0c0pter0](https://github.com/pir0c0pter0).

> **Status:** plugins prontos para uso. Cada um é documentado individualmente dentro da sua pasta.

## 📦 Plugins disponíveis

| Plugin | Versão | Descrição |
|---|---|---|
| [`qwen-review`](./qwen-review) | 0.1.0 | Stop-time review gate via API Qwen 3.7 Max — bloqueia stop quando o Qwen identifica problema no turn anterior. |

---

## 🚀 Instalação (passo a passo)

### Opção A — via slash command (recomendado, mais fácil)

Dentro de uma sessão Claude Code:

```
/plugin marketplace add pir0c0pter0/claude-plugins
```

Pronto — Claude Code registra a marketplace, baixa o repo, lista os plugins disponíveis. Daí:

```
/plugin install qwen-review@pir0c0pter0
```

(Alguns versões usam `/plugin` interativo com aba Marketplaces — abre menu pra escolher.)

### Opção B — editando `settings.json` manualmente

Útil pra setups compartilhados em team (commita junto). Edite `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "pir0c0pter0": {
      "source": {
        "source": "github",
        "repo": "pir0c0pter0/claude-plugins"
      }
    }
  },
  "enabledPlugins": {
    "qwen-review@pir0c0pter0": true
  }
}
```

> ⚠️ Por essa rota, a primeira vez exige **restart do Claude Code** (não basta `/reload-plugins` — registro de marketplaces só recarrega no startup). Atualizações posteriores dos plugins funcionam com `/reload-plugins`.

### Confirmar instalação (qualquer opção)

```
/plugin list
```

Deve aparecer `qwen-review@pir0c0pter0` ✅.

Se der erro, rode `/doctor`.

### 2. Configurar o plugin (use o wizard)

```
! node ~/.claude/plugins/cache/pir0c0pter0/claude-plugins/<versão>/qwen-review/scripts/qwen-review.mjs wizard
```

O `!` é importante — abre terminal real (necessário pro readline funcionar). Veja [seção do wizard abaixo](#-wizard-de-configuração).

Como descobrir o `<versão>`:
```bash
ls ~/.claude/plugins/cache/pir0c0pter0/claude-plugins/
```

### 3. Habilitar o gate no workspace

Dentro do projeto onde você quer review automático:

```
/qwen-review:setup --enable
```

O gate é por workspace — habilita só no projeto atual (state isolado por SHA-256 do realpath do diretório).

### 4. Confirmar tudo funcionando

```
/qwen-review:status
```

Deve mostrar `envOk: true`, `reviewGateEnabled: true`, e os valores que o wizard escreveu.

---

## 🧙 Wizard de configuração

O wizard é a forma recomendada de configurar `QWEN_API_KEY`, base URL, modelo e modo. Roda interativamente, mostra summary, pede confirmação, e escreve atomicamente em `~/.claude/settings.json` preservando todos os outros campos.

### Como rodar

```
! node ~/.claude/plugins/cache/pir0c0pter0/claude-plugins/<versão>/qwen-review/scripts/qwen-review.mjs wizard
```

> O **prefixo `!`** é essencial. Sem ele, o Claude Code executa via Bash tool (sem TTY) e os prompts do `readline` ficam vazios. Com `!`, o comando roda no terminal real do usuário e os prompts funcionam.

### O que ele pergunta

```
qwen-review setup wizard
────────────────────────
Reading current config from /home/user/.claude/settings.json

Current API key: sk-•••efd                  ← mostra mascarada se já existe
New API key (blank to keep current): ▌      ← deixe vazio pra manter

Base URL options:
  1) DashScope International — https://dashscope-intl.aliyuncs.com/compatible-mode/v1
  2) DashScope China — https://dashscope.aliyuncs.com/compatible-mode/v1
  3) OpenRouter — https://openrouter.ai/api/v1
  4) Custom (enter manually)
Choose 1-4 [1]: ▌                            ← default na escolha atual

Model [qwen3-max]: ▌

Review mode:
  fast — 1024 tokens, ~3-15s, sem thinking (default — recomendado pro dia-a-dia)
  thinking — 8192 tokens, ~60-180s, enable_thinking=true (review profundo)
Mode (fast|thinking) [fast]: ▌

────────────────────────
Will write to env:
  QWEN_API_KEY      = sk-•••efd
  QWEN_BASE_URL     = https://dashscope-intl.aliyuncs.com/compatible-mode/v1
  QWEN_MODEL        = qwen3-max
  QWEN_REVIEW_MODE  = thinking
Target: /home/user/.claude/settings.json

Write these values? [y/N] ▌
```

Confirma com `y`. Settings.json escrito atomicamente com mode `0o600` (owner-only, mesmo se você tinha `0o644` antes — é arquivo de credenciais).

### O que ele faz com o `settings.json`

Pega:
```json
{ "permissions": {...}, "env": {"OUTRO": "x"}, "enabledPlugins": {...} }
```

Vira (mantém o resto, mescla apenas as 4 chaves do qwen no env):
```json
{
  "permissions": {...},
  "env": {
    "OUTRO": "x",
    "QWEN_API_KEY": "sk-...",
    "QWEN_BASE_URL": "...",
    "QWEN_MODEL": "qwen3-max",
    "QWEN_REVIEW_MODE": "thinking"
  },
  "enabledPlugins": {...}
}
```

**Garantias de segurança** (vem dos 4-5 turnos do Codex review gate):
- Mode final do arquivo sempre `0o600` (owner-only, independente de umask)
- Credenciais nunca chegam ao disco em um tmp file com perms wider
- Sem TOCTOU window de chmod-after-rename (chmod nunca é chamado depois do rename)
- Short writes do POSIX cobertos por loop interno (`writeFileSync(fd, buf)`)
- Stale tmp de runs anteriores limpo antes da nova escrita

---

## 📚 Plugins detalhes

### qwen-review

**Stop-time review gate via API Qwen 3.7 Max.** No fim de cada turn do Claude, um hook `Stop` envia o `last_assistant_message` + `git diff HEAD` + conteúdo (pré-redactado) dos arquivos modificados pro Qwen. Se o Qwen responder `BLOCK: <razão>`, o hook devolve `{decision: "block"}` e o Claude continua o turn tentando corrigir, em vez de parar.

**Comandos:**

| Comando | Descrição |
|---|---|
| `/qwen-review:wizard` | Doc + instrução pra rodar o wizard interativo via `!` |
| `/qwen-review:setup [--enable\|--disable]` | Liga/desliga o gate no workspace atual + ping na API |
| `/qwen-review:status` | Mostra config + último review (JSON) |
| `/qwen-review:check [--diff-only]` | Roda review manual on-demand contra `git diff HEAD` |

**Env vars** (todas opcionais exceto `QWEN_API_KEY`):

| Var | Default | Notas |
|---|---|---|
| `QWEN_API_KEY` | — | **Obrigatório.** Sem ela, o gate auto-skip com aviso. |
| `QWEN_BASE_URL` | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` | OpenAI-compat endpoint. |
| `QWEN_MODEL` | `qwen3-max` | Override pra outros nomes (ex: `qwen/qwen3-max` no OpenRouter). |
| `QWEN_REVIEW_MODE` | `fast` | `fast` (1024 tok, 120s) ou `thinking` (8192 tok, 600s, `enable_thinking=true`). `deep` é alias backward-compat de `thinking`. |
| `QWEN_REVIEW_TIMEOUT_MS` | 120000 / 600000 | Override timeout HTTP. |
| `QWEN_REVIEW_MAX_TOKENS` | 1024 / 8192 | Override cap de saída do modelo. |
| `QWEN_REVIEW_MAX_FILES` | 5 | Quantos arquivos enviar em `CHANGED_FILES_CONTENT`. |
| `QWEN_REVIEW_REDACT_SECRETS` | `1` | `0` desliga redaction (**NÃO recomendado**). |
| `QWEN_REVIEW_EXCLUDE_GLOBS` | — | Globs extras pra skip de arquivos, separados por `:`. |
| `QWEN_REVIEW_DEBUG` | `0` | `1` grava `.qwen-review-debug.log` no workspace (prompt + resposta crua). |

**Segurança (pre-redaction de secrets, ativa por default):**

- **File-level skip:** `.env*`, `*.key`/`*.pem`/`*.crt`/`*.p12`, paths com `secret`/`credential`/`token`, binários, symlinks, hardlinks (qualquer file com `nlink > 1`) — todos viram placeholder `[file excluded: <razão>]` no prompt
- **Content-level regex:** AWS keys, OpenAI/Qwen keys (`sk-…`), GitHub tokens (`ghp_…`/`gho_…`/etc), JWTs, PEM private key blocks, Slack tokens → substituídos por `[REDACTED:<tipo>]`
- Aplicado também no `git diff` (não só nos arquivos), e nos diffs sintéticos de untracked
- Test count: **68/68** — incluindo regressões pra todos os leaks que o Codex stop-gate pegou durante o desenvolvimento

Detalhes completos em [`qwen-review/README.md`](./qwen-review/README.md) e [`qwen-review/docs/superpowers/specs/`](./qwen-review/docs/superpowers/specs/).

---

## 🛠 Adicionar um novo plugin nessa marketplace

1. Crie pasta no root do repo: `./<nome-do-plugin>/`
2. Dentro dela, layout padrão Claude Code: `.claude-plugin/plugin.json`, `hooks/`, `commands/`, `scripts/`, `prompts/`, etc.
3. Edite `.claude-plugin/marketplace.json` e adicione no array `plugins`:
   ```json
   {
     "name": "<nome>",
     "source": "./<nome>",
     "description": "...",
     "version": "0.1.0",
     "author": { "name": "Mario Junior" },
     "keywords": ["..."],
     "category": "workflow"
   }
   ```
4. Commit + push pro `main`
5. Usuários puxam via `/reload-plugins` e habilitam com `"<nome>@pir0c0pter0": true` no `enabledPlugins`

---

## 🆘 Troubleshooting

| Sintoma | Diagnóstico | Fix |
|---|---|---|
| `Plugin qwen-review not found in marketplace local` | Tentativa de marketplace local sem manifest correto | Use marketplace github (esse repo) em vez de local |
| `/plugin list` não mostra qwen-review depois de editar settings.json | Marketplaces nova só carrega no startup | Feche e abra Claude Code (não basta `/reload-plugins`) |
| `/qwen-review:status` mostra `envOk: false` | `QWEN_API_KEY` não setado | Rode o wizard ou edite `env.QWEN_API_KEY` em `~/.claude/settings.json` |
| Gate não dispara | `stopReviewGate: false` no state do workspace | `/qwen-review:setup --enable` no projeto |
| Reviews lentos demais (>30s) | Modo `thinking` ativo | Wizard → modo `fast`, ou `unset QWEN_REVIEW_MODE` |
| Block falso por estilo | Modelo opinou sobre estética | Edite o template em `prompts/stop-review.md` no plugin (CLAUDE.md como input ainda é v0.2) |
| API rejeita modelo | Modelo do default não disponível no seu provider | Wizard → escolha modelo correto (ex: `qwen-max-latest` em DashScope, ou `qwen/qwen3-max` em OpenRouter) |
| Erro de JSON inválido após editar settings.json manualmente | Settings.json corrompido | Restaure backup, ou apague o arquivo e rode wizard de novo |
| Stale `.qwen-tmp` no `~/.claude/` | Wizard crashou mid-write | Próxima execução do wizard limpa automaticamente |

---

## 📝 Licença

Por plugin, dentro de cada pasta. Default MIT salvo notado em contrário.
