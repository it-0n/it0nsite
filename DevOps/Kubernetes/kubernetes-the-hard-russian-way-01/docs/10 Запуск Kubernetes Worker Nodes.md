- [Подготовка](#подготовка)
- [Настройка Kubernetes Worker Nodes](#настройка-kubernetes-worker-nodes)
  - [Устанавливаем необходимые пакеты](#устанавливаем-необходимые-пакеты)
  - [Выключаем swap](#выключаем-swap)
  - [Создаём необходимые директории и устанавливаем бинарники](#создаём-необходимые-директории-и-устанавливаем-бинарники)
  - [Настройка CNI](#настройка-cni)
    - [Что такое CNI](#что-такое-cni)
      - [По-человечески](#по-человечески)
      - [Что он делает](#что-он-делает)
      - [Зачем это нужно](#зачем-это-нужно)
      - [Очень коротко](#очень-коротко)
    - [Создаем файл конфигурации `brige`:](#создаем-файл-конфигурации-brige)
  - [Настройка containerd](#настройка-containerd)
  - [Настройка Kubelet](#настройка-kubelet)
  - [Настройка Kubernetes Proxy](#настройка-kubernetes-proxy)
- [Запуск Worker Services](#запуск-worker-services)
- [Проверка](#проверка)


---


В этой лабораторной работе мы запустим две **worker-node**. На них будут установлены следующие компоненты:

- [runc](https://github.com/opencontainers/runc)
- [container networking plugins](https://github.com/containernetworking/cni).
- [containerd](https://github.com/containerd/containerd)
- [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet)
- [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies)

# Подготовка

Выполните следующие команды на машине `jumpbox` в каталоге `kthrw01`.

Скачиваем конфиги для настройки сети на нодах и `kubelet`:

```bash
wget -P configs https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/10-bridge.conf && \
wget -P configs https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/kubelet-config.yaml
```

Настройка скачанных конфигов и копирование на ноды:

```bash
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conf > 10-bridge.conf

  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml

  scp 10-bridge.conf kubelet-config.yaml \
  root@${HOST}:~/
done
```

Скачиваем оставшиеся нужные юнит файлы и конфиги:

```bash
wget -P units https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/units/containerd.service && \
wget -P units https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/units/kubelet.service && \
wget -P units https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/units/kube-proxy.service && \
wget -P configs https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/99-loopback.conf && \
wget -P configs https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/containerd-config.toml && \
wget -P configs https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/kube-proxy-config.yaml
```

Копируем их на ноды:

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@${HOST}:~/
done
```

И так же копируем всё содержимое папки `downloads/cni-plugins/`:

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/cni-plugins/* \
    root@${HOST}:~/cni-plugins/
done
```

# Настройка Kubernetes Worker Nodes

> Эти команды должны выполняться на **каждой** из нод: `node-0` и `node-1`.

## Устанавливаем необходимые пакеты

```bash
apt update && \
apt -y install socat conntrack ipset kmod
```
> Пакет `socat` обеспечивает поддержку команды переадресации портов `kubectl`.

## Выключаем swap

В Kubernetes поддержка **swap-памяти** сильно ограничена, потому что при её использовании трудно гарантировать предсказуемое поведение и корректно учитывать потребление памяти pod’ами.

Проверяем включен ли swap:

```bash
swapon --show
```

Если вывод пустой, вернее его нет, то swap выключен. Если же вы видите что-то вроед этого:

```
NAME      TYPE      SIZE USED PRIO
/dev/dm-2 partition 936M   0B   -2
```

То swap включен и его необходимо выключить в текущей сессии командой:

```bash
swapoff -a
```

Если вы устанавливали Debian точно как в этом руководстве, то сдедующие ваши шаги такие:

```bash
sed -i '/swap/s/^/#/' /etc/fstab && \
systemctl mask swap.target
```

> Если вы используете другой дистрибутив Linux, то изучите документацию как в нём отключить swap.

## Создаём необходимые директории и устанавливаем бинарники

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

```bash
mv crictl kube-proxy kubelet runc /usr/local/bin/ && \
mv containerd containerd-shim-runc-v2 containerd-stress /bin/ && \
mv cni-plugins/* /opt/cni/bin/
```

## Настройка CNI 

Прежде чем приступать к настройке давайте разберемся что такое CNI в Kubernetes.

### Что такое CNI
Container Network Interface (CNI)  в Kubernetes — это **стандартный способ подключать сеть к Pod’ам**. 

Если совсем просто: когда Kubernetes запускает Pod, ему нужно дать [сеть](https://kubernetes.io/ru/docs/concepts/cluster-administration/networking/) — IP-адрес, интерфейс, маршрутизацию, чтобы Pod мог общаться с другими Pod’ами и сервисами. Именно этим и занимается CNI-плагин. 

#### По-человечески
Представь, что Pod — это отдельная машина без сетевого кабеля.  
[CNI](https://www.redhat.com/de/blog/cni-kubernetes) как раз “втыкает кабель”, выдаёт адрес и настраивает связь так, чтобы Pod оказался полноценным участником сети кластера.

#### Что он делает

Обычно CNI-плагин:
- создаёт сетевой интерфейс для Pod’а;
- назначает ему IP-адрес;
- настраивает маршруты;
- обеспечивает связь с другими Pod’ами и иногда снаружи кластера.

#### Зачем это нужно

Без CNI Pod’ы просто не смогли бы нормально обмениваться данными.  
Kubernetes сам не “строит сеть” целиком — он опирается на CNI-решения, такие как Calico, Flannel, Cilium и другие.

#### Очень коротко

**CNI = сеть для Pod’ов.**  

Это механизм, который делает так, чтобы Pod в Kubernetes мог подключиться к сети и общаться с остальными.

### Создаем файл конфигурации `brige`:

```bash
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

Чтобы обеспечить обработку сетевого трафика, проходящего через мост CNI, с помощью `iptables`, загрузите и настройте модуль ядра `br-netfilter`:

```bash
modprobe br-netfilter && \
echo "br-netfilter" >> /etc/modules-load.d/modules.conf
```

```bash
{
  echo "net.bridge.bridge-nf-call-iptables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  echo "net.bridge.bridge-nf-call-ip6tables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  sysctl -p /etc/sysctl.d/kubernetes.conf
}
```

## Настройка containerd

Копируем конфиги:

```bash
{
  mkdir -p /etc/containerd/
  mv containerd-config.toml /etc/containerd/config.toml
  mv containerd.service /etc/systemd/system/
}
```

## Настройка Kubelet

Копируем конфиги:

```bash
{
  mv kubelet-config.yaml /var/lib/kubelet/
  mv kubelet.service /etc/systemd/system/
}
```

## Настройка Kubernetes Proxy

Копируем конфиги:

```bash
{
  mv kube-proxy-config.yaml /var/lib/kube-proxy/
  mv kube-proxy.service /etc/systemd/system/
}
```

# Запуск Worker Services 

```bash
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```

Проверяем:

```bash
systemctl is-active kubelet
```

Вывод долежн быть: `active`

# Проверка

Эту команду запустите на `jumpbox` в папке `kthrw01`.

```bash
ssh root@server \
  "kubectl get nodes \
  --kubeconfig admin.kubeconfig"
```

Вывод должен быть таким:

```
NAME     STATUS   ROLES    AGE     VERSION
node-0   Ready    <none>   3m9s    v1.36.1
node-1   Ready    <none>   3m14s   v1.36.1
```
