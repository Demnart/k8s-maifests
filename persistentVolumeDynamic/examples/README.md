# Depoyment, PVC, PV and StorageClass  

После того как мы рассмотрели каждый компонент по отдельности, давайте разберём примеры использования ```PV``` и ```PVC```.  

## Использование PVC в Deployment.  

В манифесте ```Deployment``` с одним подом, который будет каждые 5 секунд, дописывать текущую дату в файл, находящийся в персистентном хранилище.  

Файл ```final-deployment.yml```:

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateful-app
  labels: &labels
    app.kuberntes.io/name: stateful-app
    app.kuberntes.io/instance: test
    app.kuberntes.io/version: v0.0.1
spec:
  replicas: 1
  selector:
    matchLabels: *labels
  template:
    metadata:
      labels: *labels
    spec:
      # Том, который будет использоваться контейнером
      # Его источник PVC с именем 'app-data-pvc'
      volumes:
      - name: app-data
        persistentVolumeClaim:
          claimName: app-data-pvc
      containers:
      - name: main
        image: busybox:1.37
        command: ["/bin/sh", "-c"]
        args:
        - |
          while true; do
            echo "Log entry $(date)" >> /data/app.log;
            sleep 5;
          done
        resources:
          limits:
            memory: "100Mi"
            cpu: "250m"
          requests:
            memory: "32Mi"
            cpu: "100m"
        volumeMounts:
          - name: app-data
            mountPath: /data

---
# PVC, который будет запрашивать хранилище.
# Он будет создан вместе с Deployment.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  labels:
    app.kuberntes.io/name: app-data-pvc
    app.kuberntes.io/instance: test
    app.kuberntes.io/version: v0.0.1
spec:
  resources:
    requests:
      storage: 256Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  # Указываем StorageClass, что мы создали ранее.
  storageClassName: nfs-client


```

В одном манифисте мы определяем и ```Deployment``` и ```PersistentVolumeClaim```.

Применим манифест:

```sh
kubectl -n work apply -f final-deployment.yml 
```
```sh
deployment.apps/stateful-app created
persistentvolumeclaim/app-data-pvc created
```  

Проверим создан ли ```PV```:

```sh
kubectl -n work get pvc,pv
```
```sh
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/app-data-pvc   Bound    pvc-196c6d76-c9c5-4f93-8c95-7cac973a5343   256Mi      RWX            nfs-client     <unset>                 38s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-196c6d76-c9c5-4f93-8c95-7cac973a5343   256Mi      RWX            Delete           Bound    work/app-data-pvc   nfs-client     <unset>                          38s
```

Проверим созданный ```Deployment```:  

```sh
kubectl -n work get all
```
```sh
NAME                                READY   STATUS    RESTARTS   AGE
pod/stateful-app-6cb64d6959-7x6xg   1/1     Running   0          108s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stateful-app   1/1     1            1           108s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stateful-app-6cb64d6959   1         1         1       108s
```  

Проверим логи:  
```sh
#Сначала получаем имя пода
POD_NAME=$(kubectl -n work get pods -l app.kuberntes.io/name=stateful-app -o jsonpath='{.items[0].metadata.name}')
#Смотрим содержимое файла
kubectl -n work exec -it $POD_NAME -- cat /data/app.log
```
```sh
Log entry Sun May  3 13:25:00 UTC 2026
Log entry Sun May  3 13:25:05 UTC 2026
Log entry Sun May  3 13:25:10 UTC 2026
Log entry Sun May  3 13:25:15 UTC 2026
Log entry Sun May  3 13:25:20 UTC 2026
```  

Теперь удалим ```Deployment```, ```PVC```, чтобы посмотреть как сработает ```Reclaim Policy```:  

```sh
kubectl -n work delete -f final-deployment.yml --force
```
```sh
Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
deployment.apps "stateful-app" force deleted
persistentvolumeclaim "app-data-pvc" force deleted
```

Подождав секунд 30(из-за grace-period(тк у нас был бесконечный цикл while)), проверим, что ```PV``` был удален:  

```sh
kubectl get pv
```  

Вывод будет пустым:  

```sh
No resources found
```  


## Deployment с несколькими репликами и общим PVC  

Рассмотрим, что происходит, когда ```Deployment``` маштабируется до нескольких реплик, использующих один и тот же ```PVC```.  

Файл ```deployment-2-replicas.yml```:  
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-replica-app
  labels: &labels
    app.kuberntes.io/name: multi-replica-app
    app.kuberntes.io/instance: test
spec:
  replicas: 2
  selector:
    matchLabels: *labels
  template:
    metadata:
      labels: *labels
    spec:
      # Том, который будет использоваться контейнером
      # Его источник PVC с именем 'shared-data-pvc'
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: shared-data-pvc
      containers:
      - name: main
        image: busybox:1.37
        command: ["/bin/sh", "-c"]
        args:
        - |
          while true; do
            echo "[$(date '+%H:%M:%S')] Pod: $HOSTNAME" >> /data/shared.log;
            sleep 3;
          done
        resources:
          limits:
            memory: "100Mi"
            cpu: "250m"
          requests:
            memory: "32Mi"
            cpu: "100m"
        volumeMounts:
          - name: shared-data
            mountPath: /data

---
# PVC, который будет запрашивать хранилище.
# Он будет создан вместе с Deployment.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
  labels:
    app.kuberntes.io/name: shared-data-pvc
    app.kuberntes.io/instance: test
spec:
  resources:
    requests:
      storage: 256Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  # Указываем StorageClass, что мы создали ранее.
  storageClassName: nfs-client

```

Применим манифест:

```sh
kubectl -n work apply -f deployment-2-replicas.yml
```
```sh
deployment.apps/multi-replica-app created
persistentvolumeclaim/shared-data-pvc created
```

Проверим состояние ресурсов:  

```sh
kubectl -n work get deployment,pod,pvc -l app.kubernetes.io/name=multi-replica-app
```
```sh
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/multi-replica-app   2/2     2            2           2m33s

NAME                                     READY   STATUS    RESTARTS   AGE
pod/multi-replica-app-55dd4c44df-b9gtv   1/1     Running   0          2m33s
pod/multi-replica-app-55dd4c44df-vhnng   1/1     Running   0          2m33s
```
```sh
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
shared-data-pvc   Bound    pvc-c12877a3-f6d8-4224-8ce2-24cc17dfc61d   256Mi      RWX            nfs-client     <unset>  
```

Оба пода запущены и используют один ```PVC```. Посмотрим что пишеться в лог:  
```sh
 kubectl -n work exec $(kubectl -n work get pods -l app.kuberntes.io/name=multi-replica-app -o jsonpath='{.items[0].metadata.nam
e}') -- cat /data/shared.log
```
```sh
[13:44:29] Pod: multi-replica-app-55dd4c44df-vhnng
[13:44:29] Pod: multi-replica-app-55dd4c44df-b9gtv
[13:44:32] Pod: multi-replica-app-55dd4c44df-vhnng
[13:44:33] Pod: multi-replica-app-55dd4c44df-b9gtv
[13:44:35] Pod: multi-replica-app-55dd4c44df-vhnng
[13:44:36] Pod: multi-replica-app-55dd4c44df-b9gtv
[13:44:39] Pod: multi-replica-app-55dd4c44df-vhnng
```  

### Проблема нестабильной идентичности  

Удалим один из подов и посмотрим, что произойдет:  

```sh
kubectl -n work delete pod multi-replica-app-55dd4c44df-b9gtv --force --grace-period=0
```
```sh
pod "multi-replica-app-55dd4c44df-b9gtv" deleted
```

Проверим поды после пересоздания:

```sh
kubectl -n work get pods -l app.kuberntes.io/name=multi-replica-app
```
```sh
AME                                 READY   STATUS    RESTARTS   AGE
multi-replica-app-55dd4c44df-q885h   1/1     Running   0          4s
multi-replica-app-55dd4c44df-vhnng   1/1     Running   0          9m34s
```

```Deployment``` создал новый под с другим именем. Посмотрим лог:

```sh
 kubectl -n work exec $(kubectl -n work get pods -l app.kuberntes.io/name=multi-replica-app -o jsonpath='{.items[0].metadata.name}') -- tail -10 /data/shared.log
```

Новый под ```q885h``` продолжил писать в тот же файл, но с новым именем.

## Проблемы испольлзования Deployment с PVC  

При использовании ```Deployment``` c персистентым хранилидем возникают следующие проблемы:

  1. **Общее хранилище для всех реплик** - все поды используют один и тот же ```PVC```. Если  приложение может работать только с одним пользователем, то могут возникнуть повреждение данных.  
  2. **Нестабильная идентичность** - при пересоздании пода он получает случайное имя. Допустим приложение 1 обращается к файлу 1, а приложение 2 обращается к файлу 2. Если удалить под, то при создании нового пода ему дастся новое имя и будет не понятно какое приложение должно обращаться к какому файлу.  
  3. **Ограничения ReadWriteOnce** - если ```StorageClass``` поддерживает только ```ReadWriteOnce```(например не поддерживается одновременная запись со стороны многих клиентов), то при использовании Deployment только 1 под получит доступ к тому, остальные поды не смогут стартануть из-за ошибки доступа к тому.  
  4. **Конкурентный доступ к файлам** - даже с ```ReadWriteMany``` одновременная запись в один файл. Если это не сетевое хранилище(как у нас NFS), могут возникнуть ошибки.  

## StatefulSet и volumeClaimTemplates  

Для приложений, требующих стабильных сетевых идентификаторов и уникального, постоянного хранилища для каждой реплики(например, база данных), ```Deployment``` подходит далеко не всегда. Для таких задач обычно используют ```StatefulSet```.  

Одной из ключевых особенностей ```StatefulSet``` является возожмность автоматически создавать ```PersistentVolumeClaim``` для каждой реплики с помощью шаблона - ```volumeClaimTemplates```.  

## VolumeClaimTemplates  

Когда вы определяете ```VolumeClaimTemplates``` в манифесте ```StatefulSet```:  
  1. Для каждой реплики ```StatefulSet```(например, ```app-0```,```app-1```) автоматически создается **свой** ```PVC```.  
  2. Имя ```PVC``` формируется по шаблону: ```<имя-шаблона>-<имя-пода>```. Например, если имя шаблона ```www```, а имя пода ```app-0```, то ```PVC``` будет называться ```www-app-0```.  
  3. В каждом ```PVC``` определяется ```StorageClass```, что приводит к динамическому созданию уникального ```PV``` для каждой реплики.  
  4. Созданный ```PVC``` автоматически монтируется к соответствующему поду.  

Это гарантирует, что каждая реплика ```StatefulSet``` получит свой собственный, изолированный том, который будет "следовать" за ней при перезапусках и перемещениях.  

### StatefulSet с двумя репликами

Создадим ```StatefulSet``` с двумя репликами. Каждая реплика будет записывать свое имя хоста в файл, демонстрируя, что у каждой из них свой собственный том.  

Файл ```statefulset-pvc.yml```
```yml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  labels: &labels
    app.kuberntes.io/name: web
    app.kuberntes.io/instance: test
    app.kuberntes.io/version: v0.0.1
spec:
  selector:
    matchLabels: *labels
  serviceName: nginx
  replicas: 2
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: app
          image: nginx:1.27
          command: ["/bin/sh", "-c"]
          args:
            - |
              # Записываем имя хоста в index.html
              echo "Hostname: $(hostname)" > /usr/share/nginx/html/index.html;
              #Запускаем nginx
              nginx -g 'daemon off;';
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  # Шаблон для автоматического создания PVC  
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName: "nfs-client"
      resources:
        requests:
          storage: 1Gi

---
#Headless Service, необходимый для StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels: &labels
    app.kuberntes.io/name: web
    app.kuberntes.io/instance: test
    app.kuberntes.io/version: v0.0.1  
spec:
  selector: *labels
  ports:
  - port: 80
    targetPort: web
  clusterIP: None

```

Применим манифест:  

```sh
kubectl -n work apply -f  statefulset-pvc.yml
```
```sh
statefulset.apps/web created
service/nginx created
```  

Подождем несколько секунд и проверим созданные ресурсы:  

```sh
kubectl -n work get statefulset,pod,pvc
```
```sh
NAME                   READY   AGE
statefulset.apps/web   2/2     53s

NAME        READY   STATUS    RESTARTS   AGE
pod/web-0   1/1     Running   0          53s
pod/web-1   1/1     Running   0          51s

NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/www-web-0   Bound    pvc-aba5e2e3-07d8-450d-b7f5-d6b660b27ea4   1Gi        RWX            nfs-client     <unset>                 18m
persistentvolumeclaim/www-web-1   Bound    pvc-a282930c-fc86-4692-b506-aa04b16450dd   1Gi        RWX            nfs-client     <unset>                 18m
```  

Как и ожидалось, были созданы два пода(```web-0```,```web-1```) и два ```PVC```(```www-web-0```,```www-web-1```), каждый из которых связан со своим ```PV```.  

Теперь проверим содержимое ```index.html``` в каждом поде, чтобы убедиться, что они пишут в разные тома.  

Для ```web-0```:  

```sh
kubectl -n work exec web-0 -- cat /usr/share/nginx/html/index.html
```
```sh
Hostname: web-0
```

Для ```web-1```:

```sh
kubectl -n work exec web-1 -- cat /usr/share/nginx/html/index.html
```
```sh
Hostname: web-1
```

Это подтверждает, что каждая реплика ```StatefulSet``` получила свой собственный постоянный том.  

## Очистка ресурсов  

Удалим созданные ресурсы: 

```sh
# Удаление StatefulSet и Service
kubectl -n work delete -f statefulset-pvc.yml 

#Удаление PVC, созданных StatefulSet (они не удаляются автоматически вместе со StatefulSet)
kubectl -n work delete pvc www-web-0 www-web-1
```