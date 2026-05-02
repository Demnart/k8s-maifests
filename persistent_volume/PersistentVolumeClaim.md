# PersistentVolumeClaim

Если ```PersistentVolume```- это ресурс в кластере, то ```PersistenceVolumeClaim```(PVC) - это запрос на использование этого ресурса со стороны пользователя или приложения. ```PVC``` позволяет запрашивать хранилище с определенными характеристиками(размер, режим доступа), не вникая в детали его реализации.  

Это разделение ролей - ключевая идея персистеного хранения в Kubernetes:  

   - **Администратор** настраивает физические хранилища и предоставляет их в виде ```PV```.  
   - **Клиент(разработчик)** - запращивает необходимые объемы хранения для своих приложений с помощью ```PVC```.  

 # Механизм связывания PV и PVC  

 Когда клиент создает ```PVC```, Kubernetes ищет подходящий ```PV```, который может удовлетворить этот запрос. Для того, чтобы ```PV``` и ```PVC``` были связаны, должны совпасть следующие параметры:  

   - **accessModes** - ```PV``` должен поддерживать режим доступа, запрощенный в ```PVC```.  
   - **storage** - Размер ```PV``` должен быть не меньше, чем размер запрощенный в ```PVC```.  
   - **storageClassName** - ```PVC``` и ```PV``` должны принадлежать одному и тому же ```StorageClass```. Если ```storageClassName``` не указан, то используется ```PV``` без ```StorageClass```.  
   -  **selector** - ```PVC``` может использовать ```matchLabels``` для выбора ```PV``` с определенными метками.  

 Как только Kubernetes находит подходящий ```PV```, он связывает их , и оба объекта переходят в статус ```Bound```. После этого ```PVC``` можно использовать в поде как обычный том.  

 ## Создание PVC и его использование в поде.

 Продолжим наш предыдущий пример. У нас есть ```PV``` ```nfs-pv-01``` в статусе ```Available```. Теперь создадим ```PVC```, чтобы запросить это хранилище. 
 A

 Файл ```nfs-pvc.yml```:  
 
 ```yml
 ---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-01
  labels:
    app.kubernetes.io/name: app-config
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  # Запрашиваем те же режимы доступа, что и у нашего PV.
  accessModes:
    - ReadWriteMany
  # Запращиваем 500 мегабайт. Это меньше, чем 1 Gi, доступный в PV, поэтому он походит.
  resources:
    requests:
      storage: 500Mi
  selector:
    matchLabels:
      app.kubernetes.io/instance: manual

```  

Применим манифест:  

```sh
kubectl -n work apply -f nfs-pvc.yml
```  
```sh
persistentvolumeclaim/nfs-pvc-01 created
```

Теперь посмотрим на статусы PV и PVC:  

```sh
kubectl get pv,pvc -n work
``` 
```sh
NAME                         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/nfs-pv-01   1Gi        RWX            Retain           Bound    work/nfs-pvc-01                  <unset>                          23m

NAME                               STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/nfs-pvc-01   Bound    nfs-pv-01   1Gi        RWX                           <unset>                 26s
```  

## Использование PVC в поде  

Чтобы использовать хранилище, в манифисте необходимо определить том, указав в качестве его источника ```persistentVolumeClaim```.  

Файл ```pod-with-pvc.yml```:  

```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: app-writer
  labels:
    app.kubernetes.io/name: app-writer
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  volumes:
  # Определяем том и указываем, что он должен использовать PVC
  - name: nfs-storage
    persistentVolumeClaim:
      claimName: nfs-pvc-01
  containers:
  - name: writer
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c"]
    args:  
    - |
      while true; do
        echo "Log entry at $(date)" >> /data/app.log;
        sleep 5;
      done;
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    volumeMounts:
    - name: nfs-storage
      mountPath: /data

```

Запустим под:  

```sh
kubectl -n work apply -f pod-with-pvc.yml 
```
```sh
pod/app-writer created
```  

Убедимся, что под работает и проверим, чтоданные были записаны в файл:  

```sh
kubectl -n work exec app-writer -- cat /data/app.log
```
```sh
Log entry at Sat May  2 13:19:42 UTC 2026
```  

Данные успешено записаны в персистентное хранилище. Даже если мы удалим под ```app-writer``` и создадим новый, который будет читать из этого же ```PVC```, он увидит этот файл. Это и есть основное преимущество ```PersistentVolume```.  

Не забудьте удалить созданные ресурсы, чтобы они не мешали  

```sh
kubectl -n work delete po app-writer --force
kubectl -n work delete pvc nfs-pvc-01
```  

Посмотрим состояние PV:  

```sh
kubectl get pv
```
```sh
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
nfs-pv-01   1Gi        RWX            Retain           Released   work/nfs-pvc-01                  <unset>                          38m
```  

Поле ```STATUS: Released```. Несмотря на то, что этот ЗМ продолжает находится в списсках(он не удален), пользоваться им нельзя. Посмотрим, что будет если повторно использовать манифест PVC, который использовали до этого:  

```sh
kubectl -n work apply -f nfs-pvc.yml 
```
```sh
persistentvolumeclaim/nfs-pvc-01 created
```
```sh
kubectl -n work get pvc
```
```sh
NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nfs-pvc-01   Pending                                                     <unset>                 33s
```

Как видим у ```PVC``` статус ```Pending``` т.е. наш ```PVC``` не может использовать ранее созданный ```PV``` и будет ждать когда в кластере появится ```PV``` под его требования.

Удалим не нужные PVC и PV:

```sh
kubectl -n work delete pvc nfs-pvc-01
kubectl delete pv nfs-pv-01
```

**Важно!!!** Удаление ```PV``` не означает, что данные которые были записаны в этот PV будут удалены. Т.е. в зависимости от используемого типа хранилища, может возникнуть ситуация при которой ```PV``` уже удален, а физическое хранилище и данные в нем остались. 