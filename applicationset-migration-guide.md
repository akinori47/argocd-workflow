# Руководство по миграции на Argo CD ApplicationSets (Git Generator)

## Концептуальная архитектура GitOps-монорепозитория

Данная архитектура спроектирована по лучшим enterprise-практикам GitOps и базируется на двух ключевых концепциях, не зависящих от конкретного движка генерации (будь то App of Apps или ApplicationSet):

### 1. Концепция многослойного репозитория (Layered Repository)
Для обеспечения принципа DRY (Don't Repeat Yourself) код в репозитории разделен на два независимых слоя:
* **Базовый слой шаблонов (`base/`):** 
  Содержит глобальные константы, ссылки на Helm-репозитории и общие конфигурации по умолчанию (`common-values.yaml`). Он не привязан к конкретному железу или кластеру и служит единым источником правды для структуры приложений.
* **Слой окружений (`environments/`):** 
  Содержит оверлеи конкретных физических кластеров (`test`, `prod` и т.д.). Этот слой определяет:
  * **Топологию:** какие именно приложения разворачиваются на конкретном кластере (Opt-in принцип).
  * **Параметризацию:** индивидуальные настройки окружения (реплики, лимиты ресурсов) через `values.yaml`.
  * **Патчинг:** точечные инфраструктурные изменения манифестов (например, специфичные метки безопасности), которые не поддерживает Helm-чарт.

### 2. Концепция разделения жизненного цикла (Lifecycle Decoupling)
Мы разделяем управление **доставкой** и управление **контентом** на две разные сущности:
* **Доставка (Argo CD Application):** Декларативное описание того, *как* и *куда* доставить сервис (проект, неймспейс, репозиторий, ветка, политики синхронизации).
* **Контент (Kubernetes / Helm):** Описание того, *что* именно запускается (манифесты, шаблоны, ресурсы).

В схеме **App of Apps** это разделение делается через явные файлы `Application` в Git. В схеме **ApplicationSet** это делается через единый шаблон-генератор, считывающий локальные файлы настроек `config.json`. В обоих случаях разработчик может менять параметры самого сервиса в Git, не затрагивая настройки пайплайна доставки в Argo CD.

---

В данном документе описано, как перевести текущую архитектуру кластерного бутстрапа (Kustomize App of Apps) на автоматическую генерацию через **ApplicationSet**. Это позволит полностью избавиться от ручного описания манифестов приложений и JSON-патчей, перейдя к концепции саморегулирующегося GitOps-монорепозитория.

---

## 1. Целевая структура папок в репозитории

После перехода на ApplicationSet структура репозитория упрощается и приобретает следующий вид:

```plaintext
infrastructure-repo/
├── clusters/
│   ├── base/                                # ЗОНА ШАБЛОНОВ (Только значения)
│   │   └── ingress-nginx/
│   │       └── common-values.yaml
│   │
│   └── environments/                        # ЗОНА ОКРУЖЕНИЙ
│       ├── test/                            # Кластер TEST
│       │   ├── ingress-nginx/
│       │   │   ├── config.json              # [NEW] Метаданные приложения
│       │   │   ├── kustomization.yaml
│       │   │   └── values.yaml
│       │   │
│       │   └── prometheus/
│       │       ├── config.json              # [NEW] Метаданные приложения
│       │       └── kustomization.yaml
│       │
│       └── prod/                            # Кластер PROD
│           └── ingress-nginx/
│               ├── config.json              # [NEW] Метаданные приложения
│               ├── kustomization.yaml
│               ├── values.yaml
│               └── prod-patch.yaml
│
├── root-applicationset.yaml                 # [NEW] Единый генератор приложений
└── argocd-values.yaml                       # Конфигурация Argo CD
```

### Что удаляется:
* Папка `clusters/base/argocd-apps/` со всеми заготовками приложений (`ingress-nginx-app.yaml`, `prometheus-app.yaml`, `node-exporter-app.yaml`).
* Корневые файлы оверлеев `clusters/environments/test/kustomization.yaml` и `clusters/environments/prod/kustomization.yaml`.

---

## 2. Спецификации новых файлов

### А. Файл метаданных приложения (`config.json`)
В папке каждого компонента (например, `test/ingress-nginx/`) создается файл `config.json`. В нем описываются индивидуальные параметры безопасности и деплоя для конкретного сервиса. Это решает задачу разделения прав между командами.

**Пример для `test/ingress-nginx/config.json`:**
```json
{
  "namespace": "ingress-nginx",
  "project": "admin-infra"
}
```

**Пример для `test/prometheus/config.json` (доступный Команде А):**
```json
{
  "namespace": "monitoring",
  "project": "restricted-infra"
}
```

---

### Б. Шаблон ApplicationSet (`root-applicationset.yaml`)
В корне репозитория создается один файл, который заменяет родительский бутстрап. Он сканирует папки в поисках файлов `config.json` и автоматически генерирует приложения, подставляя параметры из JSON.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: test-cluster-addons
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/akinori47/argocd-workflow.git
        revision: HEAD
        # Сканируем репозиторий на наличие конфигурационных файлов
        files:
          - path: "clusters/environments/test/**/config.json"
  template:
    metadata:
      # Имя приложения формируется автоматически (например: test-ingress-nginx)
      name: 'test-{{path.basename}}'
    spec:
      # Проект назначается из config.json (разделение доступов команд)
      project: '{{config.project}}'
      source:
        repoURL: https://github.com/akinori47/argocd-workflow.git
        targetRevision: HEAD
        # Путь к сборщику Kustomize для конкретного приложения
        path: '{{path.directory}}'
      destination:
        server: https://kubernetes.default.svc
        # Целевой неймспейс берется из config.json
        namespace: '{{config.namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## 3. Инструкция по бесшовной миграции (Zero Downtime)

Чтобы перевести работающий кластер с текущей схемы на `ApplicationSet` без остановки сервисов, выполните следующие шаги:

### Шаг 1: Подготовка конфигураций в Git
1. В локальном репозитории создайте файлы `config.json` во всех папках компонентов тестовой среды (см. Секцию 2).
2. Закоммитьте и отправьте изменения в Git.

### Шаг 2: Деплой генератора ApplicationSet
1. Примените манифест `ApplicationSet` к кластеру:
   ```bash
   kubectl apply -f root-applicationset.yaml
   ```
2. Контроллер `ApplicationSet` обнаружит файлы конфигурации в Git и сгенерирует новые объекты `Application` в Kubernetes. Так как их имена и настройки совпадают с текущими, Argo CD просто свяжет их с существующими работающими ресурсами (произойдет бесшовное усыновление).

### Шаг 3: Удаление старого Kustomize-корня
1. Удалите старое родительское приложение `root-bootstrap` из Argo CD **в режиме без каскадного удаления** (это предотвратит удаление дочерних подов):
   ```bash
   kubectl delete application root-bootstrap -n argocd --cascade=false
   ```
2. Теперь за управление дочерними приложениями полностью отвечает `ApplicationSet`.

### Шаг 4: Очистка репозитория от старого кода
1. Удалите ненужную папку `clusters/base/argocd-apps/`.
2. Удалите файл `clusters/environments/test/kustomization.yaml`.
3. Закоммитьте и отправьте изменения в Git.

Миграция завершена! Ни один контейнер в кластере не был перезапущен в процессе.
