# ClickHouse, Vector и LightHouse на Yandex Cloud

Playbook `site.yml` разворачивает три сервиса на Ubuntu 24.04 amd64:

- ClickHouse: официальный APT-репозиторий, сервер и клиент, база `logs`, таблица `events` с движком MergeTree.
- Vector 0.44.0: DEB-пакет, генератор демонстрационных JSON-логов (одно событие каждые 10 секунд), отправка поля `message` в ClickHouse.
- LightHouse: статика из [VKCOM/lighthouse](https://github.com/VKCOM/lighthouse), Nginx, HTTP-прокси `/clickhouse/` к базе. Обработчики перезапускают сервисы при изменении конфигурации.

## Inventory

| Группа | Адрес | Сервис |
| --- | --- | --- |
| clickhouse | 51.250.66.218 | ClickHouse, HTTP 8123 |
| vector | 84.201.157.164 | Vector |
| lighthouse | 51.250.14.222 | Nginx, HTTP 80 |

`inventory/prod.yml`: SSH-пользователь `ubuntu`, ключ `~/.ssh/id_ed25519`, sudo без пароля. ClickHouse принимает подключения пользователя default только с localhost и адресов Vector/LightHouse. Nginx проксирует запросы посетителей LightHouse от своего адреса; это учебное окружение с доступом к интерфейсу по HTTP.

## Запуск

На управляющей Linux-машине нужны Python 3, venv, Ansible и ansible-lint. На ВМ нужны Python 3, SSH, sudo и доступ в интернет для загрузки пакетов. Security groups должны разрешать SSH с управляющей машины, HTTP 80 для браузера, TCP 8123 от Vector и LightHouse к ClickHouse.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install ansible-core ansible-lint
ansible-lint site.yml
ansible-playbook -i inventory/prod.yml site.yml --check --diff
ansible-playbook -i inventory/prod.yml site.yml --diff
ansible-playbook -i inventory/prod.yml site.yml --diff
```
 В режиме `--check` установка из локального DEB, распаковка архива, конфигурация и запуск ещё не установленных сервисов, а также SQL пропускаются: они требуют файлов и программ, которые check mode не создаёт. `get_url` в этом режиме проверяет URL, а не загружает файл. Полная проверка выполняется обычным запуском.

## Параметры и теги

Параметры находятся в `group_vars/all.yml`, адреса и SSH — в inventory. Переопределение: `-e lighthouse_port=8080`.

| Параметр | По умолчанию | Назначение |
| --- | --- | --- |
| clickhouse_database | logs | База для событий Vector |
| vector_version | 0.44.0 | Версия DEB для amd64 |
| lighthouse_version | master | Ветка или commit архива LightHouse |
| lighthouse_root | /var/www/lighthouse | Каталог статики |
| lighthouse_port | 80 | Порт Nginx |

Теги `clickhouse`, `vector`, `lighthouse` соответствуют трём play. Пример отдельного запуска: `ansible-playbook site.yml --tags lighthouse --diff`. Для Vector сначала должен быть развёрнут ClickHouse с таблицей. Для обновления статики меняйте `lighthouse_version`: распаковка выполняется один раз для каждой версии. ClickHouse устанавливается из stable-репозитория, уже установленные пакеты автоматически не обновляются.

