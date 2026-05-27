[АРТЕФАКТ]: Архитектурный план кластерного бутстрапа (ArgoCD + Kustomize)
Цель: Реализация декларативного слоя управления системной инфраструктурой (Cluster Addons) на базе локальных инстансов Argo CD с использованием паттерна Opt-in топологии и Common Values + Overrides (DRY).

1. Целевая структура папок в репозитории
Агенту необходимо создать исключительно инфраструктурную ветку clusters/ со следующей иерархией:

Plaintext
infrastructure-repo/
└── clusters/
    ├── base/                                # ЗОНА ШАБЛОНОВ (Глобальные блупринты)
    │   ├── argocd-apps/                     # Бланки Argo-приложений для Слой 1
    │   │   ├── ingress-nginx-app.yaml
    │   │   └── prometheus-app.yaml
    │   └── ingress-nginx/
    │       └── common-values.yaml           # Глобальные Helm-дефолты для Слой 2
    │
    └── environments/                        # ЗОНА ОКРУЖЕНИЙ (Локальные кластеры)
        ├── test/                            # Кластер TEST (Ограниченный набор)
        │   ├── kustomization.yaml           # Выбор компонентов (Топология)
        │   └── ingress-nginx/
        │       ├── kustomization.yaml       # Сборщик Helm для ТЕСТА
        │       └── values.yaml              # Специфичные values (Реплики=1)
        │
        └── prod/                            # Кластер PROD (Максимальный HA-набор)
            ├── kustomization.yaml           # Выбор компонентов (Топология)
            └── ingress-nginx/
                ├── kustomization.yaml       # Сборщик Helm для ПРОДА
                ├── values.yaml              # Специфичные values (Реплики=3)
                └── prod-patch.yaml          # Системный патч в обход Helm
2. Спецификации файлов (Манифесты для генерации)
Слой 1: Управление топологией кластеров (Выборочный деплой / Opt-in)
clusters/environments/test/kustomization.yaml
YAML
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# В тесте разворачиваем только Ingress и Prometheus
resources:
  - ../../base/argocd-apps/ingress-nginx-app.yaml
  - ../../base/argocd-apps/prometheus-app.yaml

patches:
  - target:
      kind: Application
      name: system-ingress-nginx
    patch: |-
      - op: replace
        path: /spec/source/path
        value: clusters/environments/test/ingress-nginx
clusters/environments/prod/kustomization.yaml
YAML
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# В проде разворачиваем полный стек системных аддонов
resources:
  - ../../base/argocd-apps/ingress-nginx-app.yaml
  - ../../base/argocd-apps/prometheus-app.yaml

patches:
  - target:
      kind: Application
      name: system-ingress-nginx
    patch: |-
      - op: replace
        path: /spec/source/path
        value: clusters/environments/prod/ingress-nginx
Слой 2: Конфигурационный слой аддонов (DRY: Helm + Kustomize Overlays)
clusters/base/ingress-nginx/common-values.yaml (Глобальные дефолты)
YAML
controller:
  config:
    log-format-escape-json: "true"
    log-format-upstream: '{"time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr", "status": "$status", "request": "$request"}'
  metrics:
    enabled: true
clusters/environments/prod/ingress-nginx/kustomization.yaml (Сборщик ПРОДА)
YAML
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.10.0
    releaseName: ingress-nginx
    namespace: ingress-nginx
    additionalValuesFiles:
      - ../../../base/ingress-nginx/common-values.yaml # Импорт общей базы
      - values.yaml                                    # Импорт локального оверрайда

patches:
  - path: prod-patch.yaml # Применение хирургического патча ИБ
clusters/environments/prod/ingress-nginx/values.yaml (Helm-параметры ПРОДА)
YAML
controller:
  replicaCount: 3
  resources:
    limits:
      cpu: "2"
      memory: 2Gi
    requests:
      cpu: "1"
      memory: 1Gi
clusters/environments/prod/ingress-nginx/prod-patch.yaml (Патч в обход Helm)
YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        corporate.internal/security-zone: "prod-dmz"
        monitored-by: "infra-agent"
Бланк ArgoCD Application (Шаблон для Слой 1)
clusters/base/argocd-apps/ingress-nginx-app.yaml
YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: system-ingress-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/infrastructure-repo.git
    targetRevision: HEAD
    path: clusters/base/ingress-nginx # Будет динамически переписано Kustomize-оверлеем
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
3. Чек-лист задач для выполнения Агентом Antigravity
[ ] Задача 1: Создать дерево директорий внутри папки clusters/ согласно Секции 1.

[ ] Задача 2: Развернуть базовые блупринты приложений в clusters/base/argocd-apps/ и файл common-values.yaml.

[ ] Задача 3: Сгенерировать файлы kustomization.yaml для сред test и prod, настроив стратегические патчи (JSON Patches) для подмены путей (/spec/source/path).

[ ] Задача 4: Сгенерировать локальные конфигурации values.yaml и prod-patch.yaml для ингресса.

[ ] Задача 5: Запустить локальную валидацию рендеринга манифестов через утилиту Kustomize, чтобы убедиться в корректности относительных путей.
