Pod’ы, назначенные на конкретный узел, получают IP-адрес из диапазона Pod CIDR этого узла. На этом этапе pod’ы ещё не могут обмениваться данными с pod’ами на других узлах, потому что не настроены сетевые [маршруты](https://cloud.google.com/compute/docs/vpc/routes).

В этой лабораторной работе мы создадим маршрут для каждого worker-node, который будет связывать диапазон Pod CIDR этого узла с его внутренним IP-адресом.

> Существуют и [другие](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) способы реализовать сетевую модель Kubernetes.

# Настройка таблиц маршрутизации

В этом разделе вы соберёте информацию, необходимую для создания маршрутов в VPC-сети `kubernetes-the-hard-russian-way-01` и настроите таблицу маршрутизации в кластере.

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
# 1. Создаем изолированный сервис для маршрутов Kubernetes
cat << 'INNER_EOF' > /etc/systemd/system/k8s-routes.service
[Unit]
Description=Add Kubernetes Static Routes
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
# Используем запуск через оболочку, чтобы PATH сам нашел утилиту ip, и гасим ошибки "File exists"
ExecStart=/bin/sh -c "ip route add ${NODE_0_SUBNET} via ${NODE_0_IP} 2>/dev/null || true"
ExecStart=/bin/sh -c "ip route add ${NODE_1_SUBNET} via ${NODE_1_IP} 2>/dev/null || true"

[Install]
WantedBy=multi-user.target
INNER_EOF

# 2. Перезагружаем конфигурацию systemd
systemctl daemon-reload

# 3. Включаем сервис в автозагрузку и запускаем его прямо сейчас
systemctl enable --now k8s-routes.service
EOF
```

```bash
ssh root@node-0 <<EOF
# 1. Создаем изолированный сервис для маршрута до node-1
cat << 'INNER_EOF' > /etc/systemd/system/node-routes.service
[Unit]
Description=Add Persistent Route to Node 1
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c "ip route add ${NODE_1_SUBNET} via ${NODE_1_IP} 2>/dev/null || true"

[Install]
WantedBy=multi-user.target
INNER_EOF

# 2. Перезагружаем конфигурацию systemd
systemctl daemon-reload

# 3. Включаем сервис в автозагрузку и запускаем его прямо сейчас
systemctl enable --now node-routes.service
EOF
```

```bash
ssh root@node-1 <<EOF
# 1. Создаем изолированный сервис для маршрута до node-0
cat << 'INNER_EOF' > /etc/systemd/system/node-routes.service
[Unit]
Description=Add Persistent Route to Node 0
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c "ip route add ${NODE_0_SUBNET} via ${NODE_0_IP} 2>/dev/null || true"

[Install]
WantedBy=multi-user.target
INNER_EOF

# 2. Перезагружаем конфигурацию systemd
systemctl daemon-reload

# 3. Включаем сервис в автозагрузку и запускаем его прямо сейчас
systemctl enable --now node-routes.service
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
