# pir0c0pter0/claude-plugins

Marketplace pessoal de plugins [Claude Code](https://claude.ai/code) mantidos por [@pir0c0pter0](https://github.com/pir0c0pter0).

> **Status:** plugins prontos para uso. Cada um é documentado individualmente dentro da sua pasta.

## 📦 Plugins disponíveis

| Plugin | Versão | Descrição |
|---|---|---|
| [`qwen-review`](./qwen-review) | 0.1.0 | Stop-time review gate via API Qwen 3.7 Max — bloqueia stop quando o Qwen identifica problema no turn anterior. |

---

## 🚀 Instalação (4 fases)

Cada fase tem uma verificação simples antes de avançar pra próxima — evita o problema de "tudo parece quebrado porque eu pulei um passo".

---

### Fase 1 — Registrar marketplace + instalar o plugin

Três rotas equivalentes. Escolha **uma**:

**Rota A — slash commands** (mais simples):

```
/plugins marketplace add pir0c0pter0/claude-plugins
/plugins install qwen-review@pir0c0pter0
```

`/plugins install` **já ativa** — hooks rodam e slash commands ficam disponíveis imediatamente. Não precisa de `/plugins enable` separado (esse só serve pra reativar após `/plugins disable`).

**Rota B — `~/.claude/settings.json` user-level manual:**

```json
{
  "extraKnownMarketplaces": {
    "pir0c0pter0": {
      "source": { "source": "github", "repo": "pir0c0pter0/claude-plugins" }
    }
  },
  "enabledPlugins": {
    "qwen-review@pir0c0pter0": true
  }
}
```

> ⚠️ Marketplace nova só carrega no **startup** — feche e abra o Claude Code da primeira vez. Atualizações subsequentes dos plugins puxam via `/reload-plugins`.

**Rota C — `<project>/.claude/settings.json` committed (team-scope):**

Mesmo JSON da Rota B, mas no `.claude/` do projeto (não home). Commitado no repo, todo dev pega a config ao clonar.

Precedência de settings (alto → baixo, override completo sem merge):

1. Managed (admin/MDM) — não override-ável
2. `.claude/settings.local.json` (gitignored, override pessoal)
3. `.claude/settings.json` (committed, team config)
4. `~/.claude/settings.json` (user-level)

> ⚠️ **Nunca** ponha `env.QWEN_API_KEY` em settings project-level — segredo no repo. A chave fica sempre em `~/.claude/settings.json` per-user (o wizard escreve lá por isso).

**Verificação da Fase 1:**

```
/plugins list
```

Deve listar `qwen-review@pir0c0pter0`. Nesse ponto o plugin tá instalado e ativo, mas ainda **sem chave** — então `/qwen-review:status` vai mostrar `envOk: false`. Isso é esperado, próxima fase resolve.

Se `/plugins list` não mostra o plugin, rode `/doctor` e revise o passo de install.

---

### Fase 2 — Configurar a API com o wizard

**Forma mais fácil** — slash command que imprime o comando pronto pra copiar:

```
/qwen-review:wizard
```

Cole a linha que ele te dá (já com `!` no início e o path resolvido) e o wizard abre. Detalhes em [seção do wizard abaixo](#-wizard-de-configuração) (incluindo alias permanente).

**Verificação da Fase 2:**

```
/qwen-review:status
```

Agora deve mostrar `envOk: true` + `apiKey: "sk-•••XXX"` (mascarada) + os outros valores. Se ainda mostra `envOk: false`, reinicie o Claude Code (env só recarrega no startup).

`reviewGateEnabled` continua `false` — fase 3.

---

### Fase 3 — Habilitar o gate no workspace

Entre no diretório do projeto onde quer review automático e rode:

```
/qwen-review:setup --enable
```

O gate é por workspace — state isolado por SHA-256 do realpath. Pode habilitar em qualquer número de projetos sem interferência.

**Verificação da Fase 3:**

```
/qwen-review:status
```

Agora deve mostrar `reviewGateEnabled: true` além do que já tinha.

---

### Fase 4 — Smoke test end-to-end

> ⚠️ **Antes** de fazer qualquer coisa nessa fase, rode `/qwen-review:status` e **anote o `lastReview.ts` atual** (pode estar `null` se nunca rodou, ou pode ter um timestamp antigo de outro workspace/dia). Só assim você consegue provar que o smoke test foi de fato a chamada que populou o estado.

Agora faça uma edit qualquer (até trivial, tipo um comentário) num arquivo do projeto e termine o turn no Claude Code. O hook `Stop` dispara, manda pro Qwen, e:

- Qwen responde `ALLOW:` → stop normal (silencioso)
- Qwen responde `BLOCK:` → Claude vê `{decision:"block"}` no transcript e continua o turn

Confirme com:

```
/qwen-review:status
```

Checagens (todas têm que valer):

1. `lastReview` é não-`null`
2. `lastReview.ts` **mudou** vs. o valor que você anotou antes (ou virou um timestamp se era `null`)
3. `lastReview.ts` é dos últimos minutos (não algo do dia anterior)
4. `lastReview.model` bate com o `QWEN_MODEL` que você configurou
5. `lastReview.latencyMs` e `promptTokens`/`completionTokens` fazem sentido (não-zero)

Se `lastReview.ts` continua igual ao valor anterior, o hook **não rodou nesse turn** — possíveis causas:
- Gate não está habilitado nesse workspace (`/qwen-review:setup --enable`)
- A edit foi exatamente vazia (atalho de `last_assistant + diff vazios` faz exit silent)
- Outro hook `Stop` falhou antes do qwen-review (`/doctor` mostra)
- Plugin não carregou direito (`/plugins list`)

Se algo deu errado em qualquer fase, `/doctor` é o diagnóstico padrão.

---

## 🧙 Wizard de configuração

O wizard é a forma recomendada de configurar `QWEN_API_KEY`, base URL, modelo e modo. Roda interativamente, mostra summary, pede confirmação, e escreve atomicamente em `~/.claude/settings.json` preservando todos os outros campos.

### Como rodar — 3 opções

**A — slash command (recomendado, sem decorar path):**

```
/qwen-review:wizard
```

Imprime a linha exata pra você copiar com `!` no início. Cola e o wizard abre.

**B — alias permanente no shell** (uma vez só, depois é só `! qwen-wizard`):

```bash
echo 'alias qwen-wizard="node $(ls -d ~/.claude/plugins/cache/pir0c0pter0/claude-plugins/*/qwen-review)/scripts/qwen-review.mjs wizard"' >> ~/.bashrc
source ~/.bashrc
```

(`~/.zshrc` se for zsh.)

Depois, dentro do Claude Code:

```
! qwen-wizard
```

**C — invocação direta** (sem setup, path completo):

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
| `/plugins list` não mostra qwen-review depois de editar settings.json | Marketplaces nova só carrega no startup | Feche e abra Claude Code (não basta `/reload-plugins`) |
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
