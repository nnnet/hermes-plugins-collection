# hermes-plugins-collection

Git-submodules сборник всех внешних плагинов для **Hermes Agent**
(`nousresearch/hermes-agent`).

Каждый плагин — **отдельный приватный репозиторий** в org `nnnet`
со своей историей и версионированием. Этот super-repo собирает их
в один корневой путь, который Hermes монтирует одним bind-mount'ом.

## Список плагинов

| Submodule path | Назначение | Repo |
|---|---|---|
| `aegis-attestation` | Tier-A детерминированный QA-attestation kanban deliverable с hybrid hook+cron | [hermes-plugin-aegis-attestation](https://github.com/nnnet/hermes-plugin-aegis-attestation) |
| `ai-gateway` | Vercel AI Gateway model-provider | [hermes-plugin-ai-gateway](https://github.com/nnnet/hermes-plugin-ai-gateway) |
| `anthropic-custom` | Anthropic Messages protocol с кастомным base_url (CLR Gateway по умолчанию) | [hermes-plugin-anthropic-custom](https://github.com/nnnet/hermes-plugin-anthropic-custom) |
| `assistant-prompt-overlay` | Patch системного промпта для роли Гермес | [hermes-plugin-assistant-prompt-overlay](https://github.com/nnnet/hermes-plugin-assistant-prompt-overlay) |
| `chief-tools` | Динамические тулзы для chief sub-agent lifecycle | [hermes-plugin-chief-tools](https://github.com/nnnet/hermes-plugin-chief-tools) |
| `claude-agent-sdk` | Claude через официальный Anthropic Agent SDK (хост-подписка) | [hermes-plugin-claude-agent-sdk](https://github.com/nnnet/hermes-plugin-claude-agent-sdk) |
| `claude-via-meridian` | Claude через хостовый Meridian-proxy | [hermes-plugin-claude-via-meridian](https://github.com/nnnet/hermes-plugin-claude-via-meridian) |
| `desire-to-goal-driver` | pre_llm_call плагин для desire-to-goal workflow | [hermes-plugin-desire-to-goal-driver](https://github.com/nnnet/hermes-plugin-desire-to-goal-driver) |
| `github-native-tools` | Тулзы GitHub repo (list/view/delete/create) с внутренней аутентификацией | [hermes-plugin-github-native-tools](https://github.com/nnnet/hermes-plugin-github-native-tools) |
| `hindsight-sanitize` | Очистка мультимодальных blob'ов из Hindsight retain | [hermes-plugin-hindsight-sanitize](https://github.com/nnnet/hermes-plugin-hindsight-sanitize) |
| `mc-tools` | Mission Control integration primitives | [hermes-plugin-mc-tools](https://github.com/nnnet/hermes-plugin-mc-tools) |
| `openrouter-custom` | OpenRouter live-filter + stable pseudo-model alias | [hermes-plugin-openrouter-custom](https://github.com/nnnet/hermes-plugin-openrouter-custom) |
| `workflow-engine` | Generic state-machine для многошаговых agent workflows | [hermes-plugin-workflow-engine](https://github.com/nnnet/hermes-plugin-workflow-engine) |
| `workflow-tools` | Backend-agnostic workflow orchestration tools | [hermes-plugin-workflow-tools](https://github.com/nnnet/hermes-plugin-workflow-tools) |

## Использование (Hermes integration)

### Первый клон с подтягиванием submodules

```bash
git clone --recurse-submodules \
  git@github.com:nnnet/hermes-plugins-collection.git
```

### Обновить все плагины до их последнего main

```bash
git submodule update --remote --merge
```

### Bind-mount в контейнер Hermes

В `infra/hermes/docker-compose.hermes-core.yml`:

```yaml
volumes:
  - ${EXTERNAL_PLUGINS_DIR}:/opt/hermes/external-plugins:ro
```

где `EXTERNAL_PLUGINS_DIR` указывает на корень склонированного
super-repo. Каждый submodule доступен по
`/opt/hermes/external-plugins/<plugin-name>/`.

Плагины регистрируются автоматически через
`config.yaml > plugins.enabled` — Hermes ищет их в discovery roots
(включая `/opt/hermes/external-plugins/`).

### Добавление нового плагина

```bash
git submodule add git@github.com:nnnet/hermes-plugin-<name>.git <name>
git commit -m "Add submodule: <name>"
git push
```

После этого добавить `<name>` в `config.yaml > plugins.enabled`
в `infra/hermes/.hermes/config.yaml`.

## Структура версионирования

- **Каждый плагин** версионируется в своём репо как обычный пакет
  (semver через git tags).
- **Super-repo** фиксирует конкретный коммит каждого submodule,
  что даёт reproducible deployment.
- Обновление плагина = коммит в plugin-repo + `git submodule update --remote`
  в super-repo + commit зафиксировавший новый submodule SHA.
