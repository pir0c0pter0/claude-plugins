# pir0c0pter0/claude-plugins

Marketplace pessoal de plugins [Claude Code](https://claude.ai/code) mantidos por [@pir0c0pter0](https://github.com/pir0c0pter0).

## Instalação

No `~/.claude/settings.json`:

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

Depois `/reload-plugins` (ou restart do Claude Code se for a primeira vez registrando a marketplace).

## Plugins disponíveis

| Plugin | Versão | Descrição |
|---|---|---|
| [`qwen-review`](./qwen-review) | 0.1.0 | Stop-time review gate via API Qwen 3.7 Max — bloqueia stop quando o Qwen identifica problema no turn anterior. |

## Adicionar um novo plugin

1. Crie a pasta `./<nome-do-plugin>/` com a estrutura padrão Claude Code (`.claude-plugin/plugin.json`, `hooks/`, `commands/`, etc.)
2. Adicione uma entry em `.claude-plugin/marketplace.json` apontando `"source": "./<nome-do-plugin>"`
3. Commit + push

Usuários da marketplace puxam com `/reload-plugins` e habilitam com `"<nome>@pir0c0pter0": true` em `enabledPlugins`.

## Licença

Por plugin, dentro de cada pasta.
