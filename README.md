# Микросервисная платформа голосования

## Архитектура

```
Браузер
  ├─→ vote.home   → [Vote App x2] → Redis → [Worker] → PostgreSQL
  └─→ result.home → [Result App x2] ──────────────────────↑
```

| Сервис     | Образ                                      | Реплики | Хранилище |
|------------|--------------------------------------------|---------|-----------|
| vote       | dockersamples/examplevotingapp_vote         | 2       | —         |
| result     | dockersamples/examplevotingapp_result       | 2       | —         |
| worker     | dockersamples/examplevotingapp_worker       | 1       | —         |
| redis      | redis:alpine                               | 1       | —         |
| postgres   | postgres:15-alpine                         | 1       | Longhorn 1Gi |

---

## Структура репозитория

```
k8s-final-project/
├── argocd/
│   └── application.yaml      # ← Единственное, что применяешь руками
└── k8s/
    ├── kustomization.yaml    # Точка входа для ArgoCD
    ├── namespace.yaml
    ├── postgres/
    │   ├── secret.yaml
    │   ├── statefulset.yaml
    │   └── service.yaml
    ├── redis/
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── worker/
    │   └── deployment.yaml
    ├── vote/
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── result/
    │   ├── deployment.yaml
    │   └── service.yaml
    └── ingress/
        └── ingress.yaml
```

---

## Деплой (шаг за шагом)

### 1. Залить репозиторий на GitHub

```bash
git init
git add .
git commit -m "feat: initial voting platform manifests"
git remote add origin https://github.com/YOUR_USERNAME/k8s-final-project
git push -u origin main
```

### 2. Применить ArgoCD Application (единственный ручной kubectl apply)

```bash
# Заменить YOUR_USERNAME в argocd/application.yaml перед применением!
kubectl apply -f argocd/application.yaml
```

После этого ArgoCD сам развернёт весь проект.

### 3. Проверить статус в ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Открыть https://localhost:8080
```

Приложение `voting-app` должно стать `Synced` + `Healthy`.

### 4. Настроить /etc/hosts (для доступа по доменам)

```bash
# Узнать IP Ingress Controller-а
kubectl get svc -n ingress-nginx

# Добавить в /etc/hosts (Linux/Mac) или C:\Windows\System32\drivers\etc\hosts (Windows)
<IP_INGRESS>  vote.home result.home
```

### 5. Проверить работу

- Открыть http://vote.home — проголосовать за Cats или Dogs
- Открыть http://result.home — убедиться, что голос отображается

---

## DNS-имена сервисов (как поды находят друг друга)

| Клиент     | Ищет   | DNS в кластере        |
|------------|--------|-----------------------|
| Vote App   | Redis  | `redis` (ClusterIP)   |
| Worker     | Redis  | `redis`               |
| Worker     | DB     | `db`                  |
| Result App | DB     | `db`                  |

---

## Troubleshooting

```bash
# Посмотреть все поды в namespace voting
kubectl get pods -n voting

# Логи конкретного пода
kubectl logs -n voting deployment/worker

# Проверить что PVC создан Longhorn-ом
kubectl get pvc -n voting

# Проверить Ingress
kubectl describe ingress voting-ingress -n voting
```
