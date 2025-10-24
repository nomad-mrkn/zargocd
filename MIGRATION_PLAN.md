# План миграции на динамические JIT RBAC роли

## Обзор

Этот документ описывает пошаговый план миграции с текущей статической системы JIT RBAC на новую динамическую архитектуру, которая поддерживает множественные приложения с различными наборами прав.

## Текущая архитектура vs Новая

### Было:
- Одна роль `argocd-manager-certmanager` для всех приложений
- Статические hooks в `shared/jit-rbac/`
- Жестко зашитые права в каждом приложении

### Стало:
- Отдельные роли для каждого типа приложений
- Динамические hooks в `shared/jit-rbac-dynamic/`
- Все роли в единой директории `shared/rbac-roles/`
- Имена файлов ролей соответствуют именам приложений
- Аннотации для указания требуемых прав

## Фазы миграции

### Фаза 1: Подготовка (1-2 дня)

#### 1.1 Создание новых RBAC ролей ✅
- [x] `shared/rbac-roles/certmanager.yaml`
- [x] `shared/rbac-roles/prometheus.yaml`
- [x] `shared/rbac-roles/ingress.yaml`
- [x] `shared/rbac-roles/velero.yaml`
- [x] `shared/rbac-roles/kustomization.yaml`

#### 1.2 Создание динамических hooks ✅
- [x] `shared/jit-rbac-dynamic/pre-hook.yaml`
- [x] `shared/jit-rbac-dynamic/post-hook.yaml`
- [x] `shared/jit-rbac-dynamic/kustomization.yaml`

#### 1.3 Валидация новых ролей
```bash
# Проверить синтаксис YAML
kubectl apply --dry-run=client -f shared/rbac-roles/

# Проверить права на создание ролей
kubectl auth can-i create clusterroles --as=system:serviceaccount:kube-system:argocd-manager
```

### Фаза 2: Тестирование (2-3 дня)

#### 2.1 Создание тестового кластера
```bash
# Создать kind кластер для тестирования
kind create cluster --name jit-rbac-test --config kind-config.yaml
```

#### 2.2 Развертывание новой системы на тесте
```bash
# Применить новые RBAC роли
kubectl apply -f shared/rbac-roles/

# Тестировать cert-manager с новой системой
kubectl apply -f clusters/test1/infr-apps/certmanager.yaml
```

#### 2.3 Валидация функциональности
- [ ] Проверить создание временных ClusterRoleBindings
- [ ] Проверить установку cert-manager
- [ ] Проверить очистку временных прав
- [ ] Проверить логи cleanup job'ов

### Фаза 3: Поэтапная миграция (3-5 дней)

#### 3.1 Миграция test1 кластера
```bash
# День 1: Обновить cert-manager
git checkout -b migrate-test1-certmanager
# Применить изменения в clusters/test1/infr-apps/certmanager.yaml
git commit -m "Migrate test1 cert-manager to dynamic JIT RBAC"
git push origin migrate-test1-certmanager
```

#### 3.2 Добавление новых приложений
```bash
# День 2: Добавить prometheus
git checkout -b add-test1-prometheus
# Создать clusters/test1/infr-apps/prometheus.yaml
git commit -m "Add prometheus with dynamic JIT RBAC"
git push origin add-test1-prometheus
```

#### 3.3 Миграция test4 кластера
```bash
# День 3-4: Повторить для test4
cp clusters/test1/infr-apps/certmanager.yaml clusters/test4/infr-apps/
# Обновить имена кластеров и endpoints
sed -i 's/test1/test4/g' clusters/test4/infr-apps/certmanager.yaml
sed -i 's/backend1/backend4/g' clusters/test4/infr-apps/certmanager.yaml
```

#### 3.4 Очистка старых ресурсов
```bash
# День 5: Удалить старые компоненты
rm -rf shared/jit-rbac/
rm -rf shared/jit-rbac-base/
git commit -m "Remove old static JIT RBAC system"
```

### Фаза 4: Мониторинг и оптимизация (1-2 дня)

#### 4.1 Настройка мониторинга
```yaml
# Добавить ServiceMonitor для отслеживания JIT RBAC операций
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: jit-rbac-monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: argo-jit-rbac
```

#### 4.2 Создание алертов
```yaml
# PrometheusRule для алертов на сбои JIT RBAC
groups:
- name: jit-rbac.rules
  rules:
  - alert: JITRBACCleanupFailed
    expr: increase(kube_job_failed_total{job_name=~"argocd-jit-rbac-cleanup.*"}[5m]) > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "JIT RBAC cleanup job failed"
```

## Rollback план

### В случае проблем на любой фазе:

#### Быстрый откат
```bash
# Вернуться к предыдущему коммиту
git revert HEAD
git push origin main

# Или переключиться на стабильную ветку
git checkout stable-branch
```

#### Восстановление старой системы
```bash
# Восстановить старые файлы из бэкапа
git checkout HEAD~1 -- shared/jit-rbac/
git checkout HEAD~1 -- shared/jit-rbac-base/
git checkout HEAD~1 -- clusters/*/infr-apps/certmanager.yaml
```

## Критерии успеха

### Технические метрики:
- [ ] Все приложения синхронизируются без ошибок
- [ ] Время жизни временных прав < 5 минут
- [ ] Cleanup jobs завершаются успешно в 100% случаев
- [ ] Нет избыточных прав (каждое приложение получает только нужные)

### Операционные метрики:
- [ ] Время развертывания нового приложения < 10 минут
- [ ] Возможность добавления нового типа приложения за < 1 час
- [ ] Zero downtime при миграции существующих приложений

## Контрольные точки

### После каждой фазы:
1. **Smoke test**: Проверить базовую функциональность
2. **Security audit**: Убедиться в отсутствии избыточных прав
3. **Performance check**: Проверить время синхронизации
4. **Documentation update**: Обновить документацию

## Команды для мониторинга

### Проверка состояния JIT RBAC:
```bash
# Посмотреть активные временные права
kubectl get clusterrolebindings -l app.kubernetes.io/part-of=argo-jit-rbac

# Проверить cleanup jobs
kubectl get jobs -n kube-system -l app.kubernetes.io/part-of=argo-jit-rbac

# Логи cleanup операций
kubectl logs -n kube-system -l app.kubernetes.io/part-of=argo-jit-rbac --tail=100
```

### Проверка приложений:
```bash
# Статус всех ArgoCD приложений
kubectl get applications -n argocd

# Детали конкретного приложения
kubectl describe application k8s-spb2-backend1-certmanager -n argocd
```

## Контакты и эскалация

### Ответственные:
- **DevOps Lead**: Основная ответственность за миграцию
- **Security Team**: Валидация RBAC политик
- **Platform Team**: Поддержка инфраструктуры

### Эскалация:
1. **Level 1**: Проблемы с отдельными приложениями → DevOps Lead
2. **Level 2**: Проблемы с RBAC системой → Security Team
3. **Level 3**: Критические проблемы с кластером → Platform Team + Management

---

**Дата создания**: $(date +%Y-%m-%d)  
**Версия**: 1.0  
**Статус**: Ready for execution
