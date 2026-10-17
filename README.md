# Argo CD Multi-Cluster Infrastructure Monorepo (DRY Merge Generator + Multi-Source)

Единый GitOps-монорепозиторий для управления системной инфраструктурой (`Tier 0 Addons`) и шаблонами приложений в нескольких сетево-изолированных контурах (`test`, `preprod`, `prod`), где каждый кластер использует собственный локальный инстанс Argo CD в режиме Pull.

---

## 1. Архитектура и принципы (Zero-Duplication DRY)

```text
+-------------------------------------------------------------------------+
|                 ЕДИНЫЙ GIT-МОНОРЕПОЗИТОРИЙ (argocd-workflow)            |
|                                                                         |
|   +-----------------------------------------------------------------+   |
|   | clusters/base/addons/*  (ОПИСАНО ОДИН РАЗ НА ВСЕ КЛАСТЕРЫ)      |   |
|   |  - addon.yaml (repoURL, chart, базовая версия, namespace, wave) |   |
|   |  - common-values.yaml (общие настройки Helm для всех сред)      |   |
|   +-----------------------------------------------------------------+   |
|              ^                            ^                       ^     |
|              | (Merge Generator)          | (Merge Generator)     |     |
|   +--------------------+       +--------------------+   +-------------+ |
|   | environments/test  |       |environments/preprod|   |environ./prod| |
|   | (только дельта!)   |       | (только дельта!)   |   |(только HA!) | |
|   +--------------------+       +--------------------+   +-------------+ |
+--------------|----------------------------|-----------------------|-----+
               | Pull                       | Pull                  | Pull
               v                            v                       v
   +-----------------------+    +-----------------------+   +-------------+
   | Кластер TEST          |    | Кластер PREPROD       |   | Кластер PROD|
   | [Свой локальный Argo] |    | [Свой локальный Argo] |   | [Свой Argo] |
   +-----------------------+    +-----------------------+   +-------------+
```

### Ключевые механизмы:
1. **Единый `ApplicationSet` с `Merge Generator` (`bootstrap/base/applicationsets/addons-appset.yaml`):**
   * Слой 1 читает базовые определения аддонов из `clusters/base/addons/*/addon.yaml` (имя чарта, `repoURL`, базовая проверенная версия, целевой `namespace`, `syncWave`).
   * Слой 2 накладывает поверх переопределения конкретного окружения из `clusters/environments/<env>/addons/*/addon.yaml` по ключу `addon`.
   * Селектор `enabled: "true"` отфильтровывает только те компоненты, которые должны работать в данном окружении.
2. **Нативный Multi-Source (`spec.sources`) без `kustomize --enable-helm`:**
   * Argo CD рендерит чарты нативно с полноценным кэшированием `repo-server` и накладывает `clusters/base/addons/<addon>/common-values.yaml` + `clusters/environments/<env>/addons/<addon>/values.yaml`.
   * Благодаря `ignoreMissingValueFiles: true`, если в окружении не нужны кастомные `values.yaml`, файл можно вообще не создавать.

---

## 2. Структура репозитория

```text
argocd-workflow/
├── argocd-values.yaml                          # Helm values для установки самого Argo CD
├── root-bootstrap.yaml                         # Точка входа (App of Apps для Control Plane)
├── schemas/
│   └── addon.schema.json                       # JSON-схема валидации addon.yaml
├── bootstrap/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── projects/
│   │   │   ├── tier0-addons.yaml               # AppProject для системных аддонов
│   │   │   └── tier1-apps.yaml                 # AppProject для прикладных сервисов
│   │   └── applicationsets/
│   │       └── addons-appset.yaml              # Merge Generator + Multi-Source для аддонов
│   └── environments/
│       ├── test/kustomization.yaml             # Привязка генератора к окружению test
│       ├── preprod/kustomization.yaml          # Привязка генератора к окружению preprod
│       └── prod/kustomization.yaml             # Привязка генератора к окружению prod
└── clusters/
    ├── base/
    │   ├── addons/                             # Базовый каталог системных аддонов (Tier 0)
    │   │   ├── envoy-gateway/{addon.yaml,common-values.yaml}
    │   │   ├── ingress-nginx/{addon.yaml,common-values.yaml}
    │   │   ├── prometheus/{addon.yaml,common-values.yaml}
    │   │   └── node-exporter/{addon.yaml,common-values.yaml}
    │   └── templates/                          # Базовые заготовки шаблонов для приложений
    │       ├── backend/
    │       └── frontend/
    └── environments/                           # Оверрайды конкретных окружений (только дельта)
        ├── test/addons/
        ├── preprod/addons/
        └── prod/addons/
```

---

## 3. Типовые операции

### А. Включение / выключение аддона в окружении
* **Opt-in (выключен в `base` по умолчанию, например `envoy-gateway`, `ingress-nginx`, `prometheus`):**
  Чтобы включить аддон в `prod`, создайте `clusters/environments/prod/addons/<addon>/addon.yaml`:
  ```yaml
  addon: envoy-gateway
  enabled: "true"
  ```
* **Opt-out (включён везде по умолчанию, например `node-exporter`):**
  В `clusters/base/addons/node-exporter/addon.yaml` стоит `enabled: "true"`, поэтому он ставится во все кластеры автоматически даже без создания папок в `environments/`. Чтобы выключить его в конкретном окружении, добавьте `addon.yaml` с `enabled: "false"`.

### Б. Обкатка новой версии чарта (`test` / `preprod` -> `prod`)
Чтобы протестировать новую версию чарта в `preprod`, не затрагивая `prod`, переопределите только поле `chart.version` в `clusters/environments/preprod/addons/ingress-nginx/addon.yaml`:
```yaml
addon: ingress-nginx
enabled: "true"
chart:
  version: "4.12.1"
```
После успешной проверки удалите блок `chart:` из `preprod` и обновите базовую версию в `clusters/base/addons/ingress-nginx/addon.yaml`.

### В. Добавление дополнительных Raw-манифестов (`ConfigMap`, `StorageClass`, `Gateway` и др.)
Дополнительные «голые» Kubernetes-манифесты кладутся прямо рядом с `values.yaml` и `addon.yaml` внутри папки аддона в нужном окружении (`clusters/environments/<env>/addons/<addon>/`) и подключаются через условный **Source 3 (Kustomize)** без создания пустых файлов-заглушек в остальных аддонах:
1. В `clusters/environments/<env>/addons/<addon>/addon.yaml` включите флаг `kustomize: "true"`:
   ```yaml
   addon: ingress-nginx
   enabled: "true"
   kustomize: "true"
   ```
2. Положите нужные манифесты (например, `custom-configmap.yaml`, `gateway.yaml`) в ту же папку и перечислите их в `kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization

   resources:
     - custom-configmap.yaml
   ```
   > Kustomize читает строго белый список из `resources:` и автоматически игнорирует лежащие рядом `addon.yaml` и `values.yaml`. Чтобы временно отключить отдельный манифест — закомментируйте его в `kustomization.yaml`; чтобы отключить все raw-манифесты аддона — поставьте `kustomize: "false"` (или уберите строку из оверлея).

---

## 4. Локальная проверка манифестов

```bash
# Проверка сборки Control Plane для каждого окружения
kubectl kustomize bootstrap/environments/test
kubectl kustomize bootstrap/environments/preprod
kubectl kustomize bootstrap/environments/prod

# Проверка рендера raw-манифестов (Source 3 Kustomize) в папке аддона
kubectl kustomize clusters/environments/test/addons/ingress-nginx

# Проверка шаблонов приложений
helm template demo-backend clusters/base/templates/backend -f clusters/base/templates/backend/common-values.yaml
helm template demo-frontend clusters/base/templates/frontend -f clusters/base/templates/frontend/common-values.yaml
```
