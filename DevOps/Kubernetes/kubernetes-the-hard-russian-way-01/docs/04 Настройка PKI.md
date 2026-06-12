- [Подготовка центра сертификации и генерация сертификатов TLS](#подготовка-центра-сертификации-и-генерация-сертификатов-tls)
  - [Центр сертификации](#центр-сертификации)
    - [Создание закрытого ключа и сертификата для CA](#создание-закрытого-ключа-и-сертификата-для-ca)
  - [Создание клиентских и серверных сертификатов](#создание-клиентских-и-серверных-сертификатов)
- [Копирование клиентских и серверных сертификатов](#копирование-клиентских-и-серверных-сертификатов)


---

# Подготовка центра сертификации и генерация сертификатов TLS

В этой лабораторной работе вы создадите PKI-инфраструктуру с помощью `openssl`. Сначала вы подготовите центр сертификации, а затем сгенерируете TLS-сертификаты для основных компонентов Kubernetes:
- **kube-apiserver**
- **kube-controller-manager**
- **kube-scheduler**
- **kubelet**
- **kube-proxy**

Все команды из этого раздела нужно выполнять на `jumpbox` каталога `kthrw01`.

---

## Центр сертификации

В этом разделе мы настроим **центр сертификации**, который будет использоваться для создания TLS-сертификатов для компонентов Kubernetes.

Создание центра сертификации и выпуск сертификатов с помощью `openssl` может занять много времени, особенно если вы делаете это впервые. Чтобы упростить работу был подготовлен файл конфигурации `ca.conf`. В нём указаны все нужные параметры для создания сертификатов каждого компонента Kubernetes.

Скачаем файл `ca.conf`:

```bash
wget https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/configs/ca.conf
```

Бегло просмотрите скачанный файл:

```
cat ca.conf
```

Вам не нужно разбираться во всех деталях файла `ca.conf`, чтобы пройти это руководство. Но полезно воспринимать его как удобную отправную точку. С его помощью можно лучше понять, как работает `openssl` и как устроено управление сертификатами.

Любой центр сертификации начинается с двух вещей: **закрытого ключа** и **корневого сертификата**. В этом разделе мы создадим **самоподписанный центр сертификации**. Для этого руководства этого достаточно, но в реальной production-среде такой подход обычно не используют.

### Создание закрытого ключа и сертификата для CA

```bash
openssl genrsa -out ca.key 4096 && \
openssl req -x509 -new -sha512 -noenc \
-key ca.key -days 3653 \
-config ca.conf \
-out ca.crt
```

В результате появятcя два файла: `ca.key`  и `ca.crt`

## Создание клиентских и серверных сертификатов

В этом разделе мы создадим сертификаты клиента и сервера для каждого компонента Kubernetes, а также сертификат клиента для пользователя Kubernetes admin.

```bash
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)
```

```bash
for i in ${certs[*]}; do
  openssl genrsa -out "${i}.key" 4096

  openssl req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} \
    -out "${i}.csr"

  openssl x509 -req -days 3653 -in "${i}.csr" \
    -copy_extensions copyall \
    -sha256 -CA "ca.crt" \
    -CAkey "ca.key" \
    -CAcreateserial \
    -out "${i}.crt"
done
```

Вывод команды должен быть такой:

```
Certificate request self-signature ok
subject=CN=admin, O=system:masters
Certificate request self-signature ok
subject=CN=system:node:node-0, O=system:nodes, C=RU, ST=MoscowReg, L=Moscow
Certificate request self-signature ok
subject=CN=system:node:node-1, O=system:nodes, C=RU, ST=MoscowReg, L=Moscow
Certificate request self-signature ok
subject=CN=system:kube-proxy, O=system:node-proxier, C=RU, ST=MoscowReg, L=Moscow
Certificate request self-signature ok
subject=CN=system:kube-scheduler, O=system:system:kube-scheduler, C=RU, ST=MoscowReg, L=Moscow
Certificate request self-signature ok
subject=CN=system:kube-controller-manager, O=system:kube-controller-manager, C=RU, ST=MoscowReg, L=Moscow
Certificate request self-signature ok
subject=CN=kubernetes, C=RU, ST=MoscowReg, L=Moscow
Certificate request self-signature ok
subject=CN=service-accounts
```

В результате выполнения приведенной выше команды будут сгенерированы закрытый ключ, запрос на сертификат и подписанный SSL-сертификат для каждого из компонентов Kubernetes. Вы можете перечислить сгенерированные файлы с помощью следующей команды:

```bash
ls -1 *.crt *.key *.csr
```
Вывод:

```
admin.crt
admin.csr
admin.key
ca.crt
ca.key
kube-api-server.crt
kube-api-server.csr
kube-api-server.key
kube-controller-manager.crt
kube-controller-manager.csr
kube-controller-manager.key
kube-proxy.crt
kube-proxy.csr
kube-proxy.key
kube-scheduler.crt
kube-scheduler.csr
kube-scheduler.key
node-0.crt
node-0.csr
node-0.key
node-1.crt
node-1.csr
node-1.key
service-accounts.crt
service-accounts.csr
service-accounts.key
```
# Копирование клиентских и серверных сертификатов

В этом разделе мы разложим сертификаты по нужным машинам и сохраним их в тех местах, где Kubernetes будет искать сертификаты и ключи для своих компонентов. В реальной среде к таким файлам нужно относиться как к **секретным данным**, потому что Kubernetes использует их для взаимной аутентификации компонентов между собой.

Скопируем нужные сертификаты и закрытые ключи на машины **node-0** и **node-1**.

```bash
for host in node-0 node-1; do
  ssh root@${host} mkdir /var/lib/kubelet/

  scp ca.crt root@${host}:/var/lib/kubelet/

  scp ${host}.crt \
    root@${host}:/var/lib/kubelet/kubelet.crt

  scp ${host}.key \
    root@${host}:/var/lib/kubelet/kubelet.key
done
```

Копируем соответствующие сертификаты и закрытые ключи на машину `server`:

```bash
scp \
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~/
```

> Клиентские сертификаты `kube-proxy`, `kube-controller-manager`, `kube-scheduler` и `kubelet` будут использоваться для создания файлов конфигурации аутентификации клиентов в следующей лабораторной работе.