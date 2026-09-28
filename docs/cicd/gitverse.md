# P1. CI/CD на GitVerse

[GitVerse](https://gitverse.ru) — отечественная платформа для хостинга кода с CI/CD, совместимым по синтаксису с GitHub Actions. Пайплайн публикует сайт на MkDocs — генераторе, выбранном по итогам T1.

Файл пайплайна: `.gitverse/workflows/pipeline.yaml`.

## Схема пайплайна

| Job | Зависит от | Что делает | Артефакт | Ветки |
|---|---|---|---|---|
| `lint` | — | `codespell` (орфография), `mkdocs build --strict` (ссылки, якоря, структура) | — | все |
| `build` | `lint` | установка зависимостей с кэшем и замером времени, сборка | `site` (1 день) | все |
| `deploy` | `build` | скачивание артефакта `site`, `rsync` по SSH на Helios | — | только `main` |

## События и поведение

| Событие | lint | build | deploy |
|---|---|---|---|
| `push` в `main` | ✅ | ✅ | ✅ |
| `push` в другую ветку | ✅ | ✅ | ❌ |
| `pull_request` | ✅ | ✅ | ❌ |
| `workflow_dispatch` из `main` | ✅ | ✅ | ✅ |
| `workflow_dispatch` из другой ветки | ✅ | ✅ | ❌ |

Деплой ограничен условием:

```yaml
if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
```

## Lint

- **Орфография** — `codespell docs README.md mkdocs.yml` находит типичные опечатки.
- **Ссылки** — `mkdocs build --strict` вместе с блоком `validation` в `mkdocs.yml` превращает в ошибку битые внутренние ссылки, несуществующие якоря и абсолютные пути.

## Кэширование зависимостей

Кэш pip хранится через `actions/cache`, ключ — хэш `requirements.txt`, поэтому кэш сбрасывается только при изменении версий пакетов:

```yaml
- uses: actions/cache@v4
  id: cache
  with:
    path: ~/.cache/pip
    key: pip-${{ hashFiles('requirements.txt') }}
- run: |
    start=$(date +%s)
    pip install -r requirements.txt
    echo "cache hit: ${{ steps.cache.outputs.cache-hit == 'true' }}, pip install took $(($(date +%s) - start))s"
```

| Запуск | `cache hit` | `pip install` |
|---|---|---|
| Первый (без кэша) | `false` | — с |
| Повторный (с кэшем) | `true` | — с |

## Секреты

Адрес, порт, логин и путь на Helios заданы в `env` job. Пароль хранится в хранилище платформы: **Настройки репозитория → Секреты → `HELIOS_PASSWORD`**. В репозитории его нет — в workflow только ссылка:

```yaml
- env:
    SSHPASS: ${{ secrets.HELIOS_PASSWORD }}
```

Маскирование подтверждается в логе job `deploy`: в заголовке шага значение переменной выводится как `SSHPASS: ***`.

## Воспроизводимость

- **Python** — точная версия `3.12.7` в `actions/setup-python`.
- **Пакеты** — все версии в `requirements.txt` зафиксированы через `==`.
- **Раннер** — каждый запуск стартует на чистой машине, состояние между запусками передаётся только через кэш и артефакты.

## Запуски

### Успешный

!!! note "Скриншот"
    Успешный запуск: все три job зелёные.

### Намеренно проваленный

В `docs/index.md` добавлена ссылка на несуществующую страницу `[сломано](missing.md)`. Job `lint` падает, `build` и `deploy` не запускаются:

```
INFO    -  Building documentation to directory: /workspace/site
WARNING -  Doc file 'index.md' contains a link 'missing.md', but the target is not found among documentation files.

Aborted with 1 warnings in strict mode!
```

Разбор: MkDocs нашёл ссылку на файл, которого нет в `docs/`. В обычном режиме это только предупреждение, и сайт опубликовался бы с битой ссылкой. Флаг `--strict` превращает предупреждение в ненулевой код выхода, job завершается с ошибкой, и зависимые job не стартуют — сломанный сайт не попадает на сервер.

!!! note "Скриншот"
    Проваленный запуск: `lint` красный, `build` и `deploy` пропущены.

## Сравнение с GitHub Actions

| Аспект | GitHub Actions | GitVerse |
|---|---|---|
| Файл | `.github/workflows/deploy.yml` | `.gitverse/workflows/pipeline.yaml` |
| Раннер | `ubuntu-latest` | `ubuntu-cloud-runner` |
| Кэш pip | `setup-python` с `cache: pip` | явный `actions/cache` по хэшу `requirements.txt` |
| Публикация | GitHub Pages + Helios | только Helios |
| Артефакт | `upload-pages-artifact` | `upload-artifact` |
| Секреты | Settings → Secrets → Actions | Настройки → Секреты |
| Ветки с деплоем | `main` | `main` |

Что потребовалось изменить относительно workflow из T3:

1. Перенести файл в `.gitverse/workflows/`.
2. Заменить метку раннера `ubuntu-latest` на `ubuntu-cloud-runner`.
3. Убрать `deploy-pages` и `upload-pages-artifact` — аналога GitHub Pages нет, сайт публикуется только на Helios через обычный артефакт.
4. Заменить встроенный кэш `setup-python` на явный `actions/cache`, чтобы видеть попадание в кэш и измерять время.
5. Запускать пайплайн на `push` во все ветки, а деплой ограничить условием `if`.
