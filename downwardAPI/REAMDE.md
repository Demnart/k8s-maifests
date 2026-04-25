[DownwardAPI](#downwardapi)  
[Использование Downward API через перменные среды окружения (env)](#использование-downward-api-через-перменные-среды-окружения-env)  
[Использование Downward API через volume](#использование-downward-api-через-volume)  
[Динамическое обновление данных](#динамическое-обновление-данных)  

# DownwardAPI

Механизм Downward API позволяет приложения, рабоатающим в контейнерах, получать информацию о самих себе и своем окружении без необходимости прямого обращения к API-серверу Kubernetes. Эта информация, относящаяся к поду или контейнеру(метаданные, статус), может быть передана двумя способами: через перменные среды окружения или через файлы в ```volume```.  

Это бывает полезно, когда приложение необходимо ззнать свой IP-адрес, имя пода, namespace, в котором он запущен, или назначенные ему лимиты по ресурсам.

Конечно, этот механизм не позволяет получить все данные из текущего манифеста пода, хранящиеся в базе данных Kubernetes. Для получения всего объема необходимо обращаться непосредственно в Kubernetes API. А это требует дополнительных разрешений, описанных в правилах RBAC.  

## Использование Downward API через перменные среды окружения (env)  

Один из самых простых способов использоватьл Downward API - это передать информацию в контейнер как перменные среды окружения. Это делается с помощью поля
 ```valueFrom.fieldRef``` в определении перменной.  
 
В ```fieldPath``` можно указать различные поля из спецификации пода. Наиболее часто используемые:  

- ```metadata.name``` - имя пода.  
- ```metadata.namespace``` - ```namespace```, в котором запущен под.  
- ```status.podIP``` - IP-адрес пода.  
- ```spec.nodeName``` - имя узла(node), на котором запущен под.  
- ```spec.serviceAccountName``` - имя ```ServiceAccount```, использумого подом.  
- ```metadata.labels['<key>']``` - значения метки(label) с ключем ```<key>```.  
- ```metadata.annotation['<key>']``` - значение аннотации с ключем ```<key>```.  

Подробный список информации, которую можно передавать через переменные серды окружения смотрите в [документации на сайте проекта kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/)

Рассмотрим пример манифеста ```downwardc-api-env.yml```:  
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-env-demo
  labels:
    app.kubernetes.io/name: downward-demo
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  containers:
  - name: main
    image: busybox:1.37
    command:
    - sh
    - -c
    args:
    - |
      echo "---";
      echo "Pod name: $POD_NAME";
      echo "Pod namespace: $POD_NAMESPACE";
      echo "Pod ip: $POD_IP";
      echo "Node Name: $NODE_NAME";
      echo "App Label: $APP_LABEL";
      sleep infinity
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: APP_LABEL
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['app.kubernetes.io/instance']
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"

```  

Применим манифест:  
```sh
kubectl -n work apply -f downwardc-api-env.yml
```  
```sh
pod/downward-api-env-demo created
```  
Убедимся, что под запущен:  
```sh
kubectl -n work get po
```  
```sh
NAME                    READY   STATUS    RESTARTS   AGE
downward-api-env-demo   1/1     Running   0          4m57s
```  

Посмотрим логи контейнера:  

```sh
kubectl -n work logs downward-api-env-demo
```  
```sh
---
Pod name: downward-api-env-demo
Pod namespace: work
Pod ip: 10.0.2.243
Node Name: worker1
App Label: test
```  
Удалим под. Чтобы не ждать, когда система принудительно удалит под(значение ```grace period``` по умолчанию 30 секунд), используем параметр ```--force```:  
```sh
kubectl -n work delete -f  downwardc-api-env.yml --force
```  
```sh
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "downward-api-env-demo" force deleted
```  

## Использование Downward API через volume  

Другой способ - смонтировать информацию как файлы в ```volume``` типа ```downwardApi```. Этот метод более гибкий, т.к. позволяет передавать не только поля ```fieldRef``` но и информацию о ресурсах контейнера ```resourceFieldRef```, например лимиты CPU или памяти.  

Данные смонтированные через ```volume```, обновляются автоматически, если они изменяются(например - метки или аннотации). За исключения случая, когда вы используете ```subPath``` при монтировании пода.  

Рассмотрим пример манифеста ```downward-api-volume.yml```: 
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-volume-demo
  labels:
    app.kubernetes.io/name: downward-volume-demo
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
  annotations:
    author: "Artiom Rogov"
spec:
  containers:
  - name: main-container
    image: busybox:1.37
    command:
    - sh
    - -c
    args:
    - |
      echo "---";
      echo "Pod name: $(cat /etc/podinfo/pod_name)";
      echo "Pod namespace: $(cat /etc/podinfo/pod_namespace)";
      echo "Labels: $(cat /etc/podinfo/labels)";
      echo "Annotations: $(cat /etc/podinfo/annotations)"
      echo "CPU limits: $(cat /etc/podinfo/cpu_limit)"
      echo "Memory limit: $(cat /etc/podinfo/memory_limit)"
      sleep infinity
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  - name: podinfo
    downwardAPI:
      items:
        # Pod fields
        - path: "pod_name"
          fieldRef:
            fieldPath: metadata.name
        - path: "pod_namespace"
          fieldRef:
            fieldPath: metadata.namespace
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
        # Container fiels
        - path: "cpu_limit"
          resourceFieldRef:
            containerName: main-container
            resource: limits.cpu
            divisor: "1"
        - path: "memory_limit"
          resourceFieldRef:
            containerName: main-container
            resource: limits.memory
            divisor: 1Mi
  restartPolicy: Never

```  

В этом манифесте мы создаем том ```podinfo``` , который использует ```DownwardAPI``` . Внутри ```items``` мы определяем, какие данные и в какие файлы нужно поместить.  

- ```path``` - это имя файла, в который будет помещена информация.  
- ```fieldRef``` - используется для метаданных пода(имя, метки, аннотации).  
- ```resourceFieldRef``` - используется для получения информации о ресурсах контейнера(CPU, memory). Обратите внимение на ```containerName```, котолрое должно соответствовать имени контейнера, и ```divisor``` для форматирования значения. Для памяти мы используем ```1Mi```, чтобы получить значение в мегабайтах.  

Применим маннифест:  

```sh
kubectl -n work apply -f downward-api-volume.yml
```  
```sh
pod/downward-api-volume-demo created
```

Убедимся, что под запущен:  
```sh
kubectl -n work get po
```  
```sh
NAME                       READY   STATUS    RESTARTS   AGE
downward-api-volume-demo   1/1     Running   0          4s
```  

А теперь проверим логи пода:  
```sh
---
Pod name: downward-api-volume-demo
Pod namespace: work
Labels: app.kubernetes.io/instance="test"
app.kubernetes.io/name="downward-volume-demo"
app.kubernetes.io/version="v0.0.1"
Annotations: author="Artiom Rogov"
kubectl.kubernetes.io/last-applied-configuration="{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{\"author\":\"Artiom Rogov\"},\"labels\":{\"app.kubernetes.io/instance\":\"test\",\"app.kubernetes.io/name\":\"downward-volume-demo\",\"app.kubernetes.io/version\":\"v0.0.1\"},\"name\":\"downward-api-volume-demo\",\"namespace\":\"work\"},\"spec\":{\"containers\":[{\"args\":[\"echo \\\"---\\\";\\necho \\\"Pod name: $(cat /etc/podinfo/pod_name)\\\";\\necho \\\"Pod namespace: $(cat /etc/podinfo/pod_namespace)\\\";\\necho \\\"Labels: $(cat /etc/podinfo/labels)\\\";\\necho \\\"Annotations: $(cat /etc/podinfo/annotations)\\\"\\necho \\\"CPU limits: $(cat /etc/podinfo/cpu_limit)\\\"\\necho \\\"Memory limit: $(cat /etc/podinfo/memory_limit)\\\"\\nsleep infinity\\n\"],\"command\":[\"sh\",\"-c\"],\"image\":\"busybox:1.37\",\"name\":\"main-container\",\"resources\":{\"limits\":{\"cpu\":\"500m\",\"memory\":\"128Mi\"},\"requests\":{\"cpu\":\"250m\",\"memory\":\"64Mi\"}},\"volumeMounts\":[{\"mountPath\":\"/etc/podinfo\",\"name\":\"podinfo\"}]}],\"restartPolicy\":\"Never\",\"volumes\":[{\"downwardAPI\":{\"items\":[{\"fieldRef\":{\"fieldPath\":\"metadata.name\"},\"path\":\"pod_name\"},{\"fieldRef\":{\"fieldPath\":\"metadata.namespace\"},\"path\":\"pod_namespace\"},{\"fieldRef\":{\"fieldPath\":\"metadata.labels\"},\"path\":\"labels\"},{\"fieldRef\":{\"fieldPath\":\"metadata.annotations\"},\"path\":\"annotations\"},{\"path\":\"cpu_limit\",\"resourceFieldRef\":{\"containerName\":\"main-container\",\"resource\":\"limits.cpu\"}},{\"path\":\"memory_limit\",\"resourceFieldRef\":{\"containerName\":\"main-container\",\"divisor\":\"1Mi\",\"resource\":\"limits.memory\"}}]},\"name\":\"podinfo\"}]}}\n"
kubernetes.io/config.seen="2026-04-25T13:55:27.733339347Z"
kubernetes.io/config.source="api"
CPU limits: 1
Memory limit: 128
```  

Удалим под:  
```sh
kubectl -n work delete po downward-api-volume-demo --force
```
```sh
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "downward-api-volume-demo" force deleted
```

## Динамическое обновление данных  

Ключевое преимущество ```downwardAPI``` при использовании ```volume``` заключается в том, что данные в файлах обновляются автоматически  при изменении метаданных пода. В отличиях от перменных среды окружения, которые устанавливаются одни раз при запуске пода контейнера, файлы в ```downwardAPI volume``` отражают актуальное состояние ресурса.  

Продемонстрируем это на прмере. Создадим под, который будет переодически считывать свои метки и выводить их в лог. Затем мы изменим одну из меток и убедимся, что приложение внутри контейнера увидит это изменение. 

Рассмотрим манифест ```downward-api-volume-dynamic.yml```:  
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-dynamic-demo
  labels:
    app.kubernetes.io/name: downward-api-dynamic-demo
    status: pending
spec:
  containers:
  - name: main-container
    image: busybox:1.37
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        echo "---"";
        echo "Readin labels at $(date): ";
        cat /etc/podinfo/labels;
        sleep 10;
      done
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    volumeMounts:
      - name: podinfo
        mountPath: /etc/podinfo
  volumes:
    downwardAPI:
      items:
        - name: "labels"
          fieldRef:
            fieldPath: metadata.labels
  restartPolicy: Never

```  

Применим манифест:  

```sh
kubectl -n work apply -f downwardAPI/downward-api-volume-dynamic.yml
```  
```sh
pod/downward-api-dynamic-demo created
```  
Проверим, чтобы под запустился:  
```sh
kubectl -n work get po
```  
```sh
NAME                        READY   STATUS    RESTARTS   AGE
downward-api-dynamic-demo   1/1     Running   0          5s
```  

Проверим логи контейнера в режиме слежения ```-f```, чтобы видеть обновления в реальном времени: 
```sh
kubectl -n work logs -f downward-api-dynamic-demo
```

```sh
Readin labels at Sat Apr 25 14:22:33 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"---
Readin labels at Sat Apr 25 14:22:43 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"---
Readin labels at Sat Apr 25 14:22:53 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"---
Readin labels at Sat Apr 25 14:23:03 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"---
Readin labels at Sat Apr 25 14:23:13 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"---
Readin labels at Sat Apr 25 14:23:23 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"---
Readin labels at Sat Apr 25 14:23:33 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
```  

Не прерывая выполнения предыдущей команды откройте новый терминал и измените метку ```status``` с ```pending``` на ```running```. Флаг ```--overwrite``` необходим для изменения существующей метки.  

```sh
kubectl -n work label pod downward-api-dynamic-demo status=running --overwrite
```  
```sh
pod/downward-api-dynamic-demo labeled
```  

Вернувшись в первый терминал, вы увидите, что содержимое файла ```/etc/podinfo/labels``` изменилось, и приложение теперь видит новое значение метки:  

```sh
Readin labels at Sat Apr 25 14:32:10 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="running"---
Readin labels at Sat Apr 25 14:32:20 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="running"---
Readin labels at Sat Apr 25 14:32:30 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
status="running"---
Readin labels at Sat Apr 25 14:32:40 UTC 2026: 
app.kubernetes.io/name="downward-api-dynamic-demo"
```  

Этот механизм позволяет создавать приложения, которые могут динамически реагировать на изменения в своей конфигурации или состоянии в Kubernetes без необходимости прямого обращения к API-серверу.  

Посмотрим внутри пода на дриекторию, куда монтируются данные:
```sh
kubectl -n work exec -it downward-api-dynamic-demo -- ls -la /etc/podinfo
```  
```sh
total 4
drwxrwxrwt    3 root     root           100 Apr 25 14:31 .
drwxr-xr-x    1 root     root          4096 Apr 25 14:28 ..
drwxr-xr-x    2 root     root            60 Apr 25 14:31 ..2026_04_25_14_31_20.1137587088
lrwxrwxrwx    1 root     root            32 Apr 25 14:31 ..data -> ..2026_04_25_14_31_20.1137587088
lrwxrwxrwx    1 root     root            13 Apr 25 14:28 labels -> ..data/labels
```  

Удаляем под:
```sh
kubectl -n work delete po downward-api-dynamic-demo --force
```