# EmptyDir

[Основные характеристики](#основные-характеристики-emptydir).  
[Типичные сценарии использования(Use Cases)](#типичные-сценарии-использованияuse-cases).  
[Создание EmptyDir](#создание-emptydir).  
[Использование EmptyDir с различными типами носителей.](#использование-emptydir-с-различными-типами-носителей).  
[Управление размером emtpyDir.](#управление-размером-emtpydir)  
[Пример использования EmtpyDir для обмена данными между контейнерами](#пример-использования-emtpydir-для-обмена-данными-между-контейнерами)  
[Как тома emptyDir отображаются на ФС сервера](#как-тома-emptydir-отображаются-на-фс-сервера)  
[Пример использования EmptyDir с InitContainer](#пример-использования-emptydir-с-initcontainer)  
[Ограничения](#ограничения)  



Тома типа ```emptyDir``` используются для предоставления временного хранилища поду. Они создаются при запуске пода и удаляются при его завершении. Данные в таких томах доступны всем контейнерам пода и могут использоваться для обмена информации между подами. 

## Основные характеристики emptyDir  

```emptyDir``` - это том Kubernetes, который:  
- Создается при запуске пода и удаляется при его завершении. 
- Изначально пустой(отсюда и название). 
- Доступен всем контейнерам в поде. 
- Может использовать различные типы носителей(диск,память). 
- Не сохраняет данные после удаления пода. 

Физически файлы томов типа ```emtpyDir``` хранятся на диске узла. где запущен под. Путь к этим файлам обычно находится в директории ```/var/lib/kubelet/pods/<pod_uid>/volumes/kubernetes.io~empty-dir```. Эти тома создаются ```kubelet```. 

## Типичные сценарии использования(Use Cases)  
```emptyDir``` идеально подходит для следующих сценариев:  
- Обмен данных между контейнерами. Как показано в примерах ниже, ```emptyDir``` является простым и эффиктивным способом для организации обмена файлами между несколькими контейнерами в одном поде.  
- Временное хранилище: Для приложений, которым нужно временное пространство для кеширования, хранения промежуточных результатов вычислений или логов, которые не требуется сохранять надолго. 
- Подготовка данных с помощью Init Containers: ```InitContainer``` может сгенерировать или загрузить конфигурационные файлы, скрипты или другие данные в ```emptyDir```, которые затем будут использоваться основным контейнером.  

## Создание EmptyDir  

EmptyDir создается при определении пода в спецификации volumes. Пример простого использования: 
```yml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app.kubernetes.io/name: myapp
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:
    - name: temp-storage
      mountPath: /tmp/data
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    ports:
      - containerPort: 80
  volumes:
  - name: temp-storage
    emptyDir: {}
```  

## Использование EmptyDir с различными типами носителей. 

EmptyDir может использовать различные типы носителей, которые определяются в поле medium:  

1. По умолчанию(пустое значение) - используется диск узла. Наиболее распространенный вариант
2. Memory - использует ```tmpfs```(файловая система в памяти). Это значительно ускоряет операции с файлами, но следует помнить, что это потребляет оперативную память узла. 
Используйте этот вариант для небольших объемов чувствительных к скорости доступа данных, например для кеша.  

Пример испольования tmpfs:  
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app.kubernetes.io/name: myapp
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:
    - name: memory-storage
      mountPath: /tmp/memory-data
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    ports:
      - containerPort: 80
  volumes:
  - name: memory-storage
    emptyDir:
      medium: Memory
```  

**Внимани** При использованиии ```medium: Memory``` том будет потреблять оперативную память узла. Если лимиты контейнера на память будут превышны, то может привести к его завершению (OOMkilled).  

## Управление размером emtpyDir.  

Начиная с Kubernetes 1.22, можно ограничить размер emtpyDir с помощью поля sizeLimit . Это помогает предотвратить переполнение диска узла из-за неконтролируемого роста временных файлов.  

```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-sized-pod
  labels:
    app.kubernetes.io/name: my-sized-pod
spec:
  containers:
  - name: myapp
    image: busybox
    volumeMounts:
    - name: data-storage
      mountPath: /data
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    ports:
      - containerPort: 80
  volumes:
  - name: data-storage
    emptyDir:
      sizeLimit: "200Mi"
```  

в этом примере, если контейнер попытается записать в /data  более 200MBб под будет выселен(evicted) с ноды.  

## Пример использования EmtpyDir для обмена данными между контейнерами  

```emptyDir``` идеально подходит длля обмена данными между несколькими контейнерами в одном поде. В этом примере один контейнер (producer) буде записаывать данные в ```emptyDir```, а другой(consumer) - читать их.  

Сначала создайте файл манифеста ```emptydir-share.yml```:  
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-share-pod
  labels:
    app.kubernetes.io/name: emptydir-share-pod
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  volumes:
  - name: shared-volume
    emptyDir: {}
  containers:
  - name: producer
    image: busybox:1.28
    command:
      - "/bin/sh"
      - "-c"
    args:
    - |
      while true; do
      echo "Data generated at $(date)" >> /shared-data/data.txt; done
    volumeMounts:
    - name: shared-volume
      mountPath: /shared-data
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
  - name: consumer
    image: busybox:1.28
    command:
    - "/bin/sh"
    - "-c"
    args:
    - |
      tail -f /shared-data/data.txt
    volumeMounts:
    - name: shared-volume
      mountPath: /shared-data
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```  

Применим манифест:  

```sh
kubectl -n work apply -f emptydir-share.yml
```  
```sh
pod/emptydir-share-pod created
```  

Проверьте, что под запустился:  

```sh
kubectl get po
NAME                 READY   STATUS    RESTARTS   AGE
emptydir-share-pod   2/2     Running   0          37s
```  

Теперь мы можем посмотреть логи контейнера-потребетиля, чтобы убедиться, что он читает данные, которые генерирует контейнер-произвдитель:  
```sh
kubectl -n work logs -f emptydir-share-pod -c consumer
```  
Результат будет выглядить прмерно так, с новыми строками, появляющимеся каждую секунду:  
```sh
Data generated at Sat Apr 25 11:56:31 UTC 2026
Data generated at Sat Apr 25 11:56:31 UTC 2026
Data generated at Sat Apr 25 11:56:31 UTC 2026
Data generated at Sat Apr 25 11:56:31 UTC 2026
Data generated at Sat Apr 25 11:56:31 UTC 2026
Data generated at Sat Apr 25 11:56:31 UTC 2026
Data generated at Sat Apr 25 11:56:31 UTC 2026
```  

## Как тома emptyDir отображаются на ФС сервера

Посмотрим на "физическую" реализацию ```emptyDir``` в файловой системе сервера. Для этого нам необходимо узнать:  
- Сервер на котором находится запущенный под  
- UID пода  

```sh
kubectl -n work get pod emptydir-share-pod -o custom-columns=NAME:.metadata.name,UID:.metadata.uid,NODE:.spec.nodeName
```  
```sh
NAME                 UID                                    NODE
emptydir-share-pod   c9d10f64-0345-4bf9-a453-56185d83f373   worker1
```  

Заходим на сервер и смотрим содержимое директории:  
```sh
root@worker1:~# ls -la /var/lib/kubelet/pods/c9d10f64-0345-4bf9-a453-56185d83f373/volumes/kubernetes.io~empty-dir/shared-volume/
total 7184
drwxrwxrwx 2 root root    4096 Apr 25 12:00 .
drwxr-xr-x 3 root root    4096 Apr 25 12:00 ..
-rw-r--r-- 1 root root 7343609 Apr 25 12:06 data.txt
```  

Удалите под:  

```sh
kubectl -n work delete pod emptydir-share-pod
```  

## Пример использования EmptyDir с InitContainer  

Следующий пример демонстрирует, как ```InitContainer``` может подготовить конфигурационные файлы для осноного контейнера, используя emptyDir в качестве общего хранилища.  

Сначала создадим ```ConfigMap```, который содержит шаблон конфигурации. Сохраните этот манифест в файле ```emptydir-initc-cf.yml```:  
```yml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app.kubernetes.io/name: app-config
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
data:
  config-template.txt: |
    APP_NAME={{APP_NAME}}
    APP_VERSION={{APP_VERSION}}
    DATABASE_URL={{DATABASE_URL}}

```  

Примение манифест для создания ```ConfigMap``` в кластере:  

```sh
kubectl -n work apply -f emptydir-initc-cf.yml 
```  
```sh
configmap/app-config created
```  

Теперь создадим ```Deployment```.```InitContainer```(```config-init```) прочитает шаблон из ```ConfigMap```, заменит переменные-плейсхлолдеры на реальные занчения из переменных окружения и запишет итоговый файл в ```emtpyDir``` том. Основной контейнер (```app-container```) затем сможет использовать этот готовый файл.  

Сохраните манифест в файле ```emtpydir-initc-dep.yml```:  
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app.kubernetes.io/name: &name app-deployment
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: *name
      app.kubernetes.io/instance: *instance
      app.kubernetes.io/version: *version
  template:
    metadata:
      labels:
        app.kubernetes.io/name: *name
        app.kubernetes.io/instance: *instance
        app.kubernetes.io/version: *version
    spec:
      volumes:
      - name: config-template
        configMap:
          name: app-config
      - name: app-config
        emptyDir: {}
      initContainers:
      - name: config-init
        image: busybox:1.28
        imagePullPolicy: IfNotPresent
        command:
        - sh
        - -c
        - |
          sed -e "s|{{APP_NAME}}|$APP_NAME|g" \
              -e "s|{{APP_VERSION}}|$APP_VERSION|g" \
              -e "s|{{DATABASE_URL}}|$DATABASE_URL|g" \
          /config-template/config-template.txt > /app-config/app-config.txt
          echo "Done"
        env:
        - name: APP_NAME
          value: "MyApplication"
        - name: APP_VERSION
          value: "v1.0.0"
        - name: DATABASE_URL
          value: "postresql://user:password@db:5432/mydb"
        volumeMounts:
        - name: config-template
          mountPath: /config-template
        - name: app-config
          mountPath: /app-config
      containers:
      - name: app-container
        image: nginx:1.28
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        volumeMounts:
        - name: app-config
          mountPath: /etc/app-config
          readOnly: true
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"

```  

Применяем манфиест:  
```sh
kubectl -n work apply -f emtpydir-initc-dep.yml
```  
Проверьте под
```sh
kubectl -n work get po -l app.kubernetes.io/name=app-deployment
```  

Кгода под перейдет в статус ```Running```, это будет означать, что ```InitContainer``` успешно завершил свою работу. Теперь можно проверить содержимое сгененированного файла внутри основного контейнера.  

```sh
kubectl -n work exec -it \
  $(kubectl -n work get pods -l app.kubernetes.io/name=app-deployment -o jsonpath='{.items[0].metadata.name}') \ -c app-container -- cat /etc/app-config/app-config.txt
```  
Результат:  
```sh
APP_NAME=MyApplication
APP_VERSION=v1.0.0
DATABASE_URL=postresql://user:password@db:5432/mydb
```  
Не забудьте удалить созданные ресурсы:  
```sh
kubectl -n work delete -f emptyDir/emptydir-initc-cf.yml
kubectl -n work delete -f emtpydir-initc-dep.yml
```  

## Ограничения

Несмотря на свою простоту и удобство, ```emptyDir``` имеет важные ограничения:  
- Данные не сохраняются: Это самое главное огранчение. После удаления пода данные полностью удаляются и будут безвозвратно утеряны.  
- Привязкя к узлу: Том ```emptyDir``` существуют только на том узле где запущен под. Если под будет перенесен на другой узел, ```emptyDir``` не перенесется вместе с ним.  