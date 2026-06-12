
- [Что такое Kubernetes control plane и worker nodes](#что-такое-kubernetes-control-plane-и-worker-nodes)
  - [Control plane](#control-plane)
  - [Worker nodes](#worker-nodes)
  - [Простая аналогия](#простая-аналогия)
  - [Зачем это разделение нужно](#зачем-это-разделение-нужно)
  - [Коротко](#коротко)
- [Создание файла базы машин](#создание-файла-базы-машин)
- [Создание и копирование SSH ключей](#создание-и-копирование-ssh-ключей)
  - [Генерируем SSH ключ](#генерируем-ssh-ключ)
  - [Копируем SSH ключ на все машины](#копируем-ssh-ключ-на-все-машины)
- [Настройка hostnames](#настройка-hostnames)
- [Настройка файла /etc/hosts](#настройка-файла-etchosts)
  - [Создаём файл hosts](#создаём-файл-hosts)
  - [Добавляем записи из файла hosts в файл /etc/hosts на jumpbox](#добавляем-записи-из-файла-hosts-в-файл-etchosts-на-jumpbox)
  - [Добавляем записи из файла hosts в файл /etc/hosts на другие машины](#добавляем-записи-из-файла-hosts-в-файл-etchosts-на-другие-машины)

---

Kubernetes нужен набор машин для размещения **Kubernetes control plane**  и **worker-nodes**, на которых в конечном итоге запускаются контейнеры.

В этой части мы подготовим машины, необходимые для настройки Kubernetes-кластера.

# Что такое Kubernetes control plane и worker nodes

Kubernetes-кластер обычно делится на две большие части: **control plane** и **worker nodes**.  
Проще говоря, control plane — это “мозг” кластера, а worker nodes — это “рабочие лошадки”, где реально запускаются контейнеры (приложения).

## Control plane

**Control plane** отвечает за управление всем кластером. Он хранит состояние кластера, решает, какие приложения где должны работать, следит за их состоянием и переносит нагрузку, если что-то ломается.  
Именно **control plane** принимает решения вроде: сколько копий приложения нужно запустить, на каких узлах их разместить и что делать, если один из узлов перестал отвечать.

Внутри control plane обычно находятся такие компоненты:
- **API server** — точка входа для всех команд и запросов к кластеру.
- **Scheduler** — выбирает, на какой worker node запустить новый pod.
- **Controller manager** — следит, чтобы фактическое состояние кластера совпадало с желаемым.
- **etcd** — распределённая база данных, где хранится состояние кластера.

Если **control plane** перестаёт работать, кластер теряет способность нормально управляться, даже если сами контейнеры на **worker nodes** ещё продолжают работать.

## Worker nodes

**Worker nodes** — это машины, на которых выполняются ваши приложения.  
На них Kubernetes запускает контейнеры, объединённые в pods. У **worker node** есть всё, что нужно для исполнения workload’ов: контейнерный runtime, kubelet и сетевые компоненты.

**Worker node** делает практическую работу:
- принимает pod’ы от **control plane**;
- запускает контейнеры;
- следит за их состоянием;
- сообщает **control plane**, живы ли приложения и сколько ресурсов они используют.

Если **worker node** выходит из строя, Kubernetes может перенести workload на другую машину, если в кластере есть резерв и приложение настроено на отказоустойчивость.

## Простая аналогия

Можно представить Kubernetes как ресторан:
- **control plane** — это управляющий персонал и шеф-повар, которые решают, что и где готовить;
- **worker nodes** — это повара и плиты, где фактически готовятся блюда.

То есть управление и исполнение разделены: одни принимают решения, другие выполняют.

## Зачем это разделение нужно

Такой дизайн делает Kubernetes:
- более масштабируемым;
- более устойчивым к сбоям;
- удобным для автоматизации;
- пригодным для больших распределённых систем.

**Control plane** может централизованно управлять десятками, сотнями или тысячами **worker nodes**, не вмешиваясь в выполнение каждого контейнера вручную.

## Коротко

- **Control plane** управляет кластером и принимает решения.
- **Worker nodes** запускают приложения.
- Вместе они образуют Kubernetes-кластер.

Сейчас нам сначала нужно подготовить эти машины, а потом уже разворачивать сам кластер.


Вот красивый перевод:

***

# Создание файла базы машин

Сейчас мы создадим текстовый файл, который будет служить **базой машин** и хранить различные атрибуты серверов, необходимых для настройки **Kubernetes control plane** и **worker-nodes**.

Ниже показана схема записей в базе машин: **одна запись на строку**.

```text
IPV4_ADDRESS FQDN HOSTNAME POD_SUBNET
```

Каждый столбец означает следующее:

- **IPV4_ADDRESS** — IPv4-адрес машины.
- **FQDN** — полное доменное имя, например `server.kubernetes.local`.
- **HOSTNAME** — имя хоста, например `server` или `node-0`.
- **POD_SUBNET** — IP-подсеть для pod’ов.

Kubernetes назначает **один IP-адрес на каждый pod**, а поле **POD_SUBNET** показывает уникальный диапазон IP-адресов, который назначается каждой машине в кластере для этой цели.

Ниже приведён пример файла `machines.txt`, который вы можете использовать. Обратите внимание, что IP-адреса скрыты. Если вы создали машины со своими IP-адресами, то просто подствьте их сюда.

```text
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0 10.200.0.0/24
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1 10.200.1.0/24
```
> Внимание! Каждая машина должна быть доступна с других машин и с `jumpbox`.

Теперь ваша задача — создать файл `machines.txt` с данными для трёх машин, которые будут использоваться для создания Kubernetes-кластера. Используйте пример выше как шаблон и добавьте сведения о своих машинах.

В моём варианте с машинами которые мы создали содержимое этого файла будет выглядеть так.

```bash
cat machines.txt
```
Вывод:

```text
192.168.88.12 server.kubernetes.local server
192.168.88.13 node-0.kubernetes.local node-0 10.200.0.0/24
192.168.88.14 node-1.kubernetes.local node-1 10.200.1.0/24
```
> Внимание! Файл создаётся в каталоге `kthrw01` на `jumpbox`.

# Создание и копирование SSH ключей

Эти ключи будут использоваться для выполнения команд на наших машинах на протяжении всего руководства. 

Выполните следующие команды с компьютера `jumpbox` из каталога `kthrw01` где вы создали файл `machines.txt`.

## Генерируем SSH ключ

```bash
ssh-keygen
```
```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519): 
Enter passphrase for "/root/.ssh/id_ed25519" (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:Nb0xhKzg47UdwMy6pchP6dkO1Z0E1DEQoWQRED+3x0U root@debian-n01
The key's randomart image is:
+--[ED25519 256]--+
|      o*==**+.E  |
|      .+=.o+.o   |
|     . o+o+ = .  |
|      + +=.* *   |
|   . o BSoo.*    |
|    o *.. ..     |
|     +.o         |
|      +..        |
|       ..        |
+----[SHA256]-----+
```

## Копируем SSH ключ на все машины

```bash
while read IP FQDN HOST SUBNET; do
  ssh-copy-id root@${IP}
done < machines.txt
```

После того как ключ добавили на машины, проверяем что всё работает:

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname
done < machines.txt
```

Вывод в моём случае такой:

```
debian-n02
debian-n03
debian-n04
```

# Настройка hostnames

Сейчас мы зададим имена хостов для машин **server**, **node-0** и **node-1**. Эти имена пригодятся, когда вы будете запускать команды с `jumpbox` для каждой из машин.  

Кроме того, имя хоста важно и внутри самого Kubernetes-кластера. Вместо того чтобы обращаться к серверу Kubernetes API по IP-адресу, клиенты будут использовать его имя хоста. Точно так же рабочие узлы **node-0** и **node-1** будут использовать свои имена, когда будут подключаться и регистрироваться в кластере Kubernetes.

Выполните следующие команды с компьютера `jumpbox` из каталога `kthrw01` где вы создали файл `machines.txt`.

```bash
while read IP FQDN HOST SUBNET; do
    CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"
    ssh -n root@${IP} "$CMD"
    ssh -n root@${IP} hostnamectl set-hostname ${HOST}
    ssh -n root@${IP} systemctl restart systemd-hostnamed
done < machines.txt
```

Проверяем:

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt
```
Вывод должен быть таким:

```
server.kubernetes.local
node-0.kubernetes.local
node-1.kubernetes.local
```

# Настройка файла /etc/hosts

В этом разделе вы создадите файл `hosts`, который затем будет добавлен в `/etc/hosts` на `jumpbox` и на всех трёх машинах кластера, используемых в этом руководстве. Это позволит обращаться к каждой машине по её имени, например `server`, `node-0` или `node-1`, вместо того чтобы запоминать IP-адреса.

Выполните следующие команды с компьютера `jumpbox` из каталога `kthrw01` где вы создали файл `machines.txt`.

## Создаём файл hosts

```bash
echo "" > hosts
echo "# Kubernetes The Hard Russian Way 01" >> hosts
```

Добавляем к записям в /etc/hosts на всех машинах:

```bash
while read IP FQDN HOST SUBNET; do
    ENTRY="${IP} ${FQDN} ${HOST}"
    echo $ENTRY >> hosts
done < machines.txt
```

Проверяем:

```bash
cat hosts
```

Вывод в моём случае такой:

```

# Kubernetes The Hard Russian Way 01
192.168.88.12 server.kubernetes.local server
192.168.88.13 node-0.kubernetes.local node-0
192.168.88.14 node-1.kubernetes.local node-1
```

## Добавляем записи из файла hosts в файл /etc/hosts на jumpbox

```bash
cat hosts >> /etc/hosts
```

Проверяем:

```bash
cat /etc/hosts
```

Вывод в моём случае такой:

```
127.0.0.1	localhost
127.0.1.1	debian-n01

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# Kubernetes The Hard Russian Way 01
192.168.88.12 server.kubernetes.local server
192.168.88.13 node-0.kubernetes.local node-0
192.168.88.14 node-1.kubernetes.local node-1
```

После этого у вас должен быть доступ по SSH к каждой машине в файле `machines.txt` по hostname.

```bash
for host in server node-0 node-1
   do ssh root@${host} hostname
done
```

Вывод должен быть таким:

```
server
node-0
node-1
```

> Внимание! Когда вы эту команду будете давать в первый раз, то вам будет предложено сохранить отпечаток, например:

```
The authenticity of host 'server (192.168.88.12)' can't be established.
ED25519 key fingerprint is SHA256:Mvdc7NIql3v8+PL1Zu/C2g161WBVQKlm6Wd6KytNIsU.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:1: [hashed name]
    ~/.ssh/known_hosts:4: [hashed name]
    ~/.ssh/known_hosts:5: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'server' (ED25519) to the list of known hosts.
server
```

## Добавляем записи из файла hosts в файл /etc/hosts на другие машины

Выполните следующие команды с компьютера `jumpbox` из каталога `kthrw01` где вы создали файл `machines.txt`.

```bash
while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n \
    root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt
```
На данный момент имена хостов можно использовать при подключении к компьютерам с вашего компьютера **jumpbox** или с любого из трех компьютеров в кластере Kubernetes. Вместо использования IP-адресов теперь вы можете подключаться к компьютерам, используя имя хоста, такое как **server**, **node-0** или **node-1**.

