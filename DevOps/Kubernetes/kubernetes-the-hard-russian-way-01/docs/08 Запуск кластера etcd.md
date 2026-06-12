Компоненты Kubernetes сами по себе не хранят состояние локально — вся информация о кластере сохраняется в `etcd`. В этой лабораторной работе вы поднимете одиночный кластер `etcd` на одной машине.

# Подготовка

> Все команды из этого раздела нужно выполнять на `jumpbox` каталога `kthrw01`.

Создаем каталог `units` и скачиваем в него `unit` файл для сервиса `etcd`:

```bash
mkdir -p units && \
wget -P units https://raw.githubusercontent.com/it-0n/it0nsite/refs/heads/main/DevOps/Kubernetes/kubernetes-the-hard-russian-way-01/units/etcd.service
```

Копируем бинарники `etcd` и systemd unit файл на `server`:

```bash
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

# Запуск etcd кластера на server

> Все команды из этого раздела нужно выполнять на `server`.

Зайдите на `server` с `jumpbox`:

```
ssh root@server
```

Вы окажетесь в домашнем каталоге пользователя `root`. Все команды запускаются из этого каталога.

## Установка бинарников etcd

```bash
mv etcd etcdctl /usr/local/bin/
```

## Конфигурируем etcd сервис

```bash
mkdir -p /etc/etcd /var/lib/etcd && \
chmod 700 /var/lib/etcd && \
cp ca.crt kube-api-server.key kube-api-server.crt /etc/etcd/
```
Каждый участник кластера `etcd` должен иметь уникальное имя. Поэтому имя `etcd` нужно задать так, чтобы оно совпадало с именем хоста текущей вычислительной машины.

Запускаем сервис `etcd`:

```bash
mv etcd.service /etc/systemd/system/ && \
systemctl daemon-reload && \
systemctl enable etcd && \
systemctl start etcd && \
systemctl status etcd --no-pager
```

## Проверка работы etcd

Выведем список членов кластера etcd:

```bash
etcdctl member list
```

Вывод:

```
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```
