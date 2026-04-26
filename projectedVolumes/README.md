# Projected Volumes

Механизм [projected volume](https://kubernetes.io/docs/concepts/storage/projected-volumes/) в Kubernetes позволяет монтировать несколько источников данных в одну и ту же директорию внутри контейнера. Этот инструмент, который позволяет гибко управлять конфигурацией, секретами и метаданными, доступными для приложения.  


В отличи и от монтирования каждого источника как отдельного тома, ```projected``` volume объединяет их, что упращает управление файлами конфигурации.  

## Источники данных для projected-томов  

```Projected``` том может включать в себя следующие типы источников:  
- ```secret``` - Позволяет монтировать данные из одного или нескольких Secret  
- ```configMap``` - Используются для монтирования конфигурационных данных из ConfigMap  
- ```downwardAPI``` - Подключает downwardAPI  
- ```serviceAccountToken``` - Монтирует токен для ```ServiceAccount```. Хотя Kubernetes по умолчанию автоматически монтирует токен в каждый под по пути ```/var/run/secrets/kuberntes.io/serviceaccount/token```, использование ```projected``` тома дает дополнительную гибкость. Например, можно настроитьо аудиторию ```audience``` и срок действия ```expiratioinSeconds``` токена, а так же смонтировать его в нестандартный путь.  

Все источники в ```projected``` томе монтируются в одну директорию, указанную в ```volumeMounts```. Важдно отметить, что данные из ```ConfigMap```, ```Secret``` и ```downwardAPI```(напрмет метки и\или аннотации) обновляются автоматически если они изменяются.  

## Использование projected-тома  

Рассмотрим классический пример, в котором мы объеденим несколько источников данных в одном ```projected``` томе. Мы создадим ```ConfigMap``` для конфигурации приложения, ```Secret``` для учетных данных и используем ```downwardAPI``` для получени я метаданных пода.  

```ConfigMap``` с настройками приложения. Сохраним манифест в файле ```projected-configmap.yml```:  
```yml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    log_level=INFO
    feature_flags=new-ui,dark-mode

```  
```Secret``` с учетными данными. Сохраним манифест в файле ```projected-secret.yml```:   

```yml
---
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: "admin"
  password: "superSecretPassword123"

```  

Применим оба манифеста:  

```sh
kubectl -n work apply \
-f projected-configmap.yml \
-f projected-secret.yml 
```  
```sh
configmap/app-config created
secret/db-credentials created
```  

Создадим под, который будет использовать ```projected``` том дя монтирования данных из ```ConfigMap```, ```Secret```, ```downwardAPI```. Манифест ```projected-volume-demo.yml```:  

```yml
---
---
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume-demo
  labels:
    app.kubernetes.io/name: projected-volume-demo
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  volumes:
  - name: all-in-one-volume
    projected:
      sources:
        # Источник 1: DownwardAPI для метаданных пода
        - downwardAPI:
            items:
            - path: "pod_name"
              fieldRef:
                fieldPath: metadata.name
            - path: "pod_namespace"
              fieldRef:
                fieldPath: metadata.namespace
        # Источник 2: ConfigMap для конфигурации приложения
        - configMap:
            name: app-config
            items:
            - key: app.properties
              path: app.properties
        # Источник 3: Secret для учетных данных
        - secret:
            name: db-credentials #имя secret
            items:
            - key: username
              path: username
            - key: password
              path: password
  containers:
  - name: main-container
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c"]
    args:  
    - |
      echo "---Reading projectoed data---";
      echo "[+] Pod Info:";
      cat /etc/pod-data/pod_name;
      echo "";
      cat /etc/pod-data/pod_namespace;
      echo "";
      echo "[+] App config:";
      cat /etc/pod-data/app.properties;
      echo "";
      echo "[+] DB Credentials:";
      echo "Username: $(cat /etc/pod-data/username)";
      echo "Password: $(cat /etc/pod-data/password)";
      echo "Done";
      sleep infinity; # Засыпаем на бесконечность
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    volumeMounts:
    - name: all-in-one-volume
      mountPath: /etc/pod-data
      readOnly: true # рекомендуется для projected томов
  restartPolicy: Never

```

В этом манифесте:  

- Мы определяем том ```all-in-one-volume``` типа ```projected```.  
- В секции ```sources``` мы перечисляем все источники данных: ```downwardAPI```, ```configMap``` и  ```secret```.  
- Каждый источник указывает, какие данные(items) и в какие файлы(path) нужно поместить внутри точки монтирования ```/etd/pod-data```.  

Применим манифест:  
```sh
kubectl -n work apply -f projected-volume-demo.yml
```  
```sh
pod/projected-volume-demo created
```  

Проверим, что под успешно создан:  
```sh
kubectl -n work get po
```  
```sh
NAME                    READY   STATUS    RESTARTS   AGE
projected-volume-demo   1/1     Running   0          30s
```  

И теперь проверим логи нашего пода:  
```sh
kubectl -n work logs projected-volume-demo
``` 
```sh
---Reading projectoed data---
[+] Pod Info:
projected-volume-demo
work
[+] App config:
log_level=INFO
feature_flags=new-ui,dark-mode

[+] DB Credentials:
Username: admin
Password: superSecretPassword123
Done
```  

Как видно из логов, приложение успешно прочитало:  
- Имя и namespace пода из ```downwardAPI```.  
- Конфигурацию из ```ConfigMap```.  
- Учетные данные из ```Secret```.  

И вс е это из одной директории ```/etc/pod-data```, что делает конфигурацию приложения чистой и предсказуемой.  

Не забудьте удалить ресурсы:  
```sh
kubectl -n work delete \
-f projected-volume-demo.yml \
-f projected-secret.yml \
-f projected-configmap.yml 
```  
```sh
pod "projected-volume-demo" deleted
secret "db-credentials" deleted
configmap "app-config" deleted
```  

## Расширенная настройка ServiceAccountToken  

Как мы уже упоминали, ```projected``` том позволяет гибко настраивать ```ServiceAccountToken```. Рассмотрим пример, где мы зададим для токена срок действия (```expirationSeconds```), аудиторию(```audience```) и смонтируем его в нестандартный путь.  

Это особенно полезно для повышения безопасности, когда токен предназначен для конкретного внешнего сервиса(напрмер, ```api-gateway```) и должен быть недействительным через определенное время.  

Манифест ```projected-satoken.yml```:

```yml
---
---
apiVersion: v1
kind: Pod
metadata:
  name: projected-sa-demo
  labels:
    app.kubernetes.io/name: projected-sa-demo
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  containers:
  - name: main-container
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c"]
    args:  
    - |
      echo "---Custom Service Account Token---";
      echo "[+] Token Path: /etc/service-account/token";
      echo "[+] Token content (last 20 chars):";
      cat /etc/service-account/token | tail -c -20;
      echo "";
      echo "---Default Service Account Token---";
      echo "[+] Token path: /var/run/secrets/kubernetes.io/serviceacoount/token";
      echo "[+] Token content (last 20 chars):";
      cat /var/run/secrets/kubernetes.io/serviceaccount/token | tail -c -20;
      echo "";
      sleep infinity;
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    volumeMounts:
    - name: custom-token
      mountPath: /etc/service-account
      readOnly: true # рекомендуется для projected томов
  volumes:
  - name: custom-token
    projected:
      sources:
        - serviceAccountToken:
            path: token # Имя файла для ткоена
            expirationSeconds: 3600 # Срок действия 1 час
            audience: "api-gateway" # Аудитория, для которой предназначен токен
  restartPolicy: Never

```  

В этом примере мы:  

- Создаем ```projected``` том, который содержи только один источник - ```ServiceAccountToken```.  
- ```path: token``` - указывает, что токен будет сохранен в файле с именем ```token```.  
- ```expirationSeconds: 3600``` - устанавливает срок жизник токена в 1 час.  
- ```audience: "api-gateway"``` - определяет, что этот токен предназначен для аутентифкации на сервисе с идентификатором ```api-gateway```.  
Применим манифест:  
```sh
kubectl -n work apply -f projected-satoken.yml 
```  
```sh
pod/projected-sa-demo created
```  
Проверим под:  
```sh
kubectl -n work get po
```  
```sh
meus@meus:~/regru/k8s-maifests/projectedVolumes$ kubectl -n work get po
NAME                READY   STATUS    RESTARTS   AGE
projected-sa-demo   1/1     Running   0          4s
```  

И теперь проверим логи. Приложение выведет последние 20 символов как кастомного, так и стандартного токена и убедимся, что токены разные:  
```sh
kubectl -n work logs projected-sa-demo
```  
```sh
---Custom Service Account Token---
[+] Token Path: /etc/service-account/token
[+] Token content (last 20 chars):
kUI_2-gsrWG3YXlTdFCQ
---Default Service Account Token---
[+] Token path: /var/run/secrets/kubernetes.io/serviceacoount/token
[+] Token content (last 20 chars):
zS7z2CqFVtt1DiuLxg3g
```  

Как мы видем, оба токена существуют, но они разные, тк кастомный токен был сгенерирован с дополнительными параметрами.  

Не забудьте удалить под:  
```
kubectl -n work delete pod projected-sa-demo
```