# Архитектурное ревью GitOps-решения (Enterprise Readiness)

Полное архитектурное ревью текущего GitOps-монорепозитория (`argocd-workflow`) с точки зрения требований **зрелой Enterprise-компании** (несколько команд, десятки кластеров, строгий ИБ-комплаенс, сетевая изоляция, высокие требования к SLA и контролируемому управлению изменениями).

---

## 1. Executive Summary: Насколько решение жизнеспособно?

**Вердикт:** **Хороший современный фундамент (~7/10 для Mid-size компаний, ~5/10 для крупного Enterprise «как есть» — требует доработки механизмов защиты от сбоев и масштабирования).**

Недавний переход от антипаттерна `kustomize --enable-helm` + `LoadRestrictionsNone` к связке **`ApplicationSet` (Merge Generator) + Native Multi-Source (`spec.sources`)** — это стратегически верный шаг. Вы используете современные паттерны Argo CD без лишних обёрток.

Однако для взрослого Enterprise в текущей реализации заложено **несколько критических бомб замедленного действия** (особенно вокруг `sync-wave`, единого `targetRevision: HEAD` и авто-прунинга в `prod`), которые при первой же человеческой ошибке в PR могут положить все кластеры одновременно.

| Критерий | Оценка | Краткий комментарий |
| :--- | :---: | :--- |
| **Топология Control Plane (Pull per cluster)** | **9/10** | Отлично подходит для изолированных контуров (DMZ/PCI-DSS), нет единого SPOF и «божественного» кластера с доступами везде. |
| **DRY и управление конфигами (Merge + Multi-Source)** | **8/10** | Элегантно разделены `base` и `environments`, но не хватает 3-го уровня иерархии (`cluster`), если в `prod` больше одного кластера. |
| **Надёжность и Blast Radius (Change Management)** | **4/10** | Все среды смотрят в `HEAD`. Ошибка в `base/` или `bootstrap/` мгновенно и одновременно убивает `test`, `preprod` и `prod`. |
| **Порядок раскатки и жизненный цикл аддонов** | **5/10** | `sync-wave` на `Application` внутри `ApplicationSet` не работает так, как задумано. Нет защиты от случайного удаления (`prune: true`). |
| **Производительность Argo CD** | **7/10** | Отказ от `--enable-helm` сильно разгрузил `repo-server`, но нет HA-режима Argo CD и есть нюансы с `ServerSideApply` и кэшем. |
| **Enterprise Security & Governance** | **4/10** | Нет управления секретами (ESO/Vault), слишком широкие права в `AppProject`, прямые походы в публичный интернет (`docker.io`, `github.io`). |

---

## 2. Сильные стороны текущего решения (Что сделано круто)

1. **Изолированный Pull-режим (Shared-Nothing Argo CD):**
   Каждый кластер сам тянет состояние из Git (`https://kubernetes.default.svc`). В Enterprise это решает главную боль ИБ: не нужно пробивать файрволы из центрального менеджмент-кластера в `kube-apiserver` закрытого `prod`-контура.
2. **Нативный Multi-Source (`spec.sources` с `ref: values`):**
   В [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml#L42-L57) и [`argocd-values.yaml`](argocd-values.yaml#L18-L21) вы убрали `kustomize --enable-helm` и `--load-restrictor=LoadRestrictionsNone`. Это закрыло уязвимость чтения произвольных файлов на `repo-server` и включило нативное кэширование Helm/OCI-чартов в Argo CD.
3. **Каталог аддонов через `Merge Generator` (Opt-in / Opt-out):**
   Сочетание базового каталога [`clusters/base/addons/`](clusters/base/addons) и тонких оверрайдов в `clusters/environments/<env>/addons/` позволяет включать обязательные агенты (`node-exporter`) глобально (`opt-out`), а тяжёлые компоненты (`envoy-gateway`, `prometheus`) — точечно (`opt-in`).
4. **Аннотация `resourceTrackingMethod: "annotation"` и `ServerSideApply=true`:**
   В [`argocd-values.yaml`](argocd-values.yaml#L23) выбран трекинг через аннотации (не ломает селекторы лейблов в сторонних чартах), а `ServerSideApply=true` в [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml#L67) решает проблему `metadata.annotations: Too long: must have at most 262144 bytes` на жирных CRD (Prometheus/Gateway API).

---

## 3. Критические подводные камни и архитектурные баги

### 3.1. Иллюзия работы `sync-wave` в `ApplicationSet` (Гонка при бутстрапе кластера)
В [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml#L36) вы проставляете волну синхронизации на генерируемые `Application`:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: '{{.syncWave}}'
```
* **В чём проблема:** В Argo CD аннотация `sync-wave` упорядочивает ресурсы **только внутри синхронизации одного родительского `Application`**. Но здесь объекты `Application` создаются контроллером **`ApplicationSet`** (`prod-cluster-addons`), а не напрямую через App-of-Apps!
* **Что произойдёт на практике:** `ApplicationSet` одновременно создаст все `Application` в кластере, и `argocd-application-controller` начнёт деплоить `envoy-gateway` (wave `-4`), `ingress-nginx` (wave `-3`) и `prometheus` (wave `0`) **параллельно**, игнорируя `sync-wave` на самих `Application`.
* **Как лечится в Argo CD:** Для пошаговой раскатки внутри `ApplicationSet` необходимо использовать `spec.strategy.type: RollingSync` с привязкой к лейблам (например, `wave: "{{.syncWave}}"` в `metadata.labels`), где каждый шаг ждёт статуса `Healthy` от предыдущего.

### 3.2. Огромный Blast Radius: единый `HEAD` + `prune: true` во всех средах
В [`root-bootstrap.yaml`](root-bootstrap.yaml#L12) и [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml#L19-L25) все контуры (`test`, `preprod`, `prod`) смотрят в одну ветку `HEAD` с включённым `automated: { prune: true, selfHeal: true }`.
* **Сценарий аварии №1 (Правка `base`):** Вы описали в README безопасный промоушен `chart.version` через `preprod`. Но если инженер меняет общую настройку в [`clusters/base/addons/ingress-nginx/common-values.yaml`](clusters/base/addons/ingress-nginx/common-values.yaml) или [`bootstrap/base/applicationsets/addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml), в момент мержа PR в `master` это изменение **одновременно раскатывается в `test`, `preprod` и `prod`** без всякого soak-time на тестовых контурах.
* **Сценарий аварии №2 (Массовое удаление через `ApplicationSet`):** В [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml) не задан `spec.syncPolicy.preserveResourcesOnDeletion: true`. Если кто-то случайно допустит ошибку в путях Git-генератора, переименует каталог или сломает селектор `enabled: "true"`, `ApplicationSet` мгновенно удалит `Application`-ы, а каскадный `prune: true` **снесёт `ingress-nginx`, `envoy-gateway` и `prometheus` в Production**.

### 3.3. Опасность `ignoreMissingValueFiles: true`
В [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml#L54-L57) указано:
```yaml
ignoreMissingValueFiles: true
valueFiles:
  - '$values/clusters/base/addons/{{.addon}}/common-values.yaml'
  - '$values/clusters/environments/ENV_PLACEHOLDER/addons/{{.addon}}/values.yaml'
```
* **Подводный камень:** Если инженер при подготовке `prod` опечатается в имени файла (например, создаст `value.yaml` или `values.yml` вместо `values.yaml`), Argo CD **не выдаст ошибку**, а молча проигнорирует prod-оверрайд и выкатит в `prod` базовый чарт с дефолтами (без HA-реплик, без лимитов и без `prod-dmz` лейблов).
* Кроме того, `ignoreMissingValueFiles: true` маскирует и отсутствие базового `common-values.yaml`, если имя папки не совпадёт с полем `addon`.

### 3.4. Хрупкие позиционные JSON-патчи в `bootstrap/environments/*/kustomization.yaml`
Во всех файлах [`bootstrap/environments/prod/kustomization.yaml`](bootstrap/environments/prod/kustomization.yaml#L11-L29), [`preprod/kustomization.yaml`](bootstrap/environments/preprod/kustomization.yaml#L11-L29) и [`test/kustomization.yaml`](bootstrap/environments/test/kustomization.yaml#L11-L29) подстановка имени среды сделана через жёсткие индексы массивов RFC 6902:
* `/spec/generators/0/merge/generators/1/git/files/0/path`
* `/spec/template/spec/sources/1/helm/valueFiles/1`

Стоит вам поменять порядок источников в `spec.sources` или добавить ещё один базовый слой в `valueFiles` в [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml), как все патч-индексы тихо поедут и начнут перезаписывать не те поля.
> **Как лучше:** Использовать Matrix/Merge с **Cluster Generator** по локальному кластеру (`in-cluster`), на секрет которого навешан лейбл `environment: prod`. Тогда папка `bootstrap/environments/*` со всеми `kustomization.yaml` и `ENV_PLACEHOLDER` не нужна вообще — `ApplicationSet` сам прочитает `{{.metadata.labels.environment}}` из кластера!

### 3.5. `RespectIgnoreDifferences=true` без возможности задать `ignoreDifferences` per-addon
В [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml#L68) включён `RespectIgnoreDifferences=true`, но в самом шаблоне `ApplicationSet` (и в [`schemas/addon.schema.json`](schemas/addon.schema.json)) нет поля `ignoreDifferences`.
* В Enterprise системные чарты (`ingress-nginx`, `cert-manager`, `kyverno`, `envoy-gateway`, операторы с `ValidatingWebhookConfiguration` / `MutatingWebhookConfiguration` и `caBundle`) в сочетании с `ServerSideApply=true` и `selfHeal: true` часто входят в бесконечный цикл синхронизации (OutOfSync loop), воюя с контроллерами в кластере за мутируемые поля. Без механизма передачи `ignoreDifferences` для конкретного аддона вам придётся хардкодить исключения всех аддонов в общий шаблон `addons-appset.yaml`.

---

## 4. Архитектурные ограничения при масштабировании (Enterprise Scale)

### 4.1. Проблема «Окружение ≠ Кластер» (Multi-Cluster Topology)
Сейчас иерархия двухуровневая: `base` -> `environments/{test,preprod,prod}`.
* В крупной компании почти никогда не бывает **одного** кластера `prod`. Обычно это `prod-eu-1`, `prod-eu-2`, `prod-pci`, `prod-dmz`, `stage-1`, `stage-2` и т.д.
* В вашей схеме при появлении второго `prod`-кластера (у которого такие же HA-настройки и retention, как у первого `prod`, но другие IP-пулы MetalLB/Cilium, другие доменные имена или размерности нод) вам придётся копировать всю папку `environments/prod` в `environments/prod-2`, теряя DRY.
* **Для Enterprise нужна 3-слойная цепочка:**
  1. `clusters/base/addons/<addon>/common-values.yaml` (глобальные дефолты)
  2. `clusters/tiers/<tier>/addons/<addon>/values.yaml` (настройки уровня `prod` / `non-prod` — реплики, retention, лимиты)
  3. `clusters/instances/<cluster-name>/addons/<addon>/values.yaml` (специфика конкретного кластера: IP, DNS, сертификаты).

### 4.2. Прямая зависимость `prod` от публичного интернета
В [`clusters/base/addons/`](clusters/base/addons) и [`tier0-addons.yaml`](bootstrap/base/projects/tier0-addons.yaml#L10-L15) указаны публичные апстримы:
* `https://kubernetes.github.io/ingress-nginx`
* `https://prometheus-community.github.io/helm-charts`
* `docker.io/envoyproxy`

В сетево-изолированных Enterprise-контурах (о которых заявлено в [`README.md`](README.md#L3)):
1. Из `prod` не будет прямого выхода в `github.io` и `docker.io`.
2. Если `repo-server` перезапустится во время сбоя GitHub Pages или упрётся в Rate Limit Docker Hub, реконсиляция инфраструктурных чартов в `prod` встанет колом.
3. Все Helm-чарты и образы в Enterprise должны зеркалироваться во внутренний корпоративный OCI-реестр (Harbor / Artifactory / Nexus).

### 4.3. Смешение `Tier 0` (Платформа) и `Tier 1` (Приложения) в одном репозитории
В [`bootstrap/base/projects/tier1-apps.yaml`](bootstrap/base/projects/tier1-apps.yaml) и [`clusters/base/templates/`](clusters/base/templates) заложена идея хранить прикладные сервисы (`backend`, `frontend`) в этом же монорепозитории:
* **Конфликт владения (Governance):** В Git нельзя разграничить права на запись по папкам нативно (только через `CODEOWNERS` и обязательный аппрув PR).
* **Шум от CI/CD:** Когда 50 продуктовых команд начнут коммитить обновления `image.tag` своих микросервисов по 200 раз в день в этот же репозиторий, история коммитов платформенной инфраструктуры превратится в свалку, а `argocd-repo-server` и `applicationset-controller` будут непрерывно перекачивать историю коммитов на каждый пуш разработчиков.
* **Незавершённость шаблонов `Tier 1`:** Чарты [`clusters/base/templates/backend`](clusters/base/templates/backend/templates/workload.yaml) и [`clusters/base/templates/frontend`](clusters/base/templates/frontend/templates/workload.yaml) сейчас на 100% дублируют друг друга, не имеют `ApplicationSet` для деплоя и не содержат обязательных для Enterprise сущностей (`securityContext`, `livenessProbe`/`readinessProbe`, `PodDisruptionBudget`, `HPA`, `ServiceAccount`, `NetworkPolicy`).

---

## 5. Производительность Argo CD: Нюансы и узкие места

1. **Ограничения `argocd-values.yaml` (Single Point of Failure внутри кластера):**
   * В [`argocd-values.yaml`](argocd-values.yaml#L35-L60) не включён HA-режим (`redis-ha`, ≥2 реплик `repoServer` и `server`).
   * Лимит памяти `controller.resources.limits.memory: 1536Mi` ([строка 39](argocd-values.yaml#L39)) на реальном Enterprise-кластере с 50+ CRD (Gateway API, Prometheus Operator, Cilium, Cert-Manager, Istio) и сотнями ресурсов при включённом `ServerSideApply=true` быстро приведёт к **OOMKill** `argocd-application-controller` (контроллер держит граф живых объектов кластера в оперативной памяти). Для prod-кластера обычно требуется от `4Gi` до `8Gi` RAM на контроллер.
2. **Особенности `manifest-generate-paths` с Multi-Source (`spec.sources`):**
   * В [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml#L35) используется `argocd.argoproj.io/manifest-generate-paths: '/clusters/base/addons/{{.addon}};/clusters/environments/ENV_PLACEHOLDER/addons/{{.addon}}'`.
   * Учтите, что `manifest-generate-paths` оптимизирует только обработку Git-вебхуков самим `application-controller`, но **не влияет на `applicationset-controller`**, который опрашивает Git-генератор (`clusters/base/addons/*/addon.yaml`) по своему циклу.
3. **Защита от удаления `ApplicationSet` при временной недоступности Git:**
   * Если корпоративный GitLab/GitHub моргнёт (502/504) в момент, когда `applicationset-controller` опрашивает файлы `clusters/base/addons/*/addon.yaml`, важно убедиться, что `ApplicationSet` не интерпретирует пустой ответ как удаление приложений (решается флагом `preserveResourcesOnDeletion: true`).

---

## 6. Безопасность и ИБ-комплаенс (Security Review)

1. **Полное отсутствие стратегии управления секретами (Secret Management):**
   * В репозитории никак не решён вопрос доставки секретов (OIDC client secret для Argo CD, TLS-сертификаты, пароли к внешним хранилищам, токены приватных репозиториев).
   * Для Enterprise стандартом де-факто с Multi-Source в Argo CD является **External Secrets Operator (ESO)** в связке с **HashiCorp Vault / Cloud KMS**, где ESO ставится самым первым аддоном в `syncWave: "-5"`, а в Git лежат только декларативные манифесты `ExternalSecret` / `SecretStore`.
2. **Эскалация привилегий через `AppProject`:**
   * В [`root-bootstrap.yaml`](root-bootstrap.yaml#L9) корневое приложение работает в проекте `default`.
   * В [`tier0-addons.yaml`](bootstrap/base/projects/tier0-addons.yaml#L16-L24) разрешены `namespace: "*"` и `clusterResourceWhitelist: "*"`. Это позволяет любому аддону модифицировать неймспейс `argocd` (например, перезаписать `argocd-cm` или `argocd-rbac-cm`). Рекомендуется явно исключить `namespace: "!argocd"` из `destinations` проекта `tier0-addons`.
   * В [`tier1-apps.yaml`](bootstrap/base/projects/tier1-apps.yaml#L17-L19) стоит `namespaceResourceWhitelist: [{group: "*", kind: "*"}]` на маску `app-*`. Это позволяет разработчикам одного сервиса удалить/изменить `ResourceQuota`, `LimitRange`, `NetworkPolicy` или `RoleBinding` в своём (или соседнем `app-*`) неймспейсе. В Enterprise `ResourceQuota`, `NetworkPolicy` и `RoleBinding` должны быть в `namespaceResourceBlacklist` для проекта `tier1-apps`.
3. **Валидация JSON-схемы [`schemas/addon.schema.json`](schemas/addon.schema.json):**
   * Сейчас одна и та же схема используется и для `clusters/base/addons/*/addon.yaml`, и для `clusters/environments/*/addons/*/addon.yaml`. Из-за этого в схеме обязательным является только поле `required: ["addon"]`.
   * Если кто-то добавит новый аддон в `base` и забудет указать `namespace`, `syncWave` или `chart.version`, схема не поймает ошибку в IDE — ошибка всплывёт только в рантайме в кластере (`missingkey=error`). Лучше разделить схему на `addon-base.schema.json` (где все поля `required`) и `addon-override.schema.json`.

---

## 7. Резюме и Roadmap: что нужно докрутить для уровня Enterprise

Если вы хотите защитить это решение на архитектурном комитете и безопасно эксплуатировать его в продакшене крупной компании, рекомендую следующий план эволюции (по приоритетам):

### Приоритет P0 (Критично для предотвращения аварий в Prod)
1. **Защитить `prod` от каскадного авто-удаления:**
   * Включить `spec.syncPolicy.preserveResourcesOnDeletion: true` в [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml).
   * Разделить ревизии Git по средам (например, `test` смотрит в `main`/`HEAD`, а `preprod` и `prod` смотрят в проверенные Git-теги/релизные ветки `release/prod`, либо промоушен изменений `base/` идёт через PR в ветку окружения / Kargo).
2. **Починить порядок раскатки аддонов (`RollingSync`):**
   * Заменить неработающий между `Application`-ами `sync-wave` на блок `spec.strategy.type: RollingSync` в [`addons-appset.yaml`](bootstrap/base/applicationsets/addons-appset.yaml) с шагами по лейблу волны (`-5`, `-4`, `-3`, `-1`, `0`).
3. **Настроить CI-гейты (Shift-Left Validation):**
   * Добавить пайплайн в CI, который проверяет:
     * валидность всех `addon.yaml` по JSON-схеме (`check-jsonschema`);
     * совпадение имени директории с полем `addon` внутри `addon.yaml`;
     * сухой рендер `kubectl kustomize` и `helm template` + `kubeconform` для всех сред.

### Приоритет P1 (Архитектура, ИБ и сетевая изоляция)
4. **Внедрить `external-secrets` (ESO) как нулевой аддон (`wave: -5`):**
   * Определить стандарт доставки секретов из корпоративного Vault/KMS во все кластеры.
5. **Убрать зависимость от публичного интернета:**
   * Перевести `repoURL` всех чартов на внутренний корпоративный OCI-реестр (Harbor/Nexus/Artifactory).
6. **Зажать права в `AppProject` ([`tier0-addons.yaml`](bootstrap/base/projects/tier0-addons.yaml) и [`tier1-apps.yaml`](bootstrap/base/projects/tier1-apps.yaml)):**
   * Запретить `tier0-addons` деплой в неймспейс `argocd` (`namespace: "!argocd"`).
   * Запретить `tier1-apps` управление `ResourceQuota`, `LimitRange`, `NetworkPolicy`, `Role`/`RoleBinding` через `namespaceResourceBlacklist`.
   * В [`argocd-values.yaml`](argocd-values.yaml) выключить `admin.enabled: "false"`, настроить SSO (OIDC) и ролевую модель в `configs.rbac`.

### Приоритет P2 (Масштабирование на 10+ кластеров и десятки команд)
7. **Отделить репозиторий приложений (`Tier 1`) от репозитория платформы (`Tier 0`):**
   * Оставить этот репозиторий чисто платформенным (`Platform GitOps Repo`), а деплой `tier1-apps` вынести в отдельный репозиторий (или per-team репозитории), подключённый к проекту `tier1-apps`.
8. **Перейти от 2-слойной (`base -> env`) к 3-слойной модели (`base -> tier -> cluster`) и убрать JSON-патчи:**
   * Использовать локальный `clusters` генератор (с лейблами `tier: prod`, `cluster: prod-eu-1` на секрете кластера в Argo CD), чтобы полностью избавиться от `bootstrap/environments/*/kustomization.yaml` и `ENV_PLACEHOLDER`.
