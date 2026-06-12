- [Подключение к jumpbox и скачивание  бинарников](#подключение-к-jumpbox-и-скачивание--бинарников)
- [Раскладываем всё по полочкам и наводим порядок](#раскладываем-всё-по-полочкам-и-наводим-порядок)
- [Устанавливаем kubectl](#устанавливаем-kubectl)

---

Сейчас мы будем настраивать машину `jumpbox`. Эта машина будет использоваться для администрирования нашего Kubernetes.

Все команды выполняются из под учётной записи `root`.

# Подключение к jumpbox и скачивание  бинарников

Как вы помните jumpbox у нас это машина с ip 192.168.88.11 поэтому подключаемся к ней:

```bash
ssh root@192.168.88.11
```

Созададим рабочий и перейдём в него:

```bash
mkdir kthrw01 && \
cd kthrw01
```

Скачаем туда файл с линками для скачивания всех необходимых бинарей.

```bash
wget https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/docs/downloads-amd64.txt
```
И посмотрим его содержимое:

`cat downloads-amd64.txt`

```
https://dl.k8s.io/v1.36.1/bin/linux/amd64/kubectl
https://dl.k8s.io/v1.36.1/bin/linux/amd64/kube-apiserver
https://dl.k8s.io/v1.36.1/bin/linux/amd64/kube-controller-manager
https://dl.k8s.io/v1.36.1/bin/linux/amd64/kube-scheduler
https://dl.k8s.io/v1.36.1/bin/linux/amd64/kube-proxy
https://dl.k8s.io/v1.36.1/bin/linux/amd64/kubelet
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.36.0/crictl-v1.36.0-linux-amd64.tar.gz
https://github.com/opencontainers/runc/releases/download/v1.4.2/runc.amd64
https://github.com/containernetworking/plugins/releases/download/v1.9.1/cni-plugins-linux-amd64-v1.9.1.tgz
https://github.com/containerd/containerd/releases/download/v2.3.1/containerd-2.3.1-linux-amd64.tar.gz
https://github.com/etcd-io/etcd/releases/download/v3.6.12/etcd-v3.6.12-linux-amd64.tar.gz
```

Скачиваем все бинари из этого файла:

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-amd64.txt
```

Время скачивания будет зависеть от скорости вашего интернет соединения. Скачиваться будет в районе 500Мб

```
kubectl                  100%[==================================>]  56,75M   768KB/s    за 81s     
kube-apiserver           100%[==================================>]  84,22M   687KB/s    за 1m 52s  
kube-controller-manager  100%[==================================>]  70,84M   780KB/s    за 90s     
kube-scheduler           100%[==================================>]  46,82M   792KB/s    за 60s     
kube-proxy               100%[==================================>]  42,15M   736KB/s    за 57s     
kubelet                  100%[==================================>]  57,30M   866KB/s    за 74s     
crictl-v1.36.0-linux-amd 100%[==================================>]  18,37M   778KB/s    за 27s     
runc.amd64               100%[==================================>]  11,67M   858KB/s    за 17s     
cni-plugins-linux-amd64- 100%[==================================>]  52,85M   869KB/s    за 74s     
containerd-2.3.1-linux-a 100%[==================================>]  32,94M   839KB/s    за 49s     
etcd-v3.6.12-linux-amd64 100%[==================================>]  23,67M   780KB/s    за 36s
```

Посмотрим папку куда это всё скачалось:

```bash
ls -oh downloads
итого 498M
-rw-r--r-- 1 root 53M мар 16 10:18 cni-plugins-linux-amd64-v1.9.1.tgz
-rw-r--r-- 1 root 33M мая 20 16:46 containerd-2.3.1-linux-amd64.tar.gz
-rw-r--r-- 1 root 19M апр 24 11:50 crictl-v1.36.0-linux-amd64.tar.gz
-rw-r--r-- 1 root 24M июн  1 16:55 etcd-v3.6.12-linux-amd64.tar.gz
-rw-r--r-- 1 root 85M мая 12 12:38 kube-apiserver
-rw-r--r-- 1 root 71M мая 12 12:38 kube-controller-manager
-rw-r--r-- 1 root 57M мая 12 12:38 kubectl
-rw-r--r-- 1 root 58M мая 12 12:38 kubelet
-rw-r--r-- 1 root 43M мая 12 12:38 kube-proxy
-rw-r--r-- 1 root 47M мая 12 12:38 kube-scheduler
-rw-r--r-- 1 root 12M апр  2 20:16 runc.amd64
```

# Раскладываем всё по полочкам и наводим порядок

```bash
# 1. Создаем целевую структуру каталогов
mkdir -p downloads/{client,cni-plugins,controller,worker}

# 2. Распаковываем crictl напрямую в worker
tar -xvf downloads/crictl-v1.36.0-linux-amd64.tar.gz \
  -C downloads/worker/

# 3. Распаковываем containerd со смещением в worker
tar -xvf downloads/containerd-2.3.1-linux-amd64.tar.gz \
  --strip-components 1 \
  -C downloads/worker/

# 4. Распаковываем cni-plugins в отдельный каталог
tar -xvf downloads/cni-plugins-linux-amd64-v1.9.1.tgz \
  -C downloads/cni-plugins/

# 5. Извлекаем только etcd и etcdctl в корень downloads
tar -xvf downloads/etcd-v3.6.12-linux-amd64.tar.gz \
  -C downloads/ \
  --strip-components 1 \
  etcd-v3.6.12-linux-amd64/etcdctl \
  etcd-v3.6.12-linux-amd64/etcd

# 6. Раскладываем бинарники по назначению
mv downloads/{etcdctl,kubectl} downloads/client/

mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
  downloads/controller/

mv downloads/{kubelet,kube-proxy} downloads/worker/

# 7. Переносим runc с переименованием
mv downloads/runc.amd64 downloads/worker/runc

# 8. Удаляем архивы
rm -rf downloads/*gz

# 9. Делаем бинари испольняемыми
chmod +x downloads/{client,cni-plugins,controller,worker}/*
```
# Устанавливаем kubectl

`kubectl` - это официальный инструмент командной строки для взаимодействия с `Kubernetes`. Мы его будем использовать когда поднимем куб.

Установка сводится к протому копированию файла:

```bash
cp downloads/client/kubectl /usr/local/bin/
```

Проверяем что всё работает:

```bash
kubectl version --client
```
Вывод:

```
Client Version: v1.36.1
Kustomize Version: v5.8.1
```

Тепрь наш `jumpbox` полностью заряжен всеми необходимыми инструментами для выполнения нашей миссии - понять, поднять, настроить и использовать куб.

