

---

Сейчас мы будем поднимать Control Plane Kubernetes.

На машине `server` будут установлены следующие компоненты:
- Kubernetes API Server
- Scheduler
- Controller Manager

# Подготовка

> Все команды из этого раздела нужно выполнять на `jumpbox` каталога `kthrw01`.

Скачиваем unit и конфиг фалы:

```bash
wget -P units https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/units/kube-apiserver.service && \
wget -P units https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/units/kube-controller-manager.service && \
wget -P units https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/units/kube-scheduler.service && \
wget -P configs https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/kube-scheduler.yaml && \
wget -P configs https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/kube-apiserver-to-kubelet.yaml
```

Копируем бинарники, конфиги и unit файлы Kubernetes на `server`:

```bash
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/
```

# Настройка компонентов Kubernetes Control Plane

> Все команды из этого и последующих разделов нужно выполнять на `server` в домашнем каталоге пользователя `root` если не указано иное.

Зайдите на `server` с `jumpbox`:

```
ssh root@server
```

Создаём директорию для конфигов Kubernetes:

```bash
mkdir -p /etc/kubernetes/config
```

## Установка бинарей Kubernetes

```bash
mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
```

## Кофигурирование the Kubernetes API Server

```bash
mkdir -p /var/lib/kubernetes/ && \
mv ca.crt ca.key \
kube-api-server.key kube-api-server.crt \
service-accounts.key service-accounts.crt \
encryption-config.yaml \
/var/lib/kubernetes/ && \
mv kube-apiserver.service /etc/systemd/system/kube-apiserver.service
```

## Конфигурирование Kubernetes Controller Manager

```bash
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/ && \
mv kube-controller-manager.service /etc/systemd/system/
```

## Конфигурирование Kubernetes Scheduler

```bash
mv kube-scheduler.kubeconfig /var/lib/kubernetes/ && \
mv kube-scheduler.yaml /etc/kubernetes/config/ && \
mv kube-scheduler.service /etc/systemd/system/
```

# Запуск компонентов Kubernetes Control Plane

```bash
systemctl daemon-reload && \
systemctl enable kube-apiserver \
kube-controller-manager kube-scheduler && \
systemctl start kube-apiserver \
    kube-controller-manager kube-scheduler
```

> Подождем секунд 10 пока всё это добро запустится.

Проверим что сервисы запустились:

```bash
systemctl status kube-apiserver \
    kube-controller-manager kube-scheduler && \
systemctl is-active kube-apiserver \
    kube-controller-manager kube-scheduler
```

# Проверка работы Kubernetes Control Plane

На этом этапе компоненты Kubernetes Control Plane уже должны быть запущены и работать. Проверьте это с помощью командной строки kubectl:

```bash
kubectl cluster-info --kubeconfig admin.kubeconfig
```

Вывод должен быть такой:

```
Kubernetes control plane is running at https://127.0.0.1:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

# RBAC для авторизации Kubelet

Role-Based Access Control (RBAC) — это модель управления доступом, где права выдаются не “всем подряд”, а по ролям.
Проще говоря, RBAC определяет, кто и что может делать в Kubernetes.

В этом разделе мы настроим права RBAC так, чтобы Kubernetes API Server мог обращаться к Kubelet API на каждой worker-node. Такой доступ нужен, чтобы получать метрики, смотреть логи и выполнять команды внутри pod’ов.

>В этом руководстве для Kubelet включён режим авторизации `Webhook` через флаг `--authorization-mode`. В этом режиме Kubelet обращается к API [SubjectAccessReview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#checking-api-access), чтобы проверить, разрешено ли выполнить запрос.

> Команды из этого раздела влияют на весь кластер, поэтому их нужно выполнять только на машине `server` из домашнего каталога пользователя `root`.

Создаём [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) `system:kube-apiserver-to-kubelet` с правами на доступ к API Kubelet и выполнение большинства стандартных операций, связанных с управлением pod’ами:

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig
```

## Проверка:

На этом этапе Kubernetes Control Plane уже должен быть запущен и работать. Выполните следующие команды на машине `jumpbox` в каталоге `kthrw01`, чтобы проверить, что всё в порядке:

```bash
curl --cacert ca.crt https://server.kubernetes.local:6443/version
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

