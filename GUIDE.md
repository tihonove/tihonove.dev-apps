# Как подключить проект к кластеру tihonove.dev

Инструкция для автора проекта и для агента, который готовит проект за него. Читается сверху вниз;
каждый шаг заканчивается проверкой.

## Что даёт платформа

- Kubernetes (k3s) из двух нод: **arm64** и **amd64** — образ должен уметь обе архитектуры.
- Свой namespace на проект, с квотой (см. «Ресурсы»).
- Любое имя `*.tihonove.dev` с HTTPS: DNS и wildcard-сертификат уже настроены, http → https редиректится.
- GitOps через Flux: кластер сам забирает манифесты из **вашего** репо и применяет их.
  Доступа к кластеру для деплоя не нужно.
- Секреты храните в своём репо зашифрованными (sops) — расшифровать их может только кластер
  (и владелец платформы).

## Как это устроено

```
ваш репо (публичный)                       этот репо (tihonove.dev-apps)
  ветка/папка с манифестами  ◄── следит ──   apps/<app>.yaml  ← регистрация (PR)
  (kustomize)                                     │
        │                                         ▼
        └──────────────► Flux ──► namespace <app> в кластере
образ: ghcr.io/<owner>/<app> (публичный, amd64+arm64)
```

Flux опрашивает ваш репо раз в минуту. Любой коммит в отслеживаемую ветку/папку — это деплой.
Откат — `git revert` этого коммита.

## Требования

- **Репо проекта публичный.** Приватные пока не поддерживаются.
- **Образ публичный** (GHCR или любой публичный registry), собран под `linux/amd64` и `linux/arm64`.
- Манифесты — обычный kustomize (папка с `kustomization.yaml`).
- В манифестах **только объекты своего namespace**: Deployment, StatefulSet, Service, Ingress, ConfigMap,
  Secret, PVC, CronJob, Job, Role/RoleBinding и т. п. Namespace, ClusterRole, CRD и прочее кластерное
  Flux применить не сможет (`Forbidden`) — весь коммит не применится.
- `metadata.namespace` писать не нужно: Flux проставит namespace проекта всем объектам сам.

## Шаг 0. Имя проекта

Одно имя на всё: namespace, имя файла регистрации, поддомен по умолчанию.
Только `a-z`, `0-9`, `-`, начинается с буквы, до 40 символов. Проверьте, что `apps/<app>.yaml`
ещё не занят.

Дальше в примерах `APP` — это имя, `OWNER` — владелец репо на GitHub.

## Шаг 1. Образ

Мультиархитектурная сборка и публикация в GHCR (нужен `docker buildx`, логин: `docker login ghcr.io`):

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/OWNER/APP:<тег> --push .
```

Тег — любой неизменяемый: короткий SHA коммита, версия. **Не используйте `latest`**: по тегу в манифесте
видно, что запущено, и откат работает, только если старый тег не перезаписан.

Новый пакет в GHCR по умолчанию **приватный**. Сделайте его публичным:
`https://github.com/users/OWNER/packages/container/APP/settings` → Change visibility → Public.

**Проверка:** `docker manifest inspect ghcr.io/OWNER/APP:<тег>` без логина показывает две платформы.

## Шаг 2. Манифесты

Возьмите заготовку [`templates/project/deploy/`](templates/project/deploy/) и положите в свой репо.
Где именно — решаете вы:

- **папка в основной ветке** (`main`, `./deploy`) — код и манифесты меняются одним коммитом;
- **отдельная ветка только с манифестами** (`deploy`, `./`) — в основной ветке нет коммитов деплоя.

Это указывается в регистрации (`BRANCH`, `PATH_IN_REPO`), платформе всё равно.

Заготовка — веб-сервис на порту 8080: Deployment `web`, Service `web` (порт 80), Ingress на
`APP.tihonove.dev`. Подставьте `OWNER`, `APP`, `TAG`. Тег образа задаётся в `kustomization.yaml`
(`images:`), а не в Deployment — так его меняет одна команда:

```bash
cd deploy && kustomize edit set image app=ghcr.io/OWNER/APP:<новый тег>
```

### Ingress: адреса

- Любые имена `*.tihonove.dev`, сколько угодно: несколько `rules` в одном Ingress или несколько Ingress.
- В `tls.hosts` перечислите те же имена, **без `secretName`** — сертификат подставится сам.
- Вложенные имена (`a.b.tihonove.dev`) и свои домены пока не поддерживаются.
- Имена не резервируются: два проекта могут заявить одно. Не берите чужие.

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

Квота на весь namespace: 20 подов, `requests` 1 CPU / 1 GiB, `limits` памяти 2 GiB,
5 PVC / 20 GiB. Контейнер без `resources` получит по умолчанию `requests` 50m / 64Mi и лимит памяти 256Mi.
Вышли за квоту — лишние поды просто не создадутся (`exceeded quota` в событиях).

Указывайте `resources` явно и ставьте `readinessProbe`: Flux считает деплой успешным, только когда поды
готовы (ждёт до 3 минут).

### Диски

PVC со `storageClassName: local-path` — данные лежат на диске ноды, под привязывается к этой ноде.
Бэкапов платформа не делает. При удалении проекта (или PVC из манифестов) **данные стираются**.

**Проверка:** `kubectl kustomize deploy/` (или `kustomize build deploy/`) собирается без ошибок, в выводе
нет `namespace:` с чужими значениями и нет кластерных объектов.

## Шаг 3. Секреты (если нужны)

Секреты лежат в вашем репо **зашифрованными** — репо публичный. Шифруем sops ключами из `keys/`:
age-ключ кластера (расшифровывает кластер) и GPG владельца платформы (чтобы он мог помочь).

1. Положите [`templates/project/.sops.yaml`](templates/project/.sops.yaml) в **корень** своего репо.
2. Импортируйте GPG-ключ владельца (один раз): `gpg --import keys/owner.asc` (файл из этого репо).
3. Создайте секрет сразу зашифрованным — открытым текстом он не должен попасть даже в рабочую копию:

   ```bash
   kubectl create secret generic app-secrets \
     --from-literal=DATABASE_URL='...' --dry-run=client -o yaml \
   | sops encrypt --filename-override deploy/secrets.enc.yaml \
       --input-type yaml --output-type yaml /dev/stdin > deploy/secrets.enc.yaml
   ```
   Или `sops edit deploy/secrets.enc.yaml` и впишите Secret руками.
4. Добавьте `secrets.enc.yaml` в `resources` своего `kustomization.yaml` и подключите в контейнер:
   ```yaml
   envFrom:
     - secretRef:
         name: app-secrets
   ```

Имя файла обязано заканчиваться на `.enc.yaml` (по нему срабатывает правило sops).
Расшифровать файл сами вы не сможете — это нормально: чтобы поменять секрет, создайте файл заново.

**Проверка:** в `secrets.enc.yaml` значения вида `ENC[AES256_GCM,...]`, внизу блок `sops:` с `age` и `pgp`.

## Шаг 4. Регистрация (PR в этот репо)

```bash
sed -e 's/APP/<app>/g' \
    -e 's#REPO_URL#https://github.com/OWNER/<repo>#' \
    -e 's/BRANCH/main/' \
    -e 's#PATH_IN_REPO#./deploy#' \
    templates/app.yaml > apps/<app>.yaml
```

Ничего в файле больше не меняйте (кроме `BRANCH`/`PATH_IN_REPO` под вашу раскладку) — PR с другими
изменениями не примут. Откройте PR с одним файлом `apps/<app>.yaml`.

Что в регистрации: namespace с квотой и значениями по умолчанию, учётка, от имени которой Flux применяет
ваши манифесты (права только в вашем namespace), источник (`GitRepository`) и применение (`Kustomization`).

## После merge

В течение пары минут:
- создаётся namespace `<app>`;
- Flux забирает ваш репо и применяет манифесты;
- `https://<app>.tihonove.dev` начинает отвечать.

Дальше этот репо больше не участвует: деплой = коммит в вашу ветку.

Если не заработало — без доступа к кластеру причину видит владелец платформы. Для самостоятельной
отладки можно попросить kubeconfig на свой namespace.

## С kubeconfig (по запросу)

Права: всё внутри своего namespace, ничего за его пределами.

```bash
kubectl get pods                          # свои поды
kubectl logs deploy/web
kubectl get events --sort-by=.lastTimestamp
kubectl rollout restart deploy/web
```

Ручные правки манифестов Flux вернёт из git — меняйте приложение через свой репо.

## Обновление и откат

- **Обновить:** новый образ → `kustomize edit set image ...` → коммит → push.
- **Откатить:** `git revert` коммита с тегом → push. Старый образ уже в registry, ничего не пересобирается.
- **Сломанные манифесты** (не собираются, кластерный объект): Flux не применит **ничего** из коммита,
  продолжит работать предыдущая версия. Исправьте и запушьте снова.
- **Новые поды не поднимаются:** старые продолжают работать, пока новые не станут готовы.

## Удаление проекта

PR, удаляющий `apps/<app>.yaml`. Flux удалит namespace целиком — со всеми объектами и **данными PVC**.

## CI

Сборку образа и обновление тега можно автоматизировать в GitHub Actions — раздел будет дополнен.
