# GitHub Actions CI/CD

## Схема пайплайна

![CI/CD пайплайн](../img/pipeline.svg)

## Триггеры и поведение

| Событие | lint | build | deploy |
|---|---|---|---|
| `push` в `main` | ✅ | ✅ | ✅ |
| `pull_request` в `main` | ✅ | ✅ | ❌ |
| `workflow_dispatch` из `main` | ✅ | ✅ | ✅ |

## Как работает

1. `lint` проверяет сайт строгой сборкой `mkdocs build --strict`.
2. `build` устанавливает зависимости (кэш pip через `setup-python` с `cache: pip`), собирает сайт и загружает его артефактом `github-pages`.
3. `deploy-pages` публикует артефакт на GitHub Pages (Settings → Pages → Source = GitHub Actions).
4. `deploy-helios` скачивает тот же артефакт и отправляет его на Helios через `rsync`.

Деплой на Helios описан на странице [Helios](helios.md).

Workflow: [`.github/workflows/deploy.yml`](https://github.com/Spoon221/SSG/blob/main/.github/workflows/deploy.yml).
