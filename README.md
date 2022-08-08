# Домашнее задание к занятию "08.02 Работа с Playbook"

Данный плейбук предназначен для установки `Clickhouse`, `Vector` на хосты, указанные в `inventory` файле.

## group_vars

| Переменная  | Назначение  |
|:---|:---|
| `clickhouse_version` | версия `Clickhouse` |
| `clickhouse_packages` | `RPM` пакеты `Clickhouse`, которые необходимо скачать |
| `vector_version` | версия `Vector` |
| `vector_dir` | каталог для распаковки `RPM` пакетов `Vector` |
| `arch` | архитектура системы |

## Inventory файл

Группа "clickhouse" состоит из 1 хоста `clickhouse-01`

Группа "vector" состоит из 1 хоста `vector-01`

## Playbook

Playbook состоит из 2 `play`.

### Play "Install Clickhouse" применяется на группу хостов "Clickhouse" и предназначен для установки и запуска `Clickhouse`

Объявляем `handler` для запуска `clickhouse-server`.

```yaml
handlers:
    - name: Start clickhouse service
      become: true
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
```

| Имя таска | Описание |
|--------------|---------|
| `Get clickhouse distrib` | Скачивание `RPM` пакетов. Используется цикл с перменными `clickhouse_packages`. Так как не у всех пакетов есть `noarch` версии, используем перехват ошибки `rescue` |
| `Install clickhouse packages` | Установка `RPM` пакетов. Используем `disable_gpg_check: true` для отключения проверки GPG подписи пакетов. В `notify` указываем, что данный таск требует запуск handler `Start clickhouse service` |
| `Flush handlers` | Форсируем применение handler `Start clickhouse service`. Это необходимо для того, чтобы handler выполнился на текущем этапе, а не по завершению тасок. Если его не запустить сейчас, то сервис не будет запущен и следующий таск завершится с ошибкой |
| `Create database` | Создаем в `Clickhouse` БД с названием "logs". Также прописываем условия, при которых таск будет иметь состояние `failed` и `changed` |

### Play "Install Vector" применяется на группу хостов "Vector" и предназначен для установки и запуска `Vector`

Объявляем `handler` для запуска `vector`.

```yaml
  handlers:
    - name: Start Vector service
      become: true
      ansible.builtin.service:
        name: vector
        state: restarted
```

| Имя таска | Описание |
|--------------|---------|
| `Download Vector distrib` | Скачивание `RPM` пакетов в текущую директорию пользователя |
| `Install Vector packages` | Установка `RPM` пакетов. Используем `disable_gpg_check: true` для отключения проверки GPG подписи пакетов |
| `Copy Vector config` | Применяем шаблон конфига `vector`. Здесь мы задаем путь конфига. Владельцем назначаем текущего пользователя `ansible`. |

### Play "Install lighthouse" применяется на группу хостов "lighthouse" и предназначен для установки и запуска `lighthouse`

Объявляем `handler` для перезапуска `Nginx`.

```yaml
 handlers:
    - name: Nginx reload
      become: true
      ansible.builtin.service:
        name: nginx
        state: restarted
```
| Имя pretask | Описание |
|--------------|---------|
| `Lighthouse \| Install git` | Устанавливаем `git` |
| `Lighhouse \| Install nginx` | Устанавливаем `Nginx` |
| `Lighthouse \| Apply nginx config` | Применяем конфиг `Nginx`. |

| Имя таска | Описание |
|--------------|---------|
| `Lighthouse \| Clone repository` | Клонируем репозиторий `lighthouse` из ветки `master` |
| `Lighthouse \| Apply config` | Применяем конфиг `Nginx` для `lighthouse`. После этого перезапускаем `nginx` для применения изменений |


## Template

Шаблон "vector.yml.j2" используется для настройки конфига `vector`. В нем мы указываем, что конфиг файл находится в переменной "vector_config" и его надо преобразовать в `YAML`.

Шаблон "nginx.conf.j2" используется для первичной настройки `nginx`. Мы задаем пользователя для работы `nginx` и удаляем настройки root директории по умолчанию.