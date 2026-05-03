# StorageClass  

[StorageClass ](#storageclass)  
[Ключевые компоненты StorageClass](#ключевые-компоненты-storageclass)  
[Предварительные требования](#предварительные-требования)  
[Создание StorageClass](#создание-storageclass)  
[Создание PersisitentVolumeClaim](#создание-persisitentvolumeclaim)  


Ручное создание ```PersisitentVolume```(PV) для каждого запроса(```PersistentVolumeClaim```) может быть утомительным, особенно в больших и динамических окружениях. Чтобы автоматизировать процесс, в Kubernetes существует объект ```StorageClass```.  

```StorageClass``` описывает "класс" хранилища, который вы предоставляете. Разные классы могут соответствовать разным уровням качества обслуживания (QoS), политикам резервного копирования или произвольным политикам, определяемым администратором кластера.  

Ключевая роль ```StorageClass``` - обеспечение **динамического выдения хранилища**(Dynamic Provisioning). Когда ```PVC``` запрашивает определенный ```StorageClass```, система автоматически  вызывает соответсвующий ```provisioner```, **приложение** которое создает ```PV``` "на лету". Вся "магия" ```StorageClass``` опирается на такие приложения.  

## Ключевые компоненты StorageClass  

  - **provisioner** - Указывает, какой плагин для работы с томами должне использоваться для создания ```PV```. Например, ```kuberntese.io/aws-ebs``` для mazon EBS или как в нашем случае ```artiom.local/nfs```  
  - **parametrs** - Параметры, специфичные для конкретного ```provisioner```. Например для ```aws-ebs``` это может быть тип диска (```gp2```,```io1```), а для NFS - путь к серверу.  
  - **reclaimPolicy** -  политика возврата(```Delete``` или ```Retain```), которая будет установлена для ```PV```, созданных с помощью этого класса. По умолчанию использутеся ```Delete```, что означает, что при удалении ```PVC``` будет удален и ```PV```, и физический том.  
  - **allowVolumeExpansion** - Если установлено в ```true```, позволяет увеличивать размер тома после его создания.  
  - **mountOptions** - Опции монтирования, которые будут применены к ```PV```.  

### Предварительные требования  

Для нашего примера мы будем использовать ```Kubernetes NFS Subdir External Provisioner```, который мы установили в кластере в предыдущем разделе.  

Особенностью данного приложения является конфигурационный параметр ```PROVISIONER_NAME``` при помощи которого указывается имя provisioner  при помощи которого можно ссылаться на этот контроллер в ```StorageClass```. Но это особенность именно этого приложения. Другие контроллеры могут иметь другие конфигурационные параметры.  

**Если вы планируете запусткать примеры в кластере, который вы не можете адмнистрировать**, создать свой ```StorageClass``` вы скорее всего не сможете. Уточните у администратора кластера имя ```StorageClass``` который вы можете использовать в этом кластере. Манифест из файла примера ```nfs-storageclass.yml``` применять не надо. В остальных примерах замените ```StorageClass``` используемый в них на ```StorageClass``` предоставленный администратором кластера.  

## Создание StorageClass  

Создадим ```StorageClass```, который будет использовать этот ```provisioner```.  

***Напомню***, что ```StorageClass``` создает администратор кластера.  

Файл ```nfs-storageclass.yml```:  

```yml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
  labels:
    app.kuberntes.io/name: app-config
    app.kuberntes.io/instance: test
    app.kuberntes.io/version: v0.0.1
# Этот provisioner должен соответствовать имени, с которым он зарегистрирован в кластере.
provisioner: artiom.local/nfs
# Параметры, передаваемые provisioner'у
parameters:
  onDelete: delete

```  

Применим манифест:  

```sh
kubectl apply -f nfs-storageclass.yml 
```
```sh
storageclass.storage.k8s.io/nfs-client created
```  

Посмотрим список существующих ```StorageClass```:  

```sh
NAME         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   artiom.local/nfs   Delete          Immediate           false                  104s
```  

## Создание PersisitentVolumeClaim  

Теперь, когда есть класс хранилища ```nfs-client```. Пользователь может создать ```PVC```, которые использует этот конкретный ```StorageClass```.  

Файл ```pvc-with-sc.yml```:

```yml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-dynamyc
  labels:
    app.kuberntes.io/name: app-config
    app.kuberntes.io/instance: test
    app.kuberntes.io/version: v0.0.1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi
  # Явно указываем, какой StorageClass использовать.
  storageClassName: nfs-client

```

Создадим этот ```PVC```:  

```sh
kubectl -n work apply -f pvc-with-sc.yml
```
```sh
persistentvolumeclaim/nfs-pvc-dynamyc created
```  

Проверяем созданный pvc, pv:  

```sh
kubectl -n work get pvc,pv
```
```sh
meus@meus:~/regru/k8s-maifests/persistentVolumeDynamic$ kubectl -n work get pvc,pv
NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/nfs-pvc-dynamyc   Bound    pvc-3966c11e-5c37-4655-8832-214110a62c2d   200Mi      RWX            nfs-client     <unset>                 46s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-3966c11e-5c37-4655-8832-214110a62c2d   200Mi      RWX            Delete           Bound    work/nfs-pvc-dynamyc   nfs-client     <unset>                          46s
```  

Обратите внимание на несколько моментов.  

  1. ```PVC``` ```nfs-pvc-dynamic``` сразу перешел в статус ```Bound```.  
  2. Автоматически был создан ```PV``` c длинным уникальным именем(```pvc-396....```).  
  3. Этот новый ```PV``` имеет ```RECLAIM POLICY``` равную ```DELETE``` и ```STORAGECLASS``` равный ```nfs-client```, как и было задно в ```StorageClass```.  
  4. Его размер (```200Mi```) точно соответствует запрошенному в ```PVC```.  

Это и есть "магия" динамического выделения. Клиенту достаточно создать ```PVC``` с указынным ```storageClass``` и система сама позаботиться о выделении необходимого хранилища.  

Теперь, если мы удалеим ```PVC```, то благодаря политике ```Delete```, связанный с ним ```PV``` также будет автомтически удален.  

```sh
kubectl -n work delete persistentvolumeclaim/nfs-pvc-dynamyc
```
```sh
persistentvolumeclaim "nfs-pvc-dynamyc" deleted
```

А затем проверим наши pvc,pv:  

```sh
kubectl -n work get pvc,pv
```
```sh
No resources found
```  

