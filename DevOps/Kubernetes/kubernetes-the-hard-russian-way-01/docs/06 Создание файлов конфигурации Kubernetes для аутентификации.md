
- [Файлы конфигурации Kubernetes для аутентификации](#файлы-конфигурации-kubernetes-для-аутентификации)
- [Client Authentication Configs](#client-authentication-configs)
  - [Создание kubelet Kubernetes Configuration Files](#создание-kubelet-kubernetes-configuration-files)
  - [Создание kube-proxy Kubernetes Configuration File](#создание-kube-proxy-kubernetes-configuration-file)
  - [Создание kube-controller-manager Kubernetes Configuration File](#создание-kube-controller-manager-kubernetes-configuration-file)
  - [Создание kube-scheduler Kubernetes Configuration File](#создание-kube-scheduler-kubernetes-configuration-file)
  - [Создание admin Kubernetes Configuration File](#создание-admin-kubernetes-configuration-file)
- [Копирование Kubernetes Configuration Files](#копирование-kubernetes-configuration-files)


---


# Файлы конфигурации Kubernetes для аутентификации

В этом разделе мы создадим специальные [Kubernetes client configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) — это конфигурационные файлы Kubernetes, которые нужны клиентам для подключения к API-серверу кластера и для подтверждения своей личности. Проще говоря, такие файлы отвечают на два вопроса: **куда подключаться** и **под каким пользователем или службой** это делать.

В Kubernetes один и тот же формат kubeconfig используется разными компонентами:  
- `kubelet`;
- `kube-proxy`;
- `kube-controller-manager`;
- `kube-scheduler`;
- и пользователем `admin`.

Внутри kubeconfig обычно находятся:
- адрес Kubernetes API-сервера;
- сертификат центра сертификации;
- клиентский сертификат и закрытый ключ;
- имя пользователя или службы;
- активный контекст подключения.

Эти файлы важны потому, что Kubernetes использует сертификаты и kubeconfig как часть своей системы безопасности. Благодаря им компоненты кластера могут безопасно подключаться к API-серверу и выполнять только те действия, которые им разрешены.

В этой лабораторной работе мы создадим отдельные kubeconfig-файлы для каждого компонента, чтобы:
- `kubelet` мог безопасно представляться от имени своего узла;
- `kube-proxy` мог работать с сетевой частью кластера;
- `kube-controller-manager` и `kube-scheduler` могли взаимодействовать с API-сервером;
- `admin` мог управлять кластером вручную.

> После создания эти файлы нужно будет разложить по нужным машинам, чтобы каждый компонент получил свой конфигурационный файл и мог использовать его при запуске.

Теперь кратко пройдемся по перечисленным компонетам Kubernetes:

- **kubelet** — это агент Kubernetes, который работает **на каждом worker-узле**. Он следит за тем, чтобы нужные pod’ы были запущены, проверяет состояние контейнеров и сообщает о нём в control plane.

- **kube-proxy** — это сетевой компонент, который тоже обычно работает **на каждом узле кластера**. Он настраивает сетевые правила, чтобы трафик мог правильно попадать к нужным сервисам и pod’ам внутри кластера.

- **kube-controller-manager** — это часть **control plane**. Он запускает набор контроллеров, которые постоянно сравнивают желаемое состояние кластера с реальным и стараются привести их к совпадению.

- **kube-scheduler** — это тоже компонент **control plane**. Он решает, **на какой worker-узел** отправить новый pod, учитывая ресурсы, ограничения и текущую загрузку узлов.

Если совсем просто:

- `kubelet` — исполняет pod’ы на узле.
- `kube-proxy` — обеспечивает сетевой доступ.
- `kube-controller-manager` — следит, чтобы кластер жил “как надо”.
- `kube-scheduler` — выбирает место для новых pod’ов.

--- 

Для наглядности можно всё свести в таблицу и добавить ещё другие компоненты чтобы сложилась полная картина:

Конечно — вот расширенная таблица с `kube-apiserver` и `etcd`:

| Компонент | Где находится | Что делает | Зачем нужен |
|---|---|---|---|
| `kubelet` | На каждом worker-узле | Запускает и контролирует pod’ы на конкретной машине, следит за их состоянием | Чтобы Kubernetes мог исполнять приложения на узлах |
| `kube-proxy` | Обычно на каждом узле кластера | Настраивает сетевые правила и маршрутизацию трафика к сервисам и pod’ам | Чтобы сеть внутри кластера работала правильно |
| `kube-controller-manager` | В control plane | Запускает контроллеры, которые следят за тем, чтобы реальное состояние кластера совпадало с желаемым | Чтобы кластер автоматически восстанавливался и поддерживал нужное состояние |
| `kube-scheduler` | В control plane | Выбирает, на какой worker-узел отправить новый pod | Чтобы Kubernetes разумно распределял нагрузку по узлам |
| `kube-apiserver` | В control plane | Принимает все запросы к Kubernetes-кластеру, проверяет их и управляет доступом к объектам кластера | Это главная точка входа в Kubernetes |
| `etcd` | В control plane | Хранит состояние кластера: настройки, объекты, статусы и другую важную информацию | Это “база данных” Kubernetes |

В двух словах: 

- `kubelet` — выполняет работу на узле.
- `kube-proxy` — отвечает за сетевой доступ.
- `kube-controller-manager` — следит, чтобы кластер оставался в нужном состоянии.
- `kube-scheduler` — решает, куда запускать pod’ы.
- `kube-apiserver` — главный интерфейс управления кластером.
- `etcd` — хранилище состояния кластера.


# Client Authentication Configs

В этом разделе мы сгенерируем файлы kubeconfig для `kubelet` и пользователя `admin`.

> Все команды из этого раздела нужно выполнять на `jumpbox` каталога `kthrw01`.

## Создание kubelet Kubernetes Configuration Files

> При создании файлов kubeconfig для Kubelets необходимо использовать сертификат клиента, соответствующий имени узла Kubelet. Это гарантирует, что Kubelets должным образом авторизован [авторизатором узла Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/node/).


```bash
for host in node-0 node-1; do
  kubectl config set-cluster kubernetes-the-hard-russian-way-01 \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-russian-way-01 \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```

В резальтате создадуться два файла:

```
node-0.kubeconfig
node-1.kubeconfig
```

## Создание kube-proxy Kubernetes Configuration File

```bash
kubectl config set-cluster kubernetes-the-hard-russian-way-01 \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-russian-way-01 \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
```

В результате будет создан файл `kube-proxy.kubeconfig`.

## Создание kube-controller-manager Kubernetes Configuration File

```bash
kubectl config set-cluster kubernetes-the-hard-russian-way-01 \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-russian-way-01 \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
```
 В результате будет создан файл `kube-controller-manager.kubeconfig`.


## Создание kube-scheduler Kubernetes Configuration File

```bash
kubectl config set-cluster kubernetes-the-hard-russian-way-01 \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-russian-way-01 \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
```

 В результате будет создан файл `kube-scheduler.kubeconfig`.


 ## Создание admin Kubernetes Configuration File

 ```bash
kubectl config set-cluster kubernetes-the-hard-russian-way-01 \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-russian-way-01 \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
```

В результате будет создан файл `admin.kubeconfig`.

# Копирование Kubernetes Configuration Files

Копирование `kubelet` и `kube-proxy` kubeconfig files на `node-0` и `node-1`:

```bash
for host in node-0 node-1; do
  ssh root@${host} "mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig \
    root@${host}:/var/lib/kube-proxy/kubeconfig \

  scp ${host}.kubeconfig \
    root@${host}:/var/lib/kubelet/kubeconfig
done
```

Копирование `kube-controller-manager` и `kube-scheduler` kubeconfig files на  `server`:

```bash
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  root@server:~/
```
