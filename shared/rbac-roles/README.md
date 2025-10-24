# RBAC Роли для приложений

Эта директория содержит все ClusterRole ресурсы для различных типов приложений, используемых в системе динамических JIT RBAC.

## Структура

```
rbac-roles/
├── certmanager.yaml    # Роль для cert-manager приложений
├── prometheus.yaml     # Роль для мониторинга (Prometheus, Grafana, AlertManager)
├── ingress.yaml        # Роль для ingress контроллеров
├── velero.yaml         # Роль для backup/restore операций
├── kustomization.yaml  # Kustomize конфигурация для всех ролей
└── README.md          # Этот файл
```

## Принцип работы

Каждый файл содержит ClusterRole с минимальными правами, необходимыми для конкретного типа приложения:

- **certmanager.yaml**: Права на cert-manager.io, acme.cert-manager.io API
- **prometheus.yaml**: Права на monitoring.coreos.com, grafana.integreatly.org API  
- **ingress.yaml**: Права на networking.k8s.io, gateway.networking.k8s.io API
- **velero.yaml**: Права на velero.io, snapshot.storage.k8s.io API

## Использование в приложениях

### 1. Указать требуемую роль в аннотациях:
```yaml
metadata:
  annotations:
    jit-rbac/required-role: "argocd-manager-certmanager"
```

### 2. Добавить источник роли:
```yaml
sources:
  - repoURL: 'https://github.com/nomad-mrkn/zargocd.git'
    targetRevision: HEAD
    path: shared/rbac-roles
    directory:
      include: certmanager.yaml  # Имя файла роли
```

### 3. Добавить динамические hooks:
```yaml
  - repoURL: 'https://github.com/nomad-mrkn/zargocd.git'
    targetRevision: HEAD
    path: shared/jit-rbac-dynamic
    kustomize:
      patchesJson6902:
        - target:
            kind: ClusterRoleBinding
            name: argo-jit-rbac-binding
          patch: |-
            - op: replace
              path: /roleRef/name
              value: argocd-manager-certmanager  # Имя роли
```

## Добавление новой роли

### 1. Создать файл роли:
```bash
# Создать новый файл с именем приложения
touch shared/rbac-roles/myapp.yaml
```

### 2. Определить права:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager-myapp
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
  labels:
    app.kubernetes.io/part-of: argo-jit-rbac-roles
    jit-rbac/app-type: myapp
rules:
  - apiGroups: ["myapp.io"]
    resources: ["*"]
    verbs: ["create", "update", "patch", "get", "list", "watch", "delete"]
```

### 3. Добавить в kustomization.yaml:
```yaml
resources:
  - myapp.yaml  # Добавить новый файл
```

### 4. Использовать в приложении:
```yaml
metadata:
  annotations:
    jit-rbac/required-role: "argocd-manager-myapp"
sources:
  - path: shared/rbac-roles
    directory:
      include: myapp.yaml
```

## Безопасность

### Принципы:
- **Least Privilege**: Каждая роль содержит минимальные права
- **Isolation**: Роли изолированы друг от друга
- **Temporary**: Права предоставляются только на время синхронизации
- **Auditable**: Все операции логируются и отслеживаются

### Мониторинг:
```bash
# Посмотреть активные временные права
kubectl get clusterrolebindings -l app.kubernetes.io/part-of=argo-jit-rbac

# Проверить использование ролей
kubectl get clusterroles -l app.kubernetes.io/part-of=argo-jit-rbac-roles
```

## Troubleshooting

### Проблема: Приложение не может создать ресурсы
**Решение**: Проверить, что роль содержит нужные права:
```bash
kubectl describe clusterrole argocd-manager-myapp
```

### Проблема: Временные права не удаляются
**Решение**: Проверить логи cleanup job'а:
```bash
kubectl logs -n kube-system -l app.kubernetes.io/part-of=argo-jit-rbac
```

### Проблема: Конфликт прав между приложениями
**Решение**: Убедиться, что каждое приложение использует свою роль и уникальные labels.

---

**Дата создания**: $(date +%Y-%m-%d)  
**Версия**: 2.0  
**Статус**: Production Ready
