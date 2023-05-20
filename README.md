# devops-DZ-12.7-K8S-storage2
# Домашнее задание к занятию «Хранение в K8s. Часть 2»

### Цель задания

В тестовой среде Kubernetes нужно обеспечить обмен файлами между контейнерам пода и доступ к логам ноды.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

# Домашнее задание к занятию «Хранение в K8s. Часть 2»

### Цель задания

В тестовой среде Kubernetes нужно создать PV и продемострировать запись и хранение файлов.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке NFS в MicroK8S](https://microk8s.io/docs/nfs).
2. [Описание Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
3. [Описание динамического провижининга](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/).
4. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1

**Что нужно сделать**

Создать Deployment приложения, использующего локальный PV, созданный вручную.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории.
4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему.
5. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

------

### Задание 2

**Что нужно сделать**

Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.

1. Включить и настроить NFS-сервер на MicroK8S.
2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS.
3. Продемонстрировать возможность чтения и записи файла изнутри пода.
4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

------

## Ответ

Проверим установку _MicroK8S_ и необходимые плагины

```bash
root@vkvm:/home/vk# microk8s enable dashboard
Infer repository core for addon dashboard
Addon core/dashboard is already enabled
root@vkvm:/home/vk# microk8s enable dns
Infer repository core for addon dns
Addon core/dns is already enabled
root@vkvm:/home/vk# microk8s enable ingress
Infer repository core for addon ingress
Addon core/ingress is already enabled
root@vkvm:/home/vk# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
vkvm   Ready    <none>   43d   v1.26.3
```

## Задание 1

### 1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool

### 2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде

Создадим папку для тома на локальном хосте

```bash
root@vkvm:/home/vk# mkdir -p /i_am_node/pv1
```

Включим поддержку _PersistentVolume_

```bash
root@vkvm:/home/vk# microk8s enable hostpath-storage
Infer repository core for addon hostpath-storage
Enabling default storage class.
WARNING: Hostpath storage is not suitable for production environments.

deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon.
```

Создадим `pv1.yml` с развёртыванием постоянного тома

<details><summary>Посмотреть YAML...</summary>

```yml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
  labels:
    app: pv1
  namespace: default
spec:
  storageClassName: storageclass1
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: "/i_am_node/pv1"
  persistentVolumeReclaimPolicy: Delete
```

</details>

[Ссылка на  pv1.yml](pv1.yml)

Создадим файл `pvc1.yml` с развёртыванием заявки на том

<details><summary>Посмотреть YAML...</summary>

```yml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  labels:
    app: pvc1
  namespace: default
spec:
  storageClassName: storageclass1
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

</details>

[Ссылка на pvc1.yml](pvc1.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk# kubectl apply -f pv1.yml -f pvc1.yml
persistentvolume/pv1 created
persistentvolumeclaim/pvc1 created
```

Проверим состояние томов и заявок

```bash
root@vkvm:/home/vk# kubectl get persistentvolume
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS    REASON   AGE
pv1    1Gi        RWO            Delete           Bound    default/pvc1   storageclass1            35s
root@vkvm:/home/vk# kubectl get persistentvolumeclaim
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc1   Bound    pv1      1Gi        RWO            storageclass1   38s
```

Создадим `deploy1.yml` с развёртыванием двух контейнеров с общим томом

<details><summary>Посмотреть YAML...</summary>

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy1
  labels:
    app: deploy1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy1
  template:
    metadata:
      labels:
        app: deploy1
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ['sh', '-c', 'сnt=1; datafile="/out/datafile.txt"; while true; do echo "$((cnt++)) | $(date)" >> $datafile; sleep 5; done']
          volumeMounts:
            - name: vol1
              mountPath: /out
        - name: multitool
          image: wbitt/network-multitool
          volumeMounts:
            - name: vol1
              mountPath: /in
      volumes:
        - name: vol1
          persistentVolumeClaim:
            claimName: pvc1
```

</details>

[Ссылка на deploy1.yml](deploy1.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk# kubectl apply -f deploy1.yml
deployment.apps/deploy1 created
```

Проверим состояние подов

```bash
root@vkvm:/home/vk# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
deploy1-68bf75d944-bztst   2/2     Running   0          14s
root@vkvm:/home/vk# kubectl describe pod deploy1-68bf75d944-bztst | grep -iE '(Mounts|Volumes)' -A2
    Mounts:
      /out from vol1 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9cx65 (ro)
--
    Mounts:
      /in from vol1 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9cx65 (ro)
--
Volumes:
  vol1:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
```

### 3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории

Проверим чтение файла из контейнера

```bash
root@vkvm:/home/vk# kubectl exec -it deploy1-68bf75d944-bztst -c multitool -- cat /in/datafile.txt
0 | Sat May 20 06:09:20 UTC 2023
1 | Sat May 20 06:09:25 UTC 2023
2 | Sat May 20 06:09:30 UTC 2023
3 | Sat May 20 06:09:35 UTC 2023
4 | Sat May 20 06:09:40 UTC 2023
5 | Sat May 20 06:09:45 UTC 2023
6 | Sat May 20 06:09:50 UTC 2023
7 | Sat May 20 06:09:55 UTC 2023
8 | Sat May 20 06:10:00 UTC 2023
9 | Sat May 20 06:10:05 UTC 2023
10 | Sat May 20 06:10:10 UTC 2023
11 | Sat May 20 06:10:15 UTC 2023
12 | Sat May 20 06:10:20 UTC 2023
13 | Sat May 20 06:10:25 UTC 2023
14 | Sat May 20 06:10:30 UTC 2023
15 | Sat May 20 06:10:35 UTC 2023
16 | Sat May 20 06:10:40 UTC 2023
```

Убедились, что **файл, записанный в одном контейнере, доступен на чтение из другого контейнера**

### 4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему

Удалим развёрнутый deployment и persistent volume claim

```bash
root@vkvm:/home/vk# kubectl delete -f deploy1.yml -f pvc1.yml
deployment.apps "deploy1" deleted
persistentvolumeclaim "pvc1" deleted
```

Проверим состояние persistent volume

```bash
kubectl get persistentvolume
root@vkvm:/home/vk# kubectl get persistentvolume
root@vkvm:/home/vk# kubectl describe persistentvolume | grep -i Events -A5
Events:
  Type     Reason              Age   From                         Message
  ----     ------              ----  ----                         -------
  Warning  VolumeFailedDelete  69s   persistentvolume-controller  host_path deleter only supports /tmp/.+ but received provided /i_am_node/pv1
```

Убедились, что **том остался, но статус тома поменялся на _Failed_**

### 5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему

Проверим файл на локальном диске ноды

```bash
root@vkvm:/home/vk# tail -10 /i_am_node/pv1/datafile.txt
27 | Sat May 20 06:11:35 UTC 2023
28 | Sat May 20 06:11:40 UTC 2023
29 | Sat May 20 06:11:45 UTC 2023
30 | Sat May 20 06:11:50 UTC 2023
31 | Sat May 20 06:11:55 UTC 2023
32 | Sat May 20 06:12:00 UTC 2023
33 | Sat May 20 06:12:05 UTC 2023
34 | Sat May 20 06:12:10 UTC 2023
35 | Sat May 20 06:12:15 UTC 2023
36 | Sat May 20 06:12:20 UTC 2023
```

Убедились, что **файл сохранился на локальном диске ноды**

Удалим развёрнутый persistent volume

```bash
root@vkvm:/home/vk# kubectl delete -f pv1.yml
persistentvolume "pv1" deleted
```

Проверим файл на локальном хосте после удаления persistent volume

```bash
root@vkvm:/home/vk# ls -l /i_am_node/pv1/datafile.txt 
-rw-r--r-- 1 root root 1248 мая 20 09:12 /i_am_node/pv1/datafile.txt
```

Убедились, что **файл остался на локальном хосте после удаления persistent volume** Вероятно это произошло потому что политику возврата с типом Delete поддерживают только некоторые облачные провайдеры, например AWS EBS.

### 5. Предоставить манифесты, а также скриншоты или вывод необходимых команд

## Задание 2

### 1. Включить и настроить NFS-сервер на MicroK8S

Установим NFS сервер на локальном хосте

```bash
root@vkvm:/home/vk# apt install nfs-kernel-server
```

Создадим папку на локальном хосте для использования в качестве shared folder

```bash
root@vkvm:/home/vk# mkdir -p /i_am_node/nfs1
root@vkvm:/home/vk# chown nobody:nogroup /i_am_node/nfs1
root@vkvm:/home/vk# chmod 0777 /i_am_node/nfs1
```

Добавим папку в NFS сервер локального хоста

```bash
root@vkvm:/home/vk# cat /etc/exports 
/i_am_node/nfs1 10.0.0.0/8(rw,sync,no_subtree_check)
/i_am_node/nfs1 192.168.0.0/16(rw,sync,no_subtree_check)
```

Перезапустим NFS сервер

```bash
root@vkvm:/home/vk# systemctl restart nfs-kernel-server
```

Установим CSI драйвер для NFS с помощью Helm

```bash
root@vkvm:/home/vk# microk8s enable helm3
Infer repository core for addon helm3
Addon core/helm3 is already enabled
root@vkvm:/home/vk# microk8s helm3 repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
"csi-driver-nfs" has been added to your repositories
root@vkvm:/home/vk# microk8s helm3 repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "csi-driver-nfs" chart repository
Update Complete. ⎈Happy Helming!⎈
root@vkvm:/home/vk# microk8s helm3 install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet
NAME: csi-driver-nfs
LAST DEPLOYED: Sat May 20 09:22:38 2023
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The CSI NFS Driver is getting deployed to your cluster.

To check CSI NFS Driver pods status, please run:

  kubectl --namespace=kube-system get pods --selector="app.kubernetes.io/instance=csi-driver-nfs" --watch
```

### 2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS

Создадим `storageclass2.yml` с развёртыванием класса хранилища.

<details><summary>Посмотреть YAML...</summary>

```yml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storageclass2
  labels:
    app: storageclass2
  namespace: default
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.122
  share: /i_am_node/nfs1
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
```

</details>

[Ссылка на storageclass2.yml](storageclass2.yml)

Укажем адрес NFS сервера и путь до шары на нём.

Запустим развёртывание

```bash
root@vkvm:/home/vk# kubectl apply -f storageclass2.yml
storageclass.storage.k8s.io/storageclass2 created
```

Проверим состояние класса хранилища

```bash
root@vkvm:/home/vk# kubectl get storageclass storageclass2
NAME            PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storageclass2   nfs.csi.k8s.io   Delete          Immediate           false                  18s
```

Создадим `pvc2.yml` с развёртыванием заявки на том

<details><summary>Посмотреть YAML...</summary>

```yml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
  labels:
    app: pvc2
  namespace: default
spec:
  storageClassName: storageclass2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

</details>

[Ссылка на pvc2.yml](pvc2.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk# kubectl apply -f pvc2.yml
persistentvolumeclaim/pvc2 created
````

Проверим состояние томов и их заявок

```bash
root@vkvm:/home/vk# kubectl get persistentvolume,persistentvolumeclaim
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS    REASON   AGE
persistentvolume/pvc-afacf83e-da0d-4451-9071-c459b1aac474   1Gi        RWO            Delete           Bound    default/pvc2   storageclass2            107s

NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
persistentvolumeclaim/pvc2   Bound    pvc-afacf83e-da0d-4451-9071-c459b1aac474   1Gi        RWO            storageclass2   108s

root@vkvm:/home/vk# kubectl describe pvc | grep -i Events -A4
Events:
  Type    Reason                 Age                    From                                                      Message
  ----    ------                 ----                   ----                                                      -------
  Normal  Provisioning           2m13s                  nfs.csi.k8s.io_vkvm_a368cbf1-484c-4289-ada7-7da1c18a890d  External provisioner is provisioning volume for claim "default/pvc2"
  Normal  ExternalProvisioning   2m13s (x2 over 2m13s)  persistentvolume-controller                               waiting for a volume to be created, either by external provisioner "nfs.csi.k8s.io" or manually created by system administrator
```

Убедились, что **том развернулся автоматически**

Создадим `deploy2.yml` с развёртыванием контейнера с томом

<details><summary>Посмотреть YAML...</summary>

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy2
  labels:
    app: deploy2
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy2
  template:
    metadata:
      labels:
        app: deploy2
    spec:
      containers:
        - name: multitool
          image: wbitt/network-multitool
          volumeMounts:
            - name: volume-2
              mountPath: /input
      volumes:
        - name: volume-2
          persistentVolumeClaim:
            claimName: pvc2
```

</details>

[Ссылка на deploy2.yml](deploy2.yml)

Запустим развёртывание  

```bash
root@vkvm:/home/vk# kubectl apply -f deploy2.yml
deployment.apps/deploy2 created
```

Проверим состояние подов

```bash
root@vkvm:/home/vk# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
deploy2-f9f757b77-jnwpb   1/1     Running   0          24s
root@vkvm:/home/vk# kubectl describe pod deploy2-f9f757b77-jnwpb | grep -iE '(Mounts|Volumes)' -A2
    Mounts:
      /input from volume-2 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9b226 (ro)
--
Volumes:
  volume-2:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
```

### 3. Продемонстрировать возможность чтения и записи файла изнутри пода

Проверим запись и чтение файла из контейнера

```bash
root@vkvm:/home/vk# kubectl exec -it deploy2-f9f757b77-jnwpb -c multitool -- sh -c "echo 'Hello from VK' > /input/output.txt"
root@vkvm:/home/vk# kubectl exec -it deploy2-f9f757b77-jnwpb -c multitool -- cat /input/output.txt
Hello from VK
```

Убедились, что **файл доступен на запись и чтение из пода**

Проверим файл на локальном диске ноды

```bash
root@vkvm:/home/vk# tail /i_am_node/nfs1/pvc-afacf83e-da0d-4451-9071-c459b1aac474/output.txt 
Hello from VK
```

Убедились, что **файл сохранился на локальном диске ноды**

### 4. Предоставить манифесты, а также скриншоты или вывод необходимых команд

Удалим развёрнутые deployment, persistent volume claim и storage class

```bash
root@vkvm:/home/vk# kubectl delete -f deploy2.yml -f pvc2.yml -f storageclass2.yml
deployment.apps "deploy2" deleted
persistentvolumeclaim "pvc2" deleted
storageclass.storage.k8s.io "storageclass2" deleted
```

Проверим состояние persistent volume

```bash
root@vkvm:/home/vk# kubectl get persistentvolume
No resources found
```

Убедились, что **том удалился автоматически (потому что мы удалили заявку на том, по которой он был создан)**

Проверим файл на локальном диске после удаления persistent volume

```bash
root@vkvm:/home/vk# ls -l /i_am_node/nfs1/
total 0
```

Убадились, что **данные на локальном хосте удалились вместе с томом**

Вывод:

a) При удалении заявки на том с hostPath -  файл остаётся на локальном хосте
b) При удалении заявки на том, автоматически созданный по заявке - файл удаляется на локальном хосте
