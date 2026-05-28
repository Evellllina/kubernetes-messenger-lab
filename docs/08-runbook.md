# Runbook: запуск и проверка лабораторной

## 1. Подготовка кластера

1. Убедиться, что установлен S3 CSI драйвер `ch.ctrox.csi.s3-driver`.
2. Проставить метки узлам:

```bash
kubectl label node <node-system> workload=system --overwrite
kubectl label node <node-app-1> workload=app --overwrite
kubectl label node <node-app-2> workload=app disk=fast --overwrite
```

3. Создать bucket в MinIO (или внешнем S3) с именем `messager-uploads`.

## 2. Проверка сборки kustomize

```bash
kubectl kustomize k8s/overlays/dev > NUL
kubectl kustomize k8s/overlays/prod > NUL
```

Обе команды должны завершаться без ошибок.

## 3. Деплой вручную (для быстрой проверки)

```bash
kubectl apply -k k8s/overlays/dev
kubectl get pods -n messager -w
```

Дождаться `Running` для Deployments и `Completed` для `migrate-users` / `migrate-messages`.

## 4. Проверка сервисов и БД-миграций

```bash
kubectl get jobs -n messager
kubectl logs job/migrate-users -n messager
kubectl logs job/migrate-messages -n messager
kubectl get svc -n messager
```

`migrate-*` должны быть `Complete`.

## 5. Проверка affinity

```bash
kubectl get pods -n messager -o wide
kubectl describe pod -n messager $(kubectl get pod -n messager -l app=postgres -o name)
kubectl describe pod -n messager $(kubectl get pod -n messager -l app=minio -o name)
kubectl describe pod -n messager $(kubectl get pod -n messager -l app=message-service -o name)
```

Проверить:

- `postgres` и `minio` на `workload=system`;
- `frontend`, `bff`, `user-service`, `message-service` на `workload=app`;
- для `message-service` в описании есть `preferred` по `disk=fast` (prod overlay).

## 6. Проверка S3 CSI монтирования

```bash
kubectl exec -n messager deploy/message-service -- sh -c "mount | grep /app/uploads"
kubectl exec -n messager deploy/message-service -- sh -c "ls -la /app/uploads"
```

Должно быть видно, что `/app/uploads` смонтирован как CSI том.

Далее через UI загрузить файл и проверить, что объект появился в bucket `messager-uploads`.

## 7. Проверка Argo CD

1. В `argocd/messager-*.yaml` заменить `repoURL` на свой GitHub репозиторий.
2. Применить Application:

```bash
kubectl apply -f argocd/messager-dev-application.yaml
kubectl apply -f argocd/messager-prod-application.yaml
kubectl get applications -n argocd
```

Ожидаемый результат: `SYNC STATUS = Synced`, `HEALTH STATUS = Healthy`.

## 8. Что приложить к сдаче

- вывод `kubectl kustomize k8s/overlays/dev` и `.../prod`;
- вывод `kubectl get pods -n messager -o wide`;
- вывод `kubectl get applications -n argocd`;
- скрин/лог загрузки файла и подтверждение, что объект есть в S3 bucket.
