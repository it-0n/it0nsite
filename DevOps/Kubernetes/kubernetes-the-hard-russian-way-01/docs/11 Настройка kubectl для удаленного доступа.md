

---

В этой лабораторной работе мы создадим файл kubeconfig для утилиты командной строки `kubectl` на основе учетных данных пользователя `admin`.

> Все команды из этого раздела нужно выполнять на `jumpbox` каталога `kthrw01`.

Ещё раз проверим что подулючение есть:

```bash
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```

Вывод должен быть таким:

```json
{
  "major": "1",
  "minor": "36",
  "emulationMajor": "1",
  "emulationMinor": "36",
  "minCompatibilityMajor": "1",
  "minCompatibilityMinor": "35",
  "gitVersion": "v1.36.1",
  "gitCommit": "756939600b9a7180fc2df6550a4585b638875e67",
  "gitTreeState": "clean",
  "buildDate": "2026-05-12T09:51:34Z",
  "goVersion": "go1.26.2",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
Создаём файл kubeconfig для аутентификации под пользователем `admin`:

```bash
{
  kubectl config set-cluster kubernetes-the-hard-russian-way-01 \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key

  kubectl config set-context kubernetes-the-hard-russian-way-01 \
    --cluster=kubernetes-the-hard-russian-way-01 \
    --user=admin

  kubectl config use-context kubernetes-the-hard-russian-way-01
}
```

Результат выполнения команды выше должен создать файл `kubeconfig` в стандартном месте `~/.kube/config`, который использует командная строка `kubectl`. Это также означает, что теперь `kubectl` можно запускать без указания файла конфигурации.

## Проверка

Дайте команду проверки версии кластера Kubernetes:

```bash
kubectl version
```

Вывод должен быть таким:

```
Client Version: v1.36.1
Kustomize Version: v5.8.1
Server Version: v1.36.1
```

Посмотрим список нод в кластере Kubernetes:

```bash
kubectl get nodes
```

Вывод должен быть примерно таким:

```
NAME     STATUS   ROLES    AGE   VERSION
node-0   Ready    <none>   75m   v1.36.1
node-1   Ready    <none>   76m   v1.36.1
```

