

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


```