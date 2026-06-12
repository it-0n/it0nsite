
---

Pod’ы, назначенные на конкретный узел, получают IP-адрес из диапазона Pod CIDR этого узла. На этом этапе pod’ы ещё не могут обмениваться данными с pod’ами на других узлах, потому что не настроены сетевые [маршруты](https://cloud.google.com/compute/docs/vpc/routes).

В этой лабораторной работе мы создадим маршрут для каждого worker-node, который будет связывать диапазон Pod CIDR этого узла с его внутренним IP-адресом.

> Существуют и [другие](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) способы реализовать сетевую модель Kubernetes.

# Настройка таблиц маршрутизации

В этом разделе вы соберёте информацию, необходимую для создания маршрутов в VPC-сети `kubernetes-the-hard-russian-way-01`.

И настроите таблицу маршрутизацию в кластере.

> Все команды из этого раздела нужно выполнять на `jumpbox` каталога `kthrw01`.

```bash
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}
```

```bash
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

```bash
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

```bash
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
```

# Проверка

## Проверка на `server`:

```
ssh root@server ip route
```

Вывод:

```
default via 192.168.88.2 dev ens32 onlink 
10.200.0.0/24 via 192.168.88.13 dev ens32 
10.200.1.0/24 via 192.168.88.14 dev ens32 
192.168.88.0/24 dev ens32 proto kernel scope link src 192.168.88.12
```

## Проверка на `node-0`:

```bash
ssh root@node-0 ip route
```

Вывод:

```
default via 192.168.88.2 dev ens32 onlink 
10.200.1.0/24 via 192.168.88.14 dev ens32 
192.168.88.0/24 dev ens32 proto kernel scope link src 192.168.88.13
```

## Проверка на `node-1`:

```bash
ssh root@node-1 ip route
```

```
default via 192.168.88.2 dev ens32 onlink 
10.200.0.0/24 via 192.168.88.13 dev ens32 
192.168.88.0/24 dev ens32 proto kernel scope link src 192.168.88.14
```
