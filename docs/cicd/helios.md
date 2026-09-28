# Helios ИТМО

Сайт публикуется в `~/public_html/ssg/` и доступен по адресу `https://se.ifmo.ru/~s506807/ssg/`.

## Деплой

Job `deploy-helios` запускается только из `main`: скачивает собранный сайт и отправляет его на сервер через `rsync` по SSH. Параметры подключения заданы в `env` job, пароль — в секрете `HELIOS_PASSWORD`:

```yaml
env:
  HELIOS_HOST: helios.cs.ifmo.ru
  HELIOS_PORT: 2222
  HELIOS_USER: s506807
  HELIOS_PATH: public_html/ssg/
```

```yaml
- env:
    SSHPASS: ${{ secrets.HELIOS_PASSWORD }}
  run: |
    sshpass -e rsync -az --delete \
      -e "ssh -p $HELIOS_PORT -o StrictHostKeyChecking=no" \
      site/ "$HELIOS_USER@$HELIOS_HOST:$HELIOS_PATH"
```

Job `deploy-helios` выполняется параллельно с `deploy-pages`, поэтому ошибка деплоя на Helios не блокирует публикацию на Pages, но видна в статусе запуска.
