***Тома в Kubernetes***   

Тома в кубернетесь можно условно разделить на два типа:  
- Ephemeral volumes - тома удаляемые после завершения работы пода  
- Persistent volumes - тома доступные после удаления пода  

***Ephemeral volume***  
Ephemeral тома создаются kubelete при запуске контейнеров пода. Если том находится на диске, он будет расположен гедто в директории /var/lib/kubelete/ Поэтому важно знать какие по размеру тома планируют использовать приложение в контейнере, чт бы заранее выделить необходимое место на диске.  

***Config Map***  

Тома типа ```configmap``` обычно используются для подключения конфигурационных файлов к контейнеру подов. Или для формирования переменных среды окружения приложения в контейнере.  

***Переменные среды окружения***  

**env**  
Предположим, что в контейнер необходимо передать значительно количество переменных среды окружения. Это можно сделать непосредственно в манифесте пода в разделе env шаблона контейнера  

Пример 01-env.yml:  
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testdp
  labels:
    app.kubernetes.io/name: &name testdp
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 2
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
      containers:
      - name: main
        image: busybox:1.28
        command:
          - 'sh'
          - '-c'
          - 'tail -f /dev/null'
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
        env:
          - name: TEST_ENV1
            value: "test1"
          - name: TEST_ENV2
            value: "test2"

```  
Так объявлять переменные окружения удобно если их 1-5 штук, но если их больше рекомендуется использовать configmap  

***envFrom***  

Создадим ```ConfigMap``` содержащий набор переменных среды окружения файл - ```02-env-configmap.yml```  
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  TEST_DATABASE_HOST: "postgresql.example.com"
  TEST_DATABASE_PORT: "5432"
  TEST_APP_ENV: "production"
  TEST_LOG_LEVEL: "info"
```  

Применяем configMap:  
```kubectl -n work apply -f 02-env-configmap.yml```  
```configmap/app-config created```  

Мы можем проверить список configMap есть в ```namespace``` work: 
```kubectl -n work get cs```  cs - сокращение от configmaps  

В примере ```03-env-configmap-dp1.yml``` подключаем созданный ```configMap``` с помощью конструкции ```envFrom```:  
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testdp
  labels:
    app.kubernetes.io/name: &name testdp
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 2
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
      containers:
      - name: main
        image: busybox:1.28
        command:
          - 'sh'
          - '-c'
          - 'tail -f /dev/null'
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
        envFrom:
          - configMapRef:
              name: app-config          

```  

С помощью команды:  
```kubectl exec testdp-6b9df6cc6d-csbpf -- sh -c 'env | grep TEST'```  
Получаем список переменных окружения:  
```sh
TEST_DATABASE_HOST=postgresql.example.com
TEST_APP_ENV=production
TEST_DATABASE_PORT=5432
TEST_LOG_LEVEL=info
```  

***env и valueFrom***

Бывают ситуации, когда в контейнер необходимо добавить только несколько переменных среды окружения из ```ConfigMap```. В следующем примере мы добавим только две переменные, находящиеся в ```ConfigMAp``` файл ```04-env-configmap-dp2.yml```  
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testdp
  labels:
    app.kubernetes.io/name: &name testdp
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 2
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
      containers:
      - name: main
        image: busybox:1.28
        command:
          - 'sh'
          - '-c'
          - 'tail -f /dev/null'
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
        env:
          - name: TEST_ENV1
            value: "test1"
          - name: TEST_ENV2
            value: "test2"
          - name: TEST_APP_ENV
            valueFrom:
              configMapKeyRef:
                name: app-config
                key: TEST_APP_ENV
          - name: TEST_LOG_LEVEL_CURRENT
            valueFrom:
              configMapKeyRef:
                name: app-config
                key: TEST_LOG_LEVEL

```

В этот раз мы используем стандартное ```env```. Но вместо ```value``` используем ```valueFrom``` в котором определяем из какого ```ConfigMap``` выбирать конкретное содерижмое.  

В случае переменной ```TEST_LOG_LEVEL_CURRENT``` мы видим, что её имя не обязательно должно совпадать с именем переменной в ```ConfigMap```.  

Применем манифиест:  
```kubectl -n work apply -f 04-env-configmap-dp2.yml```  
```deployment.apps/testdp created```  
```kubectl get pods```  
```sh
NAME                      READY   STATUS    RESTARTS   AGE
testdp-868548bd88-jzppw   1/1     Running   0          13s
testdp-868548bd88-rd64k   1/1     Running   0          13s
```
Проверяем переменные среды окружения контейнера и видим наши переменные:  
```kubectl exec testdp-868548bd88-rd64k -- sh -c 'env | grep TEST'```  
```sh
TEST_APP_ENV=production
TEST_ENV1=test1
TEST_ENV2=test2
TEST_LOG_LEVEL_CURRENT=info
```  
После проверки удаляем Deployment

```kubectl delte deployment testdp```  


***Пример ConfigMap с текстовыми файлами***  
Создадим ```ConfigMap``` с двумя текстовыми файлами ```05-file-configmap.yml```  
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
#immutable: true
data:
  file1.txt: |
    Test file 1
    and its
    contents
  file2.txt: |
    Test file 2
    ant its
    content
```  

Параметр ```immutable``` - отвечает за запрет изменения ```ConfigMap```. Те чтобы его изменить необходимо будет сначала удалить ```ConfigMap```, а потом создать по новой.  
Применяем ```ConfigMap```:  
``` kubectl -n work apply -f 05-file-configmap.yml```  

Проверить содержимое ```ConfigMap```:  
```kubectl -n work get cm app-config -o jsonpath='{.data}'```  
```json
{"file1.txt":"Test file 1\nand its\ncontents\n","file2.txt":"Test file 2\nant its\ncontent\n"}
```

***Подключение ConfigMap как volume***  

В отличии от переменных среды окружения, подключение ```ConfigMap``` как том в контейнер требует другой подход. В поде добавляют опреедение тома, в котором в качестве и сточника данных указывается ```ConfigMap```. В контейнерах этот том монтируется уже в файловую систему контейнера.  

Пример Deployment файл ```06-file-configmap-dp1.yml```:  
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testdp
  labels:
    app.kubernetes.io/name: &name testdp
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 2
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
        - name: config-volume
          configMap:
            name: app-config
      containers:
      - name: main
        image: busybox:1.28
        command:
          - 'sh'
          - '-c'
          - 'tail -f /dev/null'
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
        volumeMounts:
          - name: config-volume
            mountPath: /etc/config

```  

В шаблоне пода мы определяем том. Данные для том беруться из ```ConfigMap```:  
```yaml
    spec:
      volumes:
        - name: config-volume
          configMap:
            name: app-config
```

Потом непосредственно в определении контейнера мы этот том монтируем в файловую систему контейнера к директории ```/etc/config```:  
```yml
      containers:
      - name: main
        volumeMounts:
          - name: config-volume
            mountPath: /etc/config
```

Создаем ```Deployment```:  
```kubectl -n work apply -f 06-file-configmap-dp1.yml```  
```deployment.apps/testdp created```  
```kubectl -n work get pods```  
```sh
NAME                      READY   STATUS    RESTARTS   AGE
testdp-5f7f989889-jdnpf   1/1     Running   0          2m59s
testdp-5f7f989889-tz8gb   1/1     Running   0          2m59s
```  

Просматриваем содержимое директории ```/etc/config/``` в контейнере:  
```kubectl -n work exec testdp-5f7f989889-jdnpf -- sh -c 'ls -la /etc/config'```  

```sh
drwxrwxrwx    3 root     root          4096 Apr 19 14:12 .
drwxr-xr-x    1 root     root          4096 Apr 19 14:12 ..
drwxr-xr-x    2 root     root          4096 Apr 19 14:12 ..2026_04_19_14_12_46.50566948
lrwxrwxrwx    1 root     root            30 Apr 19 14:12 ..data -> ..2026_04_19_14_12_46.50566948
lrwxrwxrwx    1 root     root            16 Apr 19 14:12 file1.txt -> ..data/file1.txt
lrwxrwxrwx    1 root     root            16 Apr 19 14:12 file2.txt -> ..data/file2.txt
meus@meus:~/regru/k8s-maifests$ 
```  

Мы видим "маленькие хитрости" реализации монтирования файлов из ```ConfigMap```. Много символьных ссылок. Посмотрим содерижиморе реальной директории: 

```kubectl -n work exec testdp-5f7f989889-jdnpf -- sh -c 'ls -la /etc/config/..2026_04_19_14_12_46.50566948'```  
```sh
drwxr-xr-x    2 root     root          4096 Apr 19 14:12 .
drwxrwxrwx    3 root     root          4096 Apr 19 14:12 ..
-rw-r--r--    1 root     root            29 Apr 19 14:12 file1.txt
-rw-r--r--    1 root     root            28 Apr 19 14:12 file2.txt
```  

***Изменение содержимого ConfigMap```  

Система позволяет изменить содержимое ```ConfigMap```/ Что будет с данными в контейнере с уже подключенным ```ConfigMap```?  
Посмотрим содержимое файла ```file1.txt```:  
``` kubectl -n work exec testdp-5f7f989889-jdnpf -- sh -c 'cat /etc/config/file1.txt'```  
```sh
Test file 1
and its
contents
```  
Изменим содержимое ```ConfigMap```:  
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true
data:
  file1.txt: |
    Test file 1
    and its
    new contents
  file2.txt: |
    Test file 2
    ant its
    content
```  
Обновим манифест ```ConfigMap```:  
```kubectl -n work apply -f 07-file-configmap.yml```  

Убедимся, что содержимое файла ```file1.txt``` в ```ConfigMap``` в кластере изменилось:  

```kubectl -n work get cm app-config -o jsonpath='{.data.file1\.txt}'```  
```txt
Test file 1
and its
new contents
```  

Через некоторе врмемя посмотрим содержимое файла в контейнере обновится. Может занимать от 1 минуты и более.

***Подключение одного файла```  
Рассмотрим ситуацию, когда необходимо подключить только один файл из ```ConfigMap``` в директорию в которой уже есть другие файлы.  
Пример манифеста ```Deployment``` файл ```08-file-configmap-dp2.yml```:  
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testdp
  labels:
    app.kubernetes.io/name: &name testdp
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 2
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
        - name: config-volume
          configMap:
            name: app-config
      containers:
      - name: main
        image: busybox:1.28
        command:
          - 'sh'
          - '-c'
          - 'tail -f /dev/null'
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
        volumeMounts:
          - name: config-volume
            mountPath: /etc/subfile.txt
            subPath: file1.txt

```  

Для подключения только одного файла, без создания директории точки монтирования, будем использовать параметр ```subPath```. В котором мы указываем файл в ```ConfigMap```, а в ```mountPath``` пишем имя файла, на которое он будет отображен.  

```yml
        volumeMounts:
          - name: config-volume
            mountPath: /etc/subfile.txt
            subPath: file1.txt
```  
При этом в ```mountPath``` мы можем указать любое название файла, но вот в ```subPath``` мы должны указать ключ из ```ConfigMap```.  
Применяем манфиест:  
```kubectl -n work apply -f 08-file-configmap-dp2.yml```  
Получаем список подов:  
```kubectl -n work get pods```  
Проверям содержимое:  
```kubectl -n work exec testdp-844bcddd7d-czx2g -- sh -c 'ls -l /etc/s*'```  
```sh
-rw-------    1 root     root           243 Dec 31  2017 /etc/shadow
-rw-r--r--    1 root     root            33 Apr 19 14:41 /etc/subfile
```  
Посмотрим его содержимое:  

```kubectl -n work exec testdp-844bcddd7d-czx2g -- sh -c 'cat /etc/subfile'```  
```txt
Test file 1
and its
new contents
```  

В режиме ```subPath``` существует дополнительное ограничение: монтирование таких файлов доступно только в режим ```ro```. Т.е. налету данные не изменяются и необходимо производить перезагрузку подов. С другой стороны, редактировать конфигурационные файла в контейнере - это странное желание. Изменненые файлы обратно в ```ConfigMap``` не отображаются. 