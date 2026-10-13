# Подключение проекта к кластеру tihonove.dev

## Платформа

- Kubernetes (k3s), две ноды: `arm64` и `amd64`. Образ должен поддерживать обе архитектуры.
- Отдельный namespace на проект с квотой (см. [Ресурсы](#ресурсы)).
- Любое имя `*.tihonove.dev` с HTTPS: DNS и wildcard-сертификат настроены, HTTP редиректится на HTTPS.
- GitOps через Flux: кластер сам забирает манифесты из репо проекта и применяет их. Доступ к кластеру
  для деплоя не нужен.
- Секреты хранятся в репо проекта зашифрованными (sops). Расшифровать их может только кластер
  и владелец платформы.

```
репо проекта (публичный)                   tihonove.dev-apps
  ветка/папка с манифестами  ◄── следит ──   apps/<app>.yaml  ← регистрация (PR)
  (kustomize)                                     │
        │                                         ▼
        └──────────────► Flux ──► namespace <app> в кластере
образ: ghcr.io/<owner>/<app> (публичный, amd64+arm64)
```

Flux опрашивает репо проекта раз в минуту. Коммит в отслеживаемую ветку/папку — это деплой,
`git revert` этого коммита — откат.

## Требования

- Репо проекта публичный. Приватные не поддерживаются.
- Образ публичный (GHCR или другой публичный registry), собран под `linux/amd64` и `linux/arm64`.
- Манифесты — kustomize: папка с `kustomization.yaml`.
- В манифестах только объекты своего namespace: Deployment, StatefulSet, Service, Ingress, ConfigMap,
  Secret, PVC, CronJob, Job, Role, RoleBinding. Namespace, ClusterRole, CRD и другие кластерные объекты
  Flux применить не сможет (`Forbidden`), и весь коммит не применится.
- `metadata.namespace` не указывать: Flux проставляет namespace проекта всем объектам.

## 1. Имя проекта

Одно имя на всё: namespace, файл регистрации, поддомен по умолчанию.

- Только `a-z`, `0-9`, `-`; начинается с буквы; до 40 символов.
- Файл `apps/<app>.yaml` ещё не занят.

Далее в примерах `APP` — имя проекта, `OWNER` — владелец репо на GitHub.

## 2. Образ

Сборка под обе архитектуры и публикация в GHCR (нужен `docker buildx` и `docker login ghcr.io`):

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/OWNER/APP:<тег> --push .
```

- Тег неизменяемый: короткий SHA коммита или версия. `latest` не использовать: по тегу в манифесте
  видно, что запущено, и откат работает только пока старый тег не перезаписан.
- Новый пакет в GHCR по умолчанию приватный. Сделать публичным:
  `https://github.com/users/OWNER/packages/container/APP/settings` → Change visibility → Public.

**Проверка:** `docker manifest inspect ghcr.io/OWNER/APP:<тег>` без логина показывает обе платформы.

## 3. Манифесты

Скопировать [`templates/project/deploy/`](templates/project/deploy/) в репо проекта и подставить
`OWNER`, `APP`, `TAG`. Заготовка — веб-сервис на порту 8080: Deployment `web`, Service `web` (порт 80),
Ingress на `APP.tihonove.dev`.

Где хранить манифесты — на выбор, указывается при регистрации (`BRANCH`, `PATH_IN_REPO`):

| Вариант | `BRANCH` | `PATH_IN_REPO` | Особенность |
|---|---|---|---|
| папка в основной ветке | `main` | `./deploy` | код и манифесты меняются одним коммитом |
| отдельная ветка только с манифестами | `deploy` | `./` | в основной ветке нет коммитов деплоя |

Тег образа задаётся в `kustomization.yaml` (`images:`), а не в Deployment. Смена тега:

```bash
cd deploy && kustomize edit set image app=ghcr.io/OWNER/APP:<новый тег>
```

### Ingress

- Любые имена `*.tihonove.dev`, в любом количестве: несколько `rules` в одном Ingress или несколько Ingress.
- В `tls.hosts` перечислить те же имена без `secretName` — сертификат подставится сам.
- Вложенные имена (`a.b.tihonove.dev`) и свои домены не поддерживаются.
- Имена не резервируются: два проекта могут заявить одно. Чужие имена не занимать.

```yaml
spec:
  tls:
    - hosts: [APP.tihonove.dev, api-APP.tihonove.dev]
  rules:
    - host: APP.tihonove.dev
      http: { paths: [{ path: /, pathType: Prefix, backend: { service: { name: web, port: { number: 80 } } } }] }
    - host: api-APP.tihonove.dev
      http: { paths: [{ path: /, pathType: Prefix, backend: { service: { name: api, port: { number: 80 } } } }] }
```

### Ресурсы

Квота на namespace:

| Что | Лимит |
|---|---|
| поды | 20 |
| `requests` | 1 CPU, 1 GiB |
| `limits` памяти | 2 GiB |
| PVC | 5 шт., 20 GiB |

Контейнер без `resources` получает `requests` 50m / 64Mi и лимит памяти 256Mi.
При превышении квоты лишние поды не создаются (`exceeded quota` в событиях).

- `resources` указывать явно.
- `readinessProbe` обязателен: Flux считает деплой успешным только когда поды готовы (ждёт до 3 минут).

### Диски

PVC со `storageClassName: local-path`: данные лежат на диске ноды, под привязан к этой ноде.
Бэкапов нет. При удалении проекта или PVC из манифестов данные стираются.

**Проверка:** `kubectl kustomize deploy/` собирается без ошибок; в выводе нет `namespace:` с чужими
значениями и нет кластерных объектов.

## 4. Секреты

Нужно только если у проекта есть секреты. Они хранятся в репо проекта зашифрованными через sops
двумя ключами из [`keys/`](keys/): age-ключом кластера и GPG-ключом владельца платформы.

1. Положить [`templates/project/.sops.yaml`](templates/project/.sops.yaml) в корень репо проекта.
2. Импортировать GPG-ключ владельца (один раз): `gpg --import keys/owner.asc`.
3. Создать секрет сразу зашифрованным, чтобы открытый текст не попал в рабочую копию:

   ```bash
   kubectl create secret generic app-secrets \
     --from-literal=DATABASE_URL='...' --dry-run=client -o yaml \
   | sops encrypt --filename-override deploy/secrets.enc.yaml \
       --input-type yaml --output-type yaml /dev/stdin > deploy/secrets.enc.yaml
   ```

   Или `sops edit deploy/secrets.enc.yaml` и вписать Secret вручную.
4. Добавить `secrets.enc.yaml` в `resources` своего `kustomization.yaml` и подключить в контейнер:

   ```yaml
   envFrom:
     - secretRef:
         name: app-secrets
   ```

- Имя файла должно заканчиваться на `.enc.yaml` — по нему срабатывает правило sops.
- Расшифровать файл автор проекта не может. Чтобы изменить секрет, файл создаётся заново.

**Проверка:** в `secrets.enc.yaml` значения вида `ENC[AES256_GCM,...]`, внизу блок `sops:` с `age` и `pgp`.

## 5. Регистрация

PR в этот репо с одним файлом `apps/<app>.yaml`, созданным из шаблона:

```bash
sed -e 's/APP/<app>/g' \
    -e 's#REPO_URL#https://github.com/OWNER/<repo>#' \
    -e 's/BRANCH/main/' \
    -e 's#PATH_IN_REPO#./deploy#' \
    templates/app.yaml > apps/<app>.yaml
```

Кроме `BRANCH` и `PATH_IN_REPO` в файле ничего не менять. PR с другими изменениями не принимается.

В регистрации: namespace с квотой и значениями по умолчанию, сервисная учётка Flux с правами только
в этом namespace, источник (`GitRepository`) и применение (`Kustomization`).

## После merge

В течение пары минут:

- создаётся namespace `<app>`;
- Flux забирает репо проекта и применяет манифесты;
- `https://<app>.tihonove.dev` начинает отвечать.

Дальше этот репо не участвует: деплой — коммит в отслеживаемую ветку репо проекта.

## Эксплуатация

**Обновить:** собрать образ с новым тегом → `kustomize edit set image ...` → коммит → push.

**Откатить:** `git revert` коммита с тегом → push. Старый образ уже в registry.

**Сломанные манифесты** (не собираются, кластерный объект): Flux не применит ничего из коммита,
работает предыдущая версия. Исправить и запушить снова.

**Новые поды не поднимаются:** старые работают, пока новые не станут готовы.

**Отладка.** Без доступа к кластеру причину видит владелец платформы. Для самостоятельной отладки
можно запросить kubeconfig с правами только внутри своего namespace:

```bash
kubectl get pods
kubectl logs deploy/web
kubectl get events --sort-by=.lastTimestamp
kubectl rollout restart deploy/web
```

Ручные правки объектов Flux вернёт из git. Менять приложение — только через репо проекта.

**Удалить проект:** PR, удаляющий `apps/<app>.yaml`. Flux удалит namespace целиком, вместе
с данными PVC.
