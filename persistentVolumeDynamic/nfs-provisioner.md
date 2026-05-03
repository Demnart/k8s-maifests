# NFS provisioner  

[NFS provisioner](#nfs-provisioner)  
[ServiceAccount](#serviceaccount)  
[RBAC](#rbac)  
[NFS-provisioner deployment](#nfs-provisioner-deployment)  
[Установка](#установка)

Для работы с NFS  в кластере нам понадобится контроллер, позволяющий автоматически создавать тома на NFS сервере при создании ```PVC```. В качестве контроллера будем использовать проект ["kubernetes nfs subdir external provisioner"](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner).  

**Если в кластере Kubernetes уже есть контроллер**, который может создавать тома на различных блочных устройствах, то для выполнения примеров в следующих частях этой главы можно использовать его и не устанавливать - ```nfs-provisioner```. Главное точно знать его ```provisioner name```, чтобы в дальнейшем использовать его в ```StorageClass```. Можете сразу переходить к следующему разделу.  

Если такого контроллера нет, то вы:  

  1. Должны быть администратором кластера Kubernetes.
  2. Иметь "под рукой" NFS сервер, которым вы можете пользоваться.  
  3. Установить ```nfs-provisioner``` в кластер.  


На данном этапе мы не будем углубляться в подробности работы ```nfs-provisioner```, а просто создадим необходимые ресурсы. В дальнейшем мы разберем похожие технологи и манифесты подробнее. 

## ServiceAccount  

Создадим ```ServiceAccount``` для ```nfs-provisioner```, файл ```nfs/sa.yml```:  

```yml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
```  

## RBAC  

Манифест RBAC для ```nfs-provisioner``` файл ```nfs/rbac.yml```:  

```yml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]Установка
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io

```

## NFS-provisioner deployment.  

Для работы nfs-provisioner необходимо иметь рабочий NFS-сервер. Соответственно вы должны знать его IP адрес и путь к общей папке. ТАк же необходимо определить имя nfs-provisioner, которое будет использоваться в ```StorageClass```. Имя provisioner может быть любым, но желательно, чтобы оно было понятным и естественно уникальным. 

В моем случае используется NFS-сервер с такими параметрами:  

  - **IP** - 192.168.122.193  
  - **PATH** - /volume1/nfs  
  - **Provisioner name** - artiom.local/nfs - любая ASCII строка

Создадим манифест ```Deployment``` для ```nfs-provisioner``` файл ```deployment-provisioner.yml```:  

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: work
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: gcr.io/k8s-staging-sig-storage/nfs-subdir-external-provisioner@sha256:48f7d851c26c1340ed202d0311d36da007c857ea24d23bbf0ba554c471bd0870
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: artiom.local/nfs
            - name: NFS_SERVER
              value: 192.168.122.193
            - name: NFS_PATH
              value: /volume1/nfs
          resources:
            requests:
              cpu: 10m
              memory: 10Mi
            limits:
              cpu: 100m
              memory: 100Mi
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.122.193
            path: /volume1/nfs

```

Особенность этого контроллера в том, что он для каждого PV создает отдельную папку на NFS сервере. Поэтому в манифесте мы указываем путь к общей папке.  

## Установка  

На всех машинах кластера должны быть установлены пакеты ```nfs-common``` или ```nfs-utils```. Зависит от используемого дистрибутива.  

Создадим ресурсы в кластере:  

```sh
kubectl create ns nfs-provisioner
kubectl -n nfs-provisioner apply -f nfs/
```  

Проверяем что все ресурсы созданы:

```sh
kubectl get all -n nfs-provisioner
```
```sh
NAME                                          READY   STATUS    RESTARTS   AGE
pod/nfs-client-provisioner-6575cbb7d9-s2rwx   1/1     Running   0          3m25s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner   1/1     1            1           3m26s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-client-provisioner-6575cbb7d9   1         1         1       3m26s
```  

