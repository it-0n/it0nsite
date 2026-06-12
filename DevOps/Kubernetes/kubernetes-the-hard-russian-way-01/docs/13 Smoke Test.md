
В этой лабораторной работе мы выполним ряд задач, чтобы убедиться, что  Kubernetes-кластер работает правильно.


# Шифрование данных

В этом разделе мы проверим, что секретные данные могут быть зашифрованы.

> Все команды из этого раздела нужно выполнять на `jumpbox` каталога `kthrw01`.

Создадим секрет:

```bash
kubectl create secret generic kubernetes-the-hard-russian-way-01 \
  --from-literal="mykey=mydata"
```

Выведем hexdump секрета kubernetes-the-hard-russian-way-01, хранящегося в etcd:

```bash
ssh root@server \
    'etcdctl get /registry/secrets/default/kubernetes-the-hard-russian-way-01 | hexdump -C'
```

Вывод должен быть примерно таким:

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 72 75  |etes-the-hard-ru|
00000030  73 73 69 61 6e 2d 77 61  79 2d 30 31 0a 6b 38 73  |ssian-way-01.k8s|
00000040  3a 65 6e 63 3a 61 65 73  63 62 63 3a 76 31 3a 6b  |:enc:aescbc:v1:k|
00000050  65 79 31 3a 29 2b be 4d  53 17 58 38 f5 06 9f db  |ey1:)+.MS.X8....|
00000060  1b d5 db 68 02 21 6a 4f  8b 87 81 1a 5f 57 54 8f  |...h.!jO...._WT.|
00000070  7e c6 43 ec 38 93 a5 38  47 4c 79 b6 1d c7 e6 52  |~.C.8..8GLy....R|
00000080  d0 d0 53 3d 3f f3 88 6f  b5 18 c1 e5 9f c9 df bb  |..S=?..o........|
00000090  9e 5a e0 81 15 4d 7d 98  de 4b c3 9f 49 2a 4c 7d  |.Z...M}..K..I*L}|
000000a0  b7 0c 10 cd b8 01 53 3c  d6 e9 29 a7 87 e6 f0 d6  |......S<..).....|
000000b0  7d f0 61 0f a9 6f 60 4e  22 76 03 0c 68 3f 7a 93  |}.a..o`N"v..h?z.|
000000c0  cb 18 8a 99 fb 5b 5f 7b  eb f6 7a 89 be e7 75 b8  |.....[_{..z...u.|
000000d0  2c 6c 90 2d 0a 3a 08 e2  41 9c 01 ff 67 e6 00 b3  |,l.-.:..A...g...|
000000e0  04 d3 85 b9 12 a6 f8 fc  1d ad 77 14 8f c9 4a 3f  |..........w...J?|
000000f0  2b 78 12 b3 cc 12 81 69  b4 1d 92 1d e0 e4 d6 50  |+x.....i.......P|
00000100  b0 cb 12 28 58 4e 69 c0  cb 56 2f 49 67 ee 3b 89  |...(XNi..V/Ig.;.|
00000110  6b 96 b2 7b b9 9d e9 9f  ab dc 04 60 47 3e 1e ec  |k..{.......`G>..|
00000120  69 b0 5b 68 40 ff fc da  c3 33 82 01 b5 84 87 1f  |i.[h@....3......|
00000130  7e 15 16 9d 79 bd 16 39  31 8a ac 3c 7c 0a fa 40  |~...y..91..<|..@|
00000140  d1 2f 6c f2 c8 40 0f 1b  c8 6c 22 df f8 cb 2d 63  |./l..@...l"...-c|
00000150  0e 91 34 a8 db 22 87 a0  30 7d 75 c8 4f 69 eb d4  |..4.."..0}u.Oi..|
00000160  9f d6 db fd 0a                                    |.....|
00000165
```

Ключ в etcd должен начинаться с `k8s:enc:aescbc:v1:key1`, что означает: для шифрования данных был использован провайдер `aescbc`, а сам ключ шифрования — `key1`.

# Deployments

В этом разделе мы проверим, что Kubernetes умеет создавать и управлять [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)’ами.

Задеплоим nginx web server:

```bash
kubectl create deployment nginx \
  --image=nginx:latest
```

Посмотрим список pod’ов, созданных nginx-deployment’ом:

```bash
kubectl get pods -l app=nginx
```

Вывод:

```
NAME                    READY   STATUS              RESTARTS   AGE
nginx-6797d5487-2gvp9   0/1     ContainerCreating   0          99s
```

# Port Forwarding

В этом разделе мы проверим, можно ли подключаться к приложениям удалённо с помощью [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Получим полное имя pod’а nginx:

```bash
POD_NAME=$(kubectl get pods -l app=nginx \
  -o jsonpath="{.items[0].metadata.name}")
```

Перенаправим порт 8080 на `jumpbox` на прот 80 в nginx поде:

```bash
kubectl port-forward $POD_NAME 8080:80
```

Вывод будет таким (и он будет ждущим):

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Откроем новый терминал, подкючися к jumpbox и дадим команду:

```bash
curl --head http://127.0.0.1:8080
```

Вывод должен быть примерно таким:

```
HTTP/1.1 200 OK
Server: nginx/1.31.1
Date: Fri, 12 Jun 2026 21:17:38 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Fri, 22 May 2026 12:50:47 GMT
Connection: keep-alive
ETag: "6a105127-380"
Accept-Ranges: bytes
```

А в терминале где мы запустили порт форвардинг должна появиться ещё одна строка:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

Нажмите в перовм терминале  `Ctrl+c` чтобы выйти.

# Logs

Здесь мы убедимся что можем получить [логи из контейнеров](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

```bash
kubectl logs $POD_NAME
```

Вы должны увидеть что-то вроде такого:

```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/06/12 21:11:44 [notice] 1#1: using the "epoll" event method
2026/06/12 21:11:44 [notice] 1#1: nginx/1.31.1
2026/06/12 21:11:44 [notice] 1#1: built by gcc 14.2.0 (Debian 14.2.0-19) 
2026/06/12 21:11:44 [notice] 1#1: OS: Linux 6.12.90+deb13.1-amd64
2026/06/12 21:11:44 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2026/06/12 21:11:44 [notice] 1#1: start worker processes
2026/06/12 21:11:44 [notice] 1#1: start worker process 29
127.0.0.1 - - [12/Jun/2026:21:17:38 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.14.1" "-"
```

# Exec

Теперь убедимся что можем [выполнить команды в контейнере](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Выведем версию nginx выполнив команду `nginx -v` в контейнере `nginx`:

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

Вывод примерно такой:

```
nginx version: nginx/1.31.1
```

# Services

И наконец убедимся что можем опубликовать приложение используя [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Опубликуем nginx испльзуя [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```bash
kubectl expose deployment nginx \
  --port 80 --type NodePort
```

Получим порт на ноде на которой работает nginx:

```bash
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Получим имя хоста ноды на котром запущен nginx:

```bash
NODE_NAME=$(kubectl get pods \
  -l app=nginx \
  -o jsonpath="{.items[0].spec.nodeName}")
```

Ну и проверим всё через `curl`:

```bash
curl -I http://${NODE_NAME}:${NODE_PORT}
```

Вывод должен быть примерно такой:

```
HTTP/1.1 200 OK
Server: nginx/1.31.1
Date: Fri, 12 Jun 2026 21:33:27 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Fri, 22 May 2026 12:50:47 GMT
Connection: keep-alive
ETag: "6a105127-380"
Accept-Ranges: bytes
```
