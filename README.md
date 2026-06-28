# LibreSpeed в K8s

`kubectl apply -f .` поднимет MySQL (ns `db`) и LibreSpeed во frontend-режиме (ns `final`).

## Порядок / структура
- `00-namespaces.yaml` - namespaces `db` и `final`.
- `db/` - MySQL: secret, configmap, init-скрипт (telemetry_mysql.sql), 2 сервиса (mysql + mysql-headless), StatefulSet `mysql`.
- `final/` - LibreSpeed: registrysecret, servers.json cm, env cm, secret, deployment, service, ingress.

## Доступ
http://librespeed.84.201.168.40.nip.io/  (https не форсируется)

## Дополнительные объекты и аргументация
- **Namespaces** `db`/`final` - изоляция БД и приложения (требование).
- **Headless-сервис** `mysql-headless` - обязателен для StatefulSet (стабильные DNS-имена подов).
- **ClusterIP `mysql`** - точка подключения приложения к БД.
- **init-configmap `mysql-initdb`** - структура БД (`speedtest_users`) монтируется в
  `/docker-entrypoint-initdb.d/`; официальный mysql выполняет её при первом старте.
- **readiness/liveness probes** на обоих ворклоадах - корректный rollout и самовосстановление.
- **emptyDir для MySQL** - в кластере нет StorageClass; задание допускает запись в ФС контейнера.
  При наличии провижионера блок `volumes.data` меняется на `volumeClaimTemplates`.
- **envFrom + отдельные секреты** - несекретные параметры из `librespeed-env`,
  пароли (DB_PASSWORD, PASSWORD статистики) - из `librespeed-secret`.

## Заметки
- Пароль пользователя БД продублирован в `db/mysql-secret` и `final/librespeed-secret`
  (они в разных namespace - общий секрет невозможен).
- `EMAIL` обязателен при `TELEMETRY=true` (GDPR), иначе entrypoint ругается.
