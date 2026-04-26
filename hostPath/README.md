# hostPath

[hostPath](#hostpath)  
[Что такое hostPath](#что-такое-hostpath)  
[Пример 1. Запись данных на узел кластера](#пример-1-запись-данных-на-узел-кластера)  
[Пример 2. Обмен данными между подами](#пример-2-обмен-данными-между-подами)  
[Риски и ограничения hostPath](#риски-и-ограничения-hostpath)  
[Потеря данных при перезапуске пода на другом узле](#потеря-данных-при-перезапуске-пода-на-другом-узле)  
[Проблемы безопасности](#проблемы-безопасности)  
[Защита с помощью Admission Controllers](#защита-с-помощью-admission-controllers)  
[Рекомендации по использованию](#рекомендации-по-использованию)




## Что такое hostPath  

Том(Volume) типа [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/) позволяет монтировать файл или директорию из файловой системы хост-узла(node) непосредственно в под. Это один из самых простых способов обеспечить постоянное хранение данных, однако он имеет серьезные ограничения, которые делают его непригодным для большинства реальных приложений. 

В отличии от тома ```emptyDir```, которыйф пуст при создании пода и уничтожается вместе с ним, данные в ```hostPath``` volume сохраняются после перезапуска контейнера или даже всего пода. Однако, эти данные жестко привязаны к конкретному узлу кластера. Если под будет пересоздан на другом узле, он потеряет доступ к своим данным.  

## Пример 1. Запись данных на узел кластера  

Рассмотрим как под может записать данные в фс узла, на котором запущен.  

Создадим манифест ```hostpath-write.yml```:  
```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-write-demo
  labels:
    app.kubernetes.io/name: hostpath-write-demo
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
      echo "Hello form pod $(POD_NAME)!" > /data/hello.txt;
      echo "Data written to /data/hello.txt";
      echo "---";
      sleep infinity;
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    volumeMounts:
    - name: host-data
      mountPath: /data
  volumes:
  - name: host-data
    hostPath:
      # Путь на хост-машине
      path: /tmp/k8s-book-data
      # Kuberntes создаст директорию, если её нет.
      type: DirectoryOrCreate
  restartPolicy: Never

```  

В этом манифесте мы определяем ```volume``` с именем ```host-data``` и типом ```hostPath```. Он указывает на путь ```/tmp/k8s-book-data``` на хост машине. Поле ```type: DirectoryOrCreate``` гарантирует, что Kubernetes создаст эту директорию на узле, если она отсутсвует. Затем мы монтируем этот том в контейнер по пути ```/data```.  

Применим манифест:  
```sh
kubectl -n work apply -f hostpath-write.yml
```  
```sh
pod/hostpath-write-demo created
```  

Посмотрим логи:
```sh
kubectl -n work logs hostpath-write-demo
```  
```sh
Data written to /data/hello.txt
---
```  

Теперь самое интересное. Нам нужно узнать на каком узле запущен под и проверить содержимое файла на самом узле: 

```sh
# Узнаем имя узла
NODE_NAME=$(kubectl -n work get pod hostpath-write-demo -o jsonpath='{.spec.nodeName}') && \
echo "Pod is running on node: $NODE_NAME"

#Выполняем команду на узле для чтения файла
#(Эта команда предполагает наличие ssh-доступа к узлам)
ssh username@$NODE_NAME -- cat /tmp/k8s-book-data/hello.txt
```  
```sh
Hello form pod hostpath-write-demo!
```  

Как мы видим, файл, созданный внутри контейнера, успешно сохранился в фс узла.  
Не забудьте удалить под и созданные им данные на узле:  

```sh
# Удаляем под
kubectl -n work delete -f hostpath-write.yml --force

#Удаляем директорию с данными на узле
#(Используем переменную $NODE_PATH, определенную ранее).

ssh "root@${NODE_NAME}" -- rm -rf /tmp/k8s-book-data && \ 
echo "Cleaned up data on $NODE_NAME"
```  

Так директория ```/tmp/k8s-book-data``` была создана Kubernetes, для её удаления на узле могут потребоваться права superuser(sudo).  

## Пример 2. Обмен данными между подами  

Поскольку ```hostPath``` используюет общую для всех подов на одном узле фс, его можно использовать для обмена данных между ними.  

Рассмотрим манифест ```hostpath-share.yml```, который запускает два пода "писателя" и "читателя". 

```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-writer
  labels:
    app.kubernetes.io/name: hostpath-writer
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  containers:
  - name: writer
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c"]
    args:  
    - |
      while true; do
        echo "Writer says hello at $(date)" >> /data/shared.log;
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
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    hostPath:
      # Путь на хост-машине
      path: /tmp/k8s-book-shared
      # Kuberntes создаст директорию, если её нет.
      type: DirectoryOrCreate
  restartPolicy: Never
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-reader
  labels:
    app.kubernetes.io/name: hostpath-reader
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/name
            operator: In
            values:
            - hostpath-writer
        topologyKey: kubernetes.io/hostname
  containers:
  - name: reader
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c"]
    args:  
    - |
      while [ ! -f /data/shared.log ]; do sleep 2; done;
      tail -f /data/shared.log
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    hostPath:
      # Путь на хост-машине
      path: /tmp/k8s-book-shared
      # Kuberntes создаст директорию, если её нет.
      type: DirectoryOrCreate
  restartPolicy: Never

```

Здесь мы использовали ```podAffinity```, чтобы планировщик Kubernetes разместил под ```hostpath-reader``` на том же узле, где уже работает под с меткой ```app.kubernetes.io/name: hostpath-writer```. 

Применим манифест:  
```sh
kubectl -n work apply -f hostpath-share.yml
```
```sh
pod/hostpath-writer created
pod/hostpath-reader created
```  

Запустим просмотр логов "читателя":  
```sh
kubectl -n work logs -f hostpath-reader
```  
Вывод логов будет примерно так:  
```sh
Writer says hello at Sun Apr 26 12:34:40 UTC 2026
Writer says hello at Sun Apr 26 12:34:45 UTC 2026
Writer says hello at Sun Apr 26 12:34:50 UTC 2026
Writer says hello at Sun Apr 26 12:34:55 UTC 2026
Writer says hello at Sun Apr 26 12:35:00 UTC 2026
Writer says hello at Sun Apr 26 12:35:05 UTC 2026
Writer says hello at Sun Apr 26 12:35:10 UTC 2026
Writer says hello at Sun Apr 26 12:35:15 UTC 2026
Writer says hello at Sun Apr 26 12:35:20 UTC 2026
Writer says hello at Sun Apr 26 12:35:25 UTC 2026
```  

Это подтверждает, что "читатель" видит данные, которые записывает "писатель" в общую директорию на хосте.

Удалим поды и очистим данные: 
```sh
# Сначала узнаем, на каком узле работал под-писатель
NODE_NAME=$(kubectl -n work get pod hostpath-writer -o jsonpath='{.spec.nodeName}')

# Удаляем поды
kubectl -n work delete -f hostpath-share.yml --force

# Удаляем директорию с данными на узле:
ssh "root@${NODE_NAME}" -- sudo rm -rf /tmp/k8s-book-shared && \ 
echo "Cleaned up data on $NODE_NAME"
```  

## Риски и ограничения hostPath  

Использование ```hostPath``` сопряжено со значительными рисками, которые необходимо понимать. 

### Потеря данных при перезапуске пода на другом узле  

Данные, сохраненные через ```hostPath```, привязаны к конкретному узлу. Если узел выходит из строя или под по какой-либо причине переезжает на другой узел, данные будут утеряны.

Продемонстрируем это с помощью ```Deployment``` из манифеста ```hostpath-fail.yml```. Чтобы гарантировать, что после удаления под будет перезапущен на другом узле, мы добавим правило ```podAntiAffinity```. Это правило запретит планировщику размещать новый под на том же узле, где уже есть(или только что был удален) под с такой же меткой.  

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostpath-fail-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hostpath-fail
  template:
    metadata:
      labels: 
        app: hostpath-fail
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                    - hostpath-fail
              topologyKey: kubernetes.io/hostname
      containers:
      - name: main
        image: busybox:1.37
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo "Pod started on node $(NODE_NAME) at $(date)" >> /data/node-specific-data.log;
          echo "---Current content on node-specific-data.log ---";
          cat /data/node-specific-data.log;
          echo "---";
          sleep infinity;
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: host-data
          mountPath: /data
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
      volumes:
      - name: host-data
        hostPath:
          path: /tmp/k8s-hostpath-fail
          type: DirectoryOrCreate      

```  

Применим манифест:  
```sh
kubectl -n work apply -f hostpath-fail.yml
```  
```sh
deployment.apps/hostpath-fail-demo created
```  

Посмотрим лог пода, чтобы увидеть, на каком узле он запустился и какие данные записал.

```sh
# Находим имя пода
POD_NAME=$(kubectl -n work get pods -l app=hostpath-fail -o jsonpath='{.items[0].metadata.name}') && \
echo $POD_NAME

#Смотрим лог
kubectl -n work logs $POD_NAME
```  
```sh
---Current content on node-specific-data.log ---
Pod started on node worker1 at Sun Apr 26 12:57:01 UTC 2026
---
```  

Теперь имитируем сбой, выполнив перезапуск ```Deployment```. Контроллер остановит старый под и создаст новый. Благодаря ```podAntiAffinity```, он будет вынужден выбрать для него другой узел.

```sh
kubectl -n work rollout restart deployment/hostpath-fail-demo
```  
```sh
deployment.apps/hostpath-fail-demo
```  
А теперь повторяем наши команды:  
```sh
# Находим имя пода
POD_NAME=$(kubectl -n work get pods -l app=hostpath-fail -o jsonpath='{.items[0].metadata.name}') && \
echo $POD_NAME

#Смотрим лог
kubectl -n work logs $POD_NAME
```
```sh
---Current content on node-specific-data.log ---
Pod started on node worker2 at Sun Apr 26 13:07:08 UTC 2026
---
```

И видим, что под создан на другой ноде и в ней только одна запись. А так же изменилась нода на которой запущен под.  

Не забудем удалить за собой ```Deployment``` и очистить данные, оставшиеся на узлах.  

```sh
# Удаляем Deployment
kubectl -n work delete -f hostpath-fail.yml

# Поскольку под мог работать на любом из узлов кластера, 
# выполним команду очистки на всех известных узлах
for i in {1..2}; do
  NODE_NAME="worker${i}"
  echo "--- Cleaning up on node: $NODE_NAME ---"
  ssh "root@${NODE_NAME}"  -- rm -rf /tmp/k8s-hostpath-fail
done
```

## Проблемы безопасности

```hostPath``` предоствляет контейнеру прямой доступ к файловой системе узла. Это **серьезная угроза безопасности** и его использование должно быть строго ограничено.  

  1. **Доступ к ситемным файлам**: Вредносное или скомпрометированное приложение может получить доступ к чувствительным данным на хосте, напрмер к ```/etc/shadow```
  2. **Изменение конфигурации узла**: Если смонтировать системные директории в режиме ```read-write```, приложение сможте изменить конфигурацию узла, что повлияет на все остальные поды.
  3. **Доступ к docker-сокету**: Монтирование ```/var/run/docker.sock``` позволяет контейнеру управлять Docker-демоном на хосте, что равносильно получению root-доступа ко всему узлу.

**Пример: побег из контейнера через /proc**  

Рассмотрим самый опасный сценарий. Создадим под, который запускается от имени ```root``` и монтируюет директорию ```/proc``` с хост-машины, как описано в манифесте  ```hostpath-security-risk.yml```. Файловая система ```proc``` в Linux предоставляет интерфейс к структурам данных ядра и ее монтирование дает огромное возможности для атки.  

```yml
# ВНИМАНИЕ: Этот манифест демонстрирует КРАЙНЕ небезопасную конфигурацию.
# НЕ ИСПОЛЬЗУЙТЕ его в производственной среде
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-security-risk
  labels:
    app.kubernetes.io/name: hostpath-security-risk
spec:
  containers:
  - name: main
    image: busybox:1.37
    command: ["/bin/sh", "-c", "sleep 1h"]
    securityContext:
       # Запускаем контейнер от имен root
       runAsUser: 0
    volumeMounts:
    - name: host-proc
      mountPath: /proc
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
  volumes:
  - name: host-proc
    hostPath:
      path: /proc
      type: Directory

```  

Применим манифест:  
```sh
kubectl -n work apply -f hostpath-security-risk.yml
```
```sh
pod/hostpath-security-risk created
```  

После запуска такого пода злоумышленник, получивший доступ к контейнеру, может выполнить следующией действия:  

```sh
# Заходим в контейнер
kubectl -n work exec -it hostpath-security-risk -- /bin/sh

# Внутри контейнера:
# Получаем доступ к процессам хоста
ls -l /host-proc

# Это даст нам список всех процессов, запущенных на УЗЛЕ, а не внутри контейнера.
# Зная это, можно, напрмер попытаться внедрить код в другой процесс
# или прочитать из памяти чувствительные данные.
```  

Это пример наглядно показывает, как ```hostPath``` в сочетании с высокими привилегями (```runAsUser: 0```) полностью нарушает изоляцию контейнера и создает критическую уязвимость в безопасности кластера.  

Удалим под:  

```sh
kubectl -n work delete -f hostpath-security-risk.yml 
```  

## Защита с помощью Admission Controllers  

Чтобы предотвратить развертывание таких опасных конфигураций, администраторы кластера должны использовать ```admission controllers``` - специальные компоненты Kubernetes, которые могут перехватить запросы к API-серверу до того, как объект будет сохранен в etcd.  

Современные инструменты, такие как **Kyverno** или **OPA Gatekeeper**, позволяют определять гибкие политики безопасности. Напрмер, можно создать политику, которая полностью запрещает ```hostPath``` или разрешает его использование только для определенных, заранее одобренных путей.  

Вот пример политики для Kyverno, которая запрещает любые ```hostPath```-тома, кроме тех, что ведут в ```/var/log/my-app```:  

```yml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-hostpath-volumes
spec:
  validationFailureAction: Enforce
  rules:
    - name: "restrict-hostpath"
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Использование hostPath запрещено, кроме разрешенного пути /var/log/my-app."
        pattern:
          spec:
            =(volumes):
              - =(hostPath):
                  path: "/var/log/my-app"
```  

Такая политика автоматически отклонит любой под, который пытается смонтировать, напрмер ```/proc``` или ```/etc```, тем самым защищая кластер на уровне API.

## Рекомендации по использованию  

Несмотря на все недостатки ```hostPath``` может быть полезен в строго определнных сценариях:

  - **Системные агенты**: Для приложений, которым по своей природе нужен доступ к файлам хоста, Напрмер, агенты мониторинга(Prometheus node-exporter), сборщики лгов(Fluentd, Logstash) или агенты безопасности.  
  - **Одноузловые кластеры**: В средах для разработки и тестирования(например Minikube, kind), где есть только один узел, ```hostPath``` может быть простым способом сохранить данные.  
  - **Инициализация узла**: Для выполнения задач, которые должны что-то сконфигурировать на узле один раз.  

При использование ```hostPath``` всегда следуйте этим правилам:  

  - **Используйте ```nodeSelector``` или ```nodeAffinity```**: Жестко привязывайте под к конкретному узлу, чтобы избежать проблем потери данных.  
  - **Используйте ```readOnly: true```**: Если приложению нужен доступ только на чтение, всегда указывайте это явно.   
  - **Ограничивайте доступ**: Никогда не монтируйте корневую директорию (```/```) или критически важные системные пути. Используйте как можно более конкретные и ограниченные пути.  

Для большинства же задач, требующих постоянного хранения данных, следует использовать более надержные и гибки решения, такие как ```PersistentVolume``` и ```PersistentVolumeClaim``` в связке с сетевыми хранилищами(NFS,Ceph или облачные диски).