# Манифесты

- [Манифесты](#манифесты)
  - [Namespace](#namespace)
  - [Pod](#pod)
    - [Создание Pod](#создание-pod)
    - [Состояния](#состояния)
      - [Состояние пода](#состояние-пода)
      - [Состояние контейнера](#состояние-контейнера)
    - [Итоговый манифест пода](#итоговый-манифест-пода)
    - [Хороший манифест](#хороший-манифест)
      - [Метки](#метки)
      - [Образ контейнера](#образ-контейнера)
      - [Ресурсы](#ресурсы)
      - [Пробы](#пробы)
    - [Дополнительные контейнеры](#дополнительные-контейнеры)
      - [Init Containers](#init-containers)

## Namespace

[Документация](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).

Namespace используются для изоляции групп ресурсов в пределах одного кластера kubernetes. Имена ресурсов должны быть уникальными в пределах namespace.

Несмотря на то, что namespace по смыслу похож на директорию, в которой хранятся фалы (ресурсы). В отличии от директорий, namespaces не могут быть вложены друг в друга. Т.е. мы получаем набор namespaces на одном уровне.

Не имеет смысла использовать namespaces для разделения версий одного и того же приложения. Для этих целей обычно используют метки (labels).

По умолчанию, при создании кластера создаются четыре namespace.

```shell
kubectl get ns
```

```txt
NAME               STATUS   AGE
default            Active   24m
kube-node-lease    Active   24m
kube-public        Active   24m
kube-system        Active   24m
```

* default - namespace по умолчанию. Если при создании ресурса явно не указывать namespace, где он будет размещён. Ресурс будет создан в namespace default. **Не удаляйте и не используйте в работе этот namespace**.
* kube-node-lease - в этом пространстве имен хранятся объекты Lease, связанные с каждой нодой кластера. Lease позволяют kubelet отправлять сигналы, чтобы control plane могла обнаруживать сбои в работе нод.

```shell
kubectl -n kube-node-lease get leases
```

```txt
NAME                HOLDER              AGE
cr1.kryukov.local   cr1.kryukov.local   27m
wr1.kryukov.local   wr1.kryukov.local   25m
wr2.kryukov.local   wr2.kryukov.local   25m
```

* kube-public - этот namespace доступен для чтения всем клиентам (в том числе не прошедшим аутентификацию).
* kube-system - в этом namespace располагаются приложения control-plane.

Создадим namespace, в котором мы будем создавать наши объекты.

```shell
kubectl create ns work
```

```shell
kubectl get ns
kubectl get ns work
```

## Pod

Pod - это наименьший компонент, содержащий коллекцию контейнеров. Он может содержать как один, так и более контейнеров, имеющих общий доступ к volumes и сетевым ресурсам.

### Создание Pod

Попробуем создать pod в нашем namespace `work`. В качестве приложения будем использовать контейнере с nginx. Создадим файл `00-pod.yaml` с его описанием:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  namespace: work
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

Сразу возникает вопрос: откуда я знаю, какая конструкция у файла конфигурации? Ответ прост - из документации. Документация Kubernetes содержит все необходимые сведения о каждом объекте и его полях и значениях. Идем в Интернет и читаем [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/). Дальше найти информацию о том, какие поля можно использовать в конфигурации пода, достаточно просто.

Теперь пошлем наше пожелание о создании пода в кластер:

```shell
kubectl apply -f manifests/00-pod.yaml
```

Проверим, что pod создался:

```shell
kubectl get pods -n work
```

Вы должны увидеть что-то вроде этого:

```txt
NAME           READY   STATUS    RESTARTS   AGE
my-first-pod   1/1     Running   0          118s
```

### Состояния

#### Состояние пода

Теперь давайте проверим состояние нашего пода:

```shell
kubectl describe pod my-first-pod -n work
```

Вы должны увидеть информацию о поде, включая его контейнеры, IP-адрес и другие параметры. Вот пример вывода:

```txt
Name:             my-first-pod
Namespace:        work
Priority:         0
Service Account:  default
Node:             wr2.kryukov.local/192.168.218.135
Start Time:       Sat, 31 May 2025 10:36:52 +0300
Labels:           <none>
Annotations:      cni.projectcalico.org/containerID: eb013296b9ce4727c96904a797fb34cd20199cf48987e2421286047d932167d2
                  cni.projectcalico.org/podIP: 10.233.124.132/32
                  cni.projectcalico.org/podIPs: 10.233.124.132/32
Status:           Running
IP:               10.233.124.132
IPs:
  IP:  10.233.124.132
Containers:
  nginx:
    Container ID:   containerd://ff3ed1e2766e3a2d7e892b7adffa46db6b27901adce9ca2965d24d9edea3477d
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:fb39280b7b9eba5727c884a3c7810002e69e8f961cc373b89c92f14961d903a0
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 31 May 2025 10:37:05 +0300
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        250m
      memory:     64Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8ns9g (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-8ns9g:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m26s  default-scheduler  Successfully assigned work/my-first-pod to wr2.kryukov.local
  Normal  Pulling    3m25s  kubelet            Pulling image "nginx:latest"
  Normal  Pulled     3m13s  kubelet            Successfully pulled image "nginx:latest" in 12.086s (12.086s including waiting). Image size: 72402122 bytes.
  Normal  Created    3m13s  kubelet            Created container: nginx
  Normal  Started    3m13s  kubelet            Started container nginx
```

Обратите внимание на поле `Status`. В нем показан текущий статус пода.

При создании пода он проходит через определенные состояния:

- **Pending** - под был принят кластером Kubernetes, но один или несколько контейнеров не были настроены и не готовы к запуску. Сюда входит время, которое модуль проводит в ожидании планирования, а также время, затрачиваемое на загрузку образов контейнеров по сети.;
- **Running** - все контейнеры в поде успешно запущены и работают или по крайней мере один контейнер все еще работает или находится в процессе запуска или перезапуска;
- **Succeeded** - все контейнеры в поде успешно завершены и не будут запущены снова;
- **Failed** - Все контейнеры в поде завершены, и по крайней мере один контейнер завершил работу с ошибкой. То есть контейнер либо завершил работу с ненулевым статусом, либо был завершён системой и не настроен на автоматический перезапуск;
- **Unknown** - состояние пода не может быть определено. Это происходит, когда не удается связаться с API Kubernetes для получения информации о поде (например, из-за проблем с сетью).

Так же рекомендуется внимательно смотреть на раздел `Events`. Он содержит события, которые произошли в кластере Kubernetes. Эти события могут помочь понять, почему поду не удается запустить. Например: `FailedCreatePodSandBox`, `FailedMount`, `ErrImagePull`.

#### Состояние контейнера

Кроме состояния пода, kubernetes отслеживает состояние каждого контейнера внутри пода. В выводе команды describe, есть отдельный раздел, посвященный контейнерам. У каждогок контейнера есть состояние, показанное в поле `State`. Это поле может иметь значение:

- `Waiting` - контейнер ждет запуска, пока не будет выполнено что-то, например, не хватает памяти или CPU. Причина (`Reason`), почему контейнер находится в этом состоянии показана в разделе `Events`.
- `Running` - контейнер запущен и выполняется.
- `Terminated` - контейнер завершен, и можно посмотреть причину завершения, прочитав поле `Reason`.

### Итоговый манифест пода

Интересно посмотреть на то, как в итоге выглядит манифест пода, после его запуска. Напомню, что после добавления манифеста пода при помощи программы `kubectl apply`, Kubernetes создает объект в API сервера. Затем его обрабатывает контроллер, потом запускается планировщик и только после этого под запускается при помощи `kubelet`. Какждое из приложений добавляет свою информацию в манифест. Давайте рассмотрим его:

```shell
kubectl get pod my-first-pod -n work -o yaml | less
```

Как видите, в итоговом манифесте появились новые поля. Все они тоже описаны в документации [Kubernetes API/ Workloads/ Pods](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/).

### Хороший манифест

Минимальный манифест пода может содержать только информацию о контейнере. Но если в дальнейшем вы хотите получать вменяемый результат от работы кластера, рекомендуется соблюдать базовый набор правил формирования манифестов.

Метки (labels), явное указание tag контейнера, ресурсы, пробы и `securityContext` помогут вам создать "правильный" манифест пода.

Ниже мы рассмотрим пример формирования манифеста пода, за исключением `securityContext`. Но к `securityContext` мы обязательно вернемся позднее.

#### Метки

Метки (labels) — это пары ключ-значение, которые можно использовать для группировки объектов Kubernetes. Они помогают управлять объектами и получать к ним доступ с помощью селекторов (selectors). Сначала может показаться, что они служат больше для комментариев и удобства, чем для функционала. Но в дальнейшем они будут регулярно использоваться при ссылке на объекты в kubernetes.

Я бы рекомендовал вам создавать как минимум две метки: содержащие название приложения и его версию. Например:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  namespace: work
  labels:
    app: nginx
    version: 1.27.5
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Можно добавлять любые другие метки, по мере необходимости. Работе они не помешают, но помогут в дальнейшем при управлении и мониторинге кластера. Боле того, для разных ресурсов в кластере существует определенный набор, рекомендуемых к использованию меток, с которыми мы познакомимся по мере изучения материала.

#### Образ контейнера

**Никогда не используйте тег `latest`!** В нашем предыдущем примере мы указали `nginx:latest`. Это очень, очень плохая практика. Почему? Потому что вы не знаете, какая версия образа будет установлена в вашем кластере. При обновлении/рестарте пода, неожиданно может прилететь новый образ, который может содержать ошибки. Или не поддерживать текущие конфигурационные параметры или функционал. Всегда явным образом указывайте тег контейнера, который вы используете.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  namespace: work
  labels:
    app: nginx
    version: 1.27.5
spec:
  containers:
    - name: nginx
      image: nginx:1.27.5
      ports:
        - containerPort: 80
```

#### Ресурсы

Всегда указывайте ресурсы для ваших контейнеров! Это поможет избежать проблем с ресурсами в кластере, ваш контейнер не сможет "съесть" все ресурсы кластера. Уж лучше ваше приложение убьёт OOMKill, чем упадет нода кластера. Выше, в нашем примере мы указали `requests` и `limits` для памяти и CPU. В дальнейшем, когда мы будем рассматривать планировщик, приоритеты и QOS, вы увидите, как подбор параметров будет влиять на жизненный цикл вашего пода.

Сейчас же, мы явным образом указываем системе, сколько памяти и CPU нужно для работы контейнера.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  namespace: work
  labels:
    app: nginx
    version: 1.27.5
spec:
  containers:
    - name: nginx
      image: nginx:1.27.5
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

#### Пробы

Хороший манифест обязательно содержит описание проб, при помощи которых kubernetes может проверять состояние вашего приложения.

Существует четыре механизма для реализации проверок:

- `exec` - выполнение командной строки внутри контейнера и возврат кода завершения (0 - успех, не 0 - ошибка).
- `tcpSocket` - отправка TCP-запроса на указанный порт. Если соединение установлено, то проверка успешна.
- `httpGet` - выполнение HTTP-запроса по указанному пути и порту. Проверка будет успешной, если код ответа будет в диапазоне 200-399.
- `grpc` - использование grpc для проверки состояния приложения (требуется дополнительная настройка).

Каждая проба возвращает Success, Failure или Unknown. 

Для проверки состояния можно использовать три типа проб:

- `livenessProbe` - Указывает, работает ли контейнер. Если проверка работоспособности не пройдена, `kubelet` убивает контейнер, и в дальнейшем учитывается restart policy. Если livenessProbe не определена, состояние по умолчанию — Success.
- `readinessProbe` - Указывает, готов ли контейнер отвечать на запросы по сети. Если проверка готовности не пройдена, контроллер EndpointSlice удаляет IP-адрес модуля из EndpointSlices всех сервисов, соответствующих модулю (*об том, что такое Service, Endpoint, EndpointSlice мы подробно поговорим позднее*). Состояние готовности по умолчанию до первоначальной задержки — Failure. Если readinessProbe не определена, состояние по умолчанию — Success.
- `startupProbe` - Указывает, запущено ли приложение в контейнере. Если определена startupProbe, до тех пор, пока она не завершится успешно, все остальные проверки отключаются. Если startupProbe завершается неудачно, kubelet завершает работу контейнера, и в дальнейшем учитывается restart policy. Если startupProbe не определена, состояние по умолчанию — Success. Этв проба применяется для приложений, которые имеют длительный период запуска (например, Java или .NET).

В нашем примере приложение запускается быстро, поэтому `startupProbe` не требуется. В качестве механизма реализации проверки можно использовать как `httpGet`, так и `tcpSocket`. В конкретном приложении, при использовании `httpGet`, каждый запрос пробы будет попадать в access логи nginx. Т.е. пробы будут мусорить в логах. Поэтому лучше использовать `tcpSocket`, для контроля открытия сетевого сокета приложением.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  namespace: work
  labels:
    app: nginx
    version: 1.27.5
spec:
  containers:
    - name: nginx
      image: nginx:1.27.5
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      livenessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 10
      readinessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 10
```

Попробуем применить новый манифест:

```shell
kubectl apply -f manifests/01-finalPod.yaml
```

Мы получим сообщение об ошибке. Мы внесли изменения в манифест пода в поля, которые нельзя изменять. В дальнейшем привыкайте к тому, что при серьезных изменениях проще удалить существующий под создать его заново. Конечно есть список полей, которые нельзя изменять, но проще удалить и создать, чем разбираться в том, что именно не так.

```shell
kubectl -n work delete pod my-first-pod
kubectl apply -f manifests/01-finalPod.yaml
```

Убедимся, что под запустился:

```shell
kubectl -n work get pods
```

Посмотрим логи приложения. Убедимся, что пробы не мусорят своими запросами в логи:

```shell
kubectl -n work logs my-first-pod 
```

Удалим под:

```shell
kubectl -n work delete pod my-first-pod
```

### Дополнительные контейнеры

#### Init Containers

Под может содержать несколько контейнеров. Это могут быть контейнеры выполняемые одновременно. Их список можно указать в секции `containers` манифеста.

Так же предусматривается возможность добавить контейнер(ы) который будет выполняться при старте пода ([Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)). Перед запуском основных контейнеров. Обычно приложения в таких контейнерах занимаются действиями по предварительной настройке.

Не буду тут приводит пример полного манифеста. Только часть, касающуюся init-контейнера:

```yaml
spec:
  initContainers:
    - name: init-myservice
      image: busybox:1.28
      command: 
        - 'sh'
        - '-c'
        - 'echo Initializing... && sleep 10'
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

Запустим под:

```shell
kubectl apply -f manifests/02-initContainer.yaml
```

Подождем немного и посмотрим логи init-контейнера:

```shell
kubectl -n work logs my-first-pod -c init-myservice
```

Посмотрите какой статус у init-контейнера:

```shell
kubectl -n work describe pod my-first-pod
```

Он завершил свою работу.

Удалим под:

```shell
kubectl -n work delete pod my-first-pod
```