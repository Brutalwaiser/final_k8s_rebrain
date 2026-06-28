# LibreSpeed в K8s

`kubectl apply -f . -R` поднимет MySQL (ns `db`) и LibreSpeed во frontend-режиме (ns `final`).

## Структура
- `00-namespaces.yaml` - namespaces `db` и `final`.
- `db/` - MySQL: secret, configmap, init-скрипт (telemetry_mysql.sql), 2 сервиса (mysql + mysql-headless), StatefulSet `mysql`.
- `final/` - LibreSpeed: registrysecret, servers.json cm, env cm, secret, deployment, service, ingress.

## Доступ
http://librespeed.84.201.168.40.nip.io/  (https не форсируется)

## Аргументация выбора

- **StatefulSet, а не Deployment для MySQL.** Для БД важны стабильная сетевая
  идентичность и привязка к своему storage. Deployment для одной реплики тоже
  взлетел бы, но StatefulSet даёт предсказуемое имя пода (`mysql-0`), упорядоченный
  старт и штатный путь к переходу на `volumeClaimTemplates`, если появится
  провижионер. Для stateful-нагрузки это семантически правильный контроллер.

- **emptyDir для данных MySQL, а не PVC.** В кластере нет StorageClass
  (`kubectl get sc` → пусто), динамический провижининг невозможен. Задание
  допускает запись в фс контейнера, поэтому выбран `emptyDir`. Минус - данные
  живут только пока жив под; для прода блок `volumes.data` заменяется на
  `volumeClaimTemplates` с реальным StorageClass, остальной манифест не меняется.

- **Образ `mysql:8.0` с зафиксированным тегом, а не `latest`.** Воспроизводимость:
  `latest` со временем уедет на новый мажор и может сломать совместимость
  (например, изменение auth-плагина). Фиксированный тег гарантирует одинаковое
  поведение при каждом `apply`.

- **Инициализация БД через init-configmap, а не вручную.** Структура
  `speedtest_users` (telemetry_mysql.sql) монтируется в
  `/docker-entrypoint-initdb.d/`. Официальный образ mysql выполняет оттуда все
  `*.sql` при первом старте на пустом dataDir - бд готова без ручных шагов и
  ребиадов образа. Альтернатива (ручной `mysql < script.sql` или кастомный образ)
  хуже воспроизводится.

- **readiness/liveness probes, хотя они не требовались.** Без readiness под
  librespeed помечается готовым раньше, чем поднимется MySQL, и крэшится при
  первом обращении к БД. Probes дают корректный порядок готовности и
  самовосстановление при зависании. Сознательно добавлены сверх минимума.

- **envFrom (configmap) + отдельные секреты.** Несекретные параметры (MODE,
  DB_HOSTNAME и т.п.) грузятся пачкой через `envFrom` из `librespeed-env` -
  компактно. Пароли (`DB_PASSWORD`, `PASSWORD` статистики) вынесены в Secret и
  прокидываются точечно через `secretKeyRef`, чтобы не светить креды в конфигмапе.

- **Хост на `nip.io`, а не свой DNS / правка hosts.** `*.84.201.168.40.nip.io`
  автоматически резолвится в IP балансировщика без настройки DNS-зоны и без правки
  `/etc/hosts` на каждой машине. Удобно для временного учебного окружения.

- **Монтирование servers.json через `subPath`.** Нужно положить ровно один файл
  в `/servers.json`. Без `subPath` configmap-volume смонтировался бы директорией
  и затёр бы содержимое целевого пути. `subPath` монтирует только нужный ключ как
  файл.

## Объекты, продиктованные требованиями (не выбор, а необходимость)
- **Namespaces** `db`/`final` - прямое требование задания (изоляция).
- **Headless-сервис** `mysql-headless` - обязателен для StatefulSet (стабильные
  DNS-имена подов вида `mysql-0.mysql-headless.db.svc`).
- **ClusterIP `mysql`** - точка подключения приложения к БД.
- **registrysecret** - pull образа из приватного registry rebrainme.

## Заметки
- Пароль пользователя базы продублирован в `db/mysql-secret` и
  `final/librespeed-secret` - они в разных namespace, общий Secret между ns
  невозможен. В проде это решается внешним хранилищем (Vault/ES).
- `EMAIL` обязателен при `TELEMETRY=true` (GDPR), иначе entrypoint падает.
- `REDACT_IP_ADDRESSES=true` → в телеметрии IP редактируется (`0.0.0.0`), при этом
  в веб-интерфейсе реальный IP клиента виден (заголовки X-Forwarded-For / X-Real-IP
  прокидываются ingress-ом).