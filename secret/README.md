[Secret](#secret)  
[Основные характеристики Secret](#основные-характеристики-secret)  
[Создание Secret](#создание-secret)  
- [Из литералов](#из-литералов)    
- [Из файлов](#из-файлов)  
- [Из YAML-манифеста](#из-yaml-манифеста)  
- [Тип kubernetes.io/tls](#тип-kubernetesiotls)  
- [Тип kubernetes.io/dockerconfigjson](#тип-kubernetesiodockerconfigjson)  

[Использование Secret](#использование-secret)  
[Файлы](#файлы)  
[Безопасность Secret](#безопасность-secret)  

# Secret  
В Kubernteses часто возникает необходимость передачи приложение конфигурационных данных. Для не конфидицильной информации используются ```ConfigMap```.  
Однако когда речь заходит о паролях, токенах, TLS-ключах, ```CongifMap``` не обеспечивает должного уровня безопасности, т.к. хранит данные в открытом виде.  

Для таких случаев предназначено объект ```Secret```.Он позволяет хранить и управлять конфидициальной информацией. Хотя по умолчанию ```Secret``` лишь кодирует данные в ```base64```. Его использование является обязательной практикой для безопасного управления чувствительными данными в кластере.  

```Secret``` - это специальный объект для конф данных. **Но важно помнить**: по умолчанию он лишь кодирует данные, а не шифрует их. Думайте об этом как о конферте, а не как о сейфе. 

## Основные характеристики Secret  

- Данные хранятся в формате base64
- Максимальный размер - 1 Mb
- Может быть смонтирован в под как том или использован для определения переменных окружения
- Поддерживает несколько типов: ```Opaque```,```kubernetes.io/tls```,```kubernetes.io/dockerconfigjson``` и другие  

## Создание Secret

Существует несколько способов создания Secret:  

### Из литералов
```sh
kubectl -n work create secret generic my-secret  --from-literal=username=admin --from-literal=password=secretpassword
```  

```generic``` создает ```Secret``` типа ```Opaque```  

После создания Secret можно проверить командой:  
```kubectl -n work get secrets```  
Результат:  
```sh
NAME        TYPE     DATA   AGE
my-secret   Opaque   2      64s
```  

Для подробной информации о Secret:

```kubectl -n  work describe secret my-secret``` 

Результат:  
```sh
Name:         my-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  14 bytes
username:  5 bytes
```  

Как мы видим, команда ```describe``` не показывает содержимое Secret, только размер данных.  

Для получения значений данных из Secret м ожно использовать следующую команду:  
```sh
kubectl -n work get secret my-secret -o jsonpath='{.data}' | jq
```  

Результат будет примерно таким:  

```json
{
  "password": "c2VjcmV0cGFzc3dvcmQ=",
  "username": "YWRtaW4="
}
```  

Для получения значения конкретного ключа и его декодирования из base64 в читаемый формат можно использовать следующую команду:  

```sh
kubectl -n work get secret my-secret -o jsonpath='{.data.username}' | base64 --decode
```  

Результат:  

```txt
admin
```  

Таким же образом мы можем получить значение из поля password.  

Удалим ```Secret```:  

```kubectl -n work delete secret my-secret```  

### Из файлов  

Содержимое ```Secret``` при его создании можно получить из файлов:  

```sh
echo -n 'admin' > ./username.txt
echo -n 'secretpassword' > ./password.txt
kubectl -n work create secret generic my-secret --from-file=./username.txt --from-file=./password.txt
```  

После создания Secret можно проверить командой: 

```sh
kubectl -n work get secrets
```  

Результат будет примерно таким:  
```sh
NAME        TYPE     DATA   AGE
my-secret   Opaque   2      16s
```  

Для получения подробной информации о Secret:  

```sh
kubectl -n work describe secret my-secret
```  

Результат:  
```sh
Name:         my-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  14 bytes
username.txt:  5 bytes
```  

Обратите внимание на то что при создании ```Secret``` из файлов название ключей в данных будут соответствовать именам файлов.  
Удалим ```Secret``` и файлы:  

```sh
kubectl -n work delete secret my-secret
rm -f password.txt username.txt 
```  

### Из YAML-манифеста    

При создании Secret из YAML-манифеста можно использовать два поля для данных:  
- ```data```- для уже закодированных в base64 данных  
- ```stringData``` - для незакодированных данных(удобнее для использования)  

Пример с data 

Файл ```01-secret.yml```:  

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  #echo -n admin | base64
  username: YWRtaW4=
  #echo -n secretpassword | base64
  password: c2VjcmV0cGFzc3dvcmQ=

```  

Применим манифест:  

```sh
kubectl apply -n work -f 01-secret.yaml 
```  

После создания ```Secret``` проверьте его:  

```sh
kubectl -n work get secrets
```  

Результат:  

```sh
NAME        TYPE     DATA   AGE
my-secret   Opaque   2      7s
```  

Для получения подробной информации о ```Secret```:  
```sh
kubectl -n work describe secret my-secret
```  

Результат:  

```sh
Name:         my-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  14 bytes
username:  5 bytes
```  

**Пример со stringData**  

Файл ```02-secret-string.yml```:  

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  username: admin
  password: secretpassword

```  

Применим манифест:  
```sh
kubectl -n work apply -f 02-secret-string.yml
```  

Посмотрим как ```Secret``` хранится в базе данных Kubernetes:  

```sh
kubectl -n work get secret my-secret -o yaml
```  

Результат:  

```yaml
apiVersion: v1
data:
  password: c2VjcmV0cGFzc3dvcmQ=
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"my-secret","namespace":"work"},"stringData":{"password":"secretpassword","username":"admin"},"type":"Opaque"}
  creationTimestamp: "2026-04-20T10:42:15Z"
  name: my-secret
  namespace: work
  resourceVersion: "46901"
  uid: f45e4f35-6a9d-48ae-9e63-390345ee983b
type: Opaque
```  

Обратите внимание на ```annotations```:  

  1. Максимальный размер данных которых можно помещать в ConfigMap и Secret - 1 Mb  
  2. Максимальный размер значения аннотации - 256 KB.  
  3. Максимальный размер файла который можно разместить в ConfigMap и Secret - менее 256 KB  

Удалим ```Secret```:  

```sh
kubectl -n work delete secret my-secret
```  

**Пример data + stringData**  

Также можно комбинировать data и stringData в одном манифесте. В этом случае значения из ```stringData``` будут иметь приоритет.  
Файл ```03-secret-sd.yaml```:  
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  password: secretpassword
  username: admin
data:
  # echo -n administrator | base64
  username: YWRtaW5pc3RyYXRvcg==

```  

Применим манифест:  
```sh
kubectl -n work apply -f 03-secret-sd.yaml 
```  
```sh 
secret/my-secret created
```  

Получаем значение username:  
```sh
kubectl -n work get secret my-secret -o jsonpath='{.data.username}' | base64 --decode
```  
Результат:  
```sh
admin
```  

Удалим Secret:  
```sh 
kubectl -n work delete secret my-secret
```  

### Тип kubernetes.io/tls

```Secret``` типа ```kubernetes.io/tls``` используется для хранения TLS-сертификтов и приватных ключей. Для создания такого Secret  сначала нужно сгенерировать сертификат и ключ с помощью OpenSSL.  

Генерация приватного ключа:  
```sh  
openssl genrsa -out tls.key 2048
```  
Геренация самподписанного сертификата:  
```sh 
openssl req -new -x509 -key tls.key -out tls.crt -days 365 -subj "/CN=myapp"
```  

После генерации сертификата и ключа можно создать ```Secret``` типа ```tls```:  
```sh
kubectl -n work create secret tls my-tl-secret --cert=tls.crt --key=tls.key
```    
```sh 
secret/my-tl-secret created
```  

Посмотрим список секретов:  
```sh 
kubectl -n work get secrets
```  

Результат:  
```sh
NAME           TYPE                DATA   AGE
my-tl-secret   kubernetes.io/tls   2      5s
```  

Для получения информации о ```Secret```:  
```sh
kubectl -n work  describe secret my-tls-secret
```  

Результат:  
```sh
Name:         my-tls-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1103 bytes
tls.key:  1704 bytes
```  

Для получения содержимого сертификата и вывода его на стандартный вывод можно использовать следующую команду:  

```sh
kubectl -n work get secret my-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d
```  

Эта команда получает Secret в формате JSON, извлекает закодированное значение поля tls.crt, а затем декодирует его из base64.  
Удалим Secret и временные файлы:  

```sh
rm -f tls.*
kubectl -n work delete secret my-tls-secret
```  

### Тип kubernetes.io/dockerconfigjson    
Secret типа ```kubernetes.io/dockerconfigjson``` используется для хранения ученных данных Docker registry. Его можно создать двумя способами:  

   1. Используя файл конфигурации Docker:  
   ```sh
   kubectl create secret generic my-registry-secret \
   --from-file=.dockerconfigjson=/path/to/.docker/config.json \
   --type=kubernetes.io/dockerconfigjson
   ```  
   2. Используя команду ```kubectl create secret docker-registry```:  
   ```sh
   kubectl create secret docker-registry my-registry-secret \
   --docker-server=https://my-registry.example.com \
   --docker-username=myuser \
   --docker-password=mypassword \
   --docker-email=myemail@example.com
   ```  

Попробуем создать Secret используя 2-ой вариант.  
Результат:  
```sh
secret/my-registry-secret created
```  

Посмотрим список Secret:  
```sh
kubectl -n work get secrets
```  
Результат:  
```sh
NAME                 TYPE                             DATA   AGE
my-registry-secret   kubernetes.io/dockerconfigjson   1      73s
```
Обратите внимание на тип созданного секрета - ```kubernetes.io/dockerconfigjson```  
Получим подробную информацию о Secret:  
```sh
kubectl -n work describe secret my-registry-secret
```  

Результат:  
```sh
Name:         my-registry-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  155 bytes
```  
Удалим Secret:  

```sh
kubectl -n work delete secret my-registry-secret
```  

## Использование Secret

По аналогии с ```ConfigMap```,```Secret``` можно использовать для хранения перменных среды окружения или содержимого файлов.  

**Переменные среды окружения**

Файл ```04-secret-env-sd.yml```:  
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  USERNAME: admin
  PASSWORD: secretpassword
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
          - secretRef:
            name: my-secret       

```  

Применим манфиест:  
```sh
kubectl -n work apply -f 04-secret-env-sd.yml
```  

Результат:  
```sh
secret/my-secret configured
deployment.apps/testdp created
```  

Посмотрим переменные среды окружения в контейнере: 

Проверим поды:  
```sh
kubectl -n work get pods
```  

Нас интересует имя пода:  
```sh
NAME                     READY   STATUS    RESTARTS   AGE
testdp-b8c694895-56k8h   1/1     Running   0          2m14s
```  

Посмотрим интересующие нас переменные среды окружения в контейнере:  

```sh
kubectl -n work exec testdp-b8c694895-56k8h -- sh -c 'env | egrep "USERNAME|PASSWORD"'
```  
Результат:  
```sh
USERNAME=admin
PASSWORD=secretpassword
```  

Удалим манифест примера:  
```sh
kubectl delete -f 04-secret-env-sd
```  

Следующий пример показывает как использовать только одно значение из Secret.  
Файл ```05-secret-env-part.yml```:  
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  USERNAME: admin
  PASSWORD: secretpassword
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
          - name: PASSWORD
            valueFrom:
              secretKeyRef:
                name: my-secret
                key: PASSWORD
```  

Применим манифест:  
```sh
kubectl -n work apply -f 05-secret-env-part.yml
```  

Проверим поды:  

```sh
kubectl -n work get pods
```

Результат:  
```sh
NAME                      READY   STATUS    RESTARTS   AGE
testdp-5b78f4d8fd-dvdh8   1/1     Running   0          3s
```  

Проверим содержимое переменных окружающей среды: 

```sh
kubectl -n work exec testdp-5b78f4d8fd-dvdh8 -- sh -c 'env | egrep "PASSWORD"'
```

Результат:  
```sh
PASSWORD=secretpassword
```

Удалим пример:  
```sh
kubectl -n work delete -f 05-secret-env-part.yml 
```  

Результат:  

```sh
secret "my-secret" deleted
deployment.apps "testdp" deleted
```  

## Файлы  

```Secret``` типа ```Opaque``` может содержать небольшие текстовые файлы, которые можно подключить к поду в виде тома.  
Создадим ```Secret``` с текстовым файлом:  
```sh
echo "Это содержимое текстового файла, который будет хранится в Secret" > myfile.txt
kubectl -n work create secret generic my-file-secret --from-file=myfile.txt
```  

Результат:  
```sh
secret/my-file-secret created
```  

В манифесте ```06-secret-file.yml``` используем данные из ```Secret```:  
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
      - name: data
        secret:
          secretName: my-file-secret
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
        - name: data
          mountPath: "/etc/myfile"
          readOnly: true

```  

Применим манифест:  

```sh
kubectl -n work apply -f 06-secret-file.yml
```  

Проверим, что файл примонтировался:  
```sh
kubectl -n work get pods
```  
```sh
NAME                      READY   STATUS    RESTARTS   AGE
testdp-6bcb64c876-dcw7c   1/1     Running   0          6s
```  

И проверка файла:  
```sh
kubectl -n work exec testdp-6bcb64c876-dcw7c -- sh -c 'ls -l /etc/my*'
```  
```sh
total 0
lrwxrwxrwx    1 root     root            17 Apr 20 12:01 myfile.txt -> ..data/myfile.txt
```  

Проверим содержимое:  
```sh
kubectl -n work exec testdp-6bcb64c876-dcw7c -- sh -c 'cat /etc/myfile/myfile.txt'
```  

Результат:  
```sh
Это содержимое текстового файла, который будет хранится в Secret
```  

Удаляем манифест и файл:  
```sh
rm -f myfile.txt
kubectl -n work delete -f 06-secret-file.yml
```  

## Безопасность Secret  

Несмотря на то, что ```Secret``` обеспечивает базовую защиту данных, важно понимать его ограничения:  
- Данные хранятся в base64, что не является шифрованием  
- Рекомендуется включить шифрование данных в etcd на уровне кластер  
- Ограничивайте доступ к Secret через RBAC  
- Используйте Secret только в тех подах, где это действительно необходимо  
