# Service

Доступ к приложению в Kubernetes осуществляется через сервисы, которые предоставляют единую точку доступа к группе Pod'ов.

[Kubernetes API: Service](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/).

Пример конфигурации сервиса для нашего приложения:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-first-nginx
  namespace: work
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.27.5
spec:
  selector:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.27.5
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      name: http
```

Традиционно, два раздела: metadata и spec.

Cсылка на другие объекты в kubernetes осуществляется при помощи меток. Имена объектов используются кране редко. В spec определён раздел selector, который содержит условия выбора Pods для обслуживания сервисом.

В ports мы говорим, на каком порту (`port`) будет доступен наш сервис и куда он будет пересылать запросы (`targetPort`). Если явно не определить `protocol`, то по умолчанию используется TCP. Имя протокола не обязательно. Но его рекомендуют указывать. В дальнейшем мы можем использовать небольшие "трюки", связанные с ссылкой на порты в контейнерах, но об этом позже.

Существует несколько типов сервисов, определяемых при помощи поля `type`. Если его явно не указывать, то по умолчанию используется тип `ClusterIP`. Сейчас мы не будем рассматривать все возможные варианты. Подробно о типах сервисов и что за этими типами скрывается мы поговорим во второй части раздела о сетях в Kubernetes.

Добавил определение сервиса для нашего приложения непосредственно в манифест приложения. Обратите внимание, что согласно синтаксиса YAML, в одном файле можно определять несколько ресурсов, разделяя их тремя дефисами `---`. Точнее говоря, каждый ресурс должен начинаться с `---`, а заканчиваться `...`. Но завершающие точки для разграничения ресурсов не обязательны.

Применим манифест:

```shell
kubectl apply -f service.yaml
```
```txt
service/my-first-nginx created
deployment.apps/my-first-deployment created
```

Посмотрим, что получилось:

```shell
kubectl -n work get all
```
```txt
NAME                                      READY   STATUS    RESTARTS   AGE
pod/my-first-deployment-cb8d9fbf5-7hkqh   1/1     Running   0          72s
pod/my-first-deployment-cb8d9fbf5-84hl5   1/1     Running   0          72s
pod/my-first-deployment-cb8d9fbf5-skbbv   1/1     Running   0          72s

NAME                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/my-first-nginx   ClusterIP   10.233.7.82   <none>        80/TCP    72s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-first-deployment   3/3     3            3           72s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/my-first-deployment-cb8d9fbf5   3         3         3       72s
```

Как вы видите, `all` включает себя и сервисы.

Посмотрим на сервис подробнее:

```shell
kubectl -n work describe service my-first-nginx
```
```yaml
Name:                     my-first-nginx
Namespace:                work
Labels:                   app.kubernetes.io/name=nginx
                          app.kubernetes.io/version=1.27.5
Annotations:              <none>
Selector:                 app.kubernetes.io/name=nginx,app.kubernetes.io/version=1.27.5
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.233.7.82
IPs:                      10.233.7.82
Port:                     http  80/TCP
TargetPort:               80/TCP
Endpoints:                10.233.124.169:80,10.233.109.218:80,10.233.124.168:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

У сервиса есть свой собственный IP-адрес! По которому мы можем к нему обратиться. Правда существует небольшой нюанс, этот IP-адрес доступен только внутри кластера. Если вы попытаетесь подключиться к нему извне, то получите ошибку.

Зайдите на любую ноду кластера и попробуйте при помощи `curl` получить доступ к сервису:

```shell
curl 10.233.7.82
```
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Так же обратите внимание на поле `Endpoints`. При создании нашего типа сервиса, создаётся объект `Endpoints`, который хранит информацию о том, какие Pod доступны по данному сервису. Т.е. Endpoints - это просто список IP адресов и портов, подов. Имя endpoints совпадает с именем сервиса.

```shell
kubectl -n work get ep my-first-nginx
```
```txt
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME             ENDPOINTS                                               AGE
my-first-nginx   10.233.109.218:80,10.233.124.168:80,10.233.124.169:80   7m12s
```

Это классика. Но время не стоит на месте и вместо endpoints в новых версиях kubernetes рекомендуется использовать EndpointSlices, которые позволяют более гибко управлять сетевыми данными. Посмотрим как это выглядит:

Посмотрим какие EndpointSlice находятся в нашем namespace:

```shell
kubectl -n work get EndpointSlice
```
```txt
NAME                   ADDRESSTYPE   PORTS   ENDPOINTS                                      AGE
my-first-nginx-5m6cx   IPv4          80      10.233.124.169,10.233.109.218,10.233.124.168   8m57s
```

Посмотрим манифест EndpointSlice:

```shell
kubectl -n work get EndpointSlice my-first-nginx-5m6cx -o yaml
```
```yaml
addressType: IPv4
apiVersion: discovery.k8s.io/v1
endpoints:
- addresses:
  - 10.233.124.169
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: wr2.kryukov.local
  targetRef:
    kind: Pod
    name: my-first-deployment-cb8d9fbf5-84hl5
    namespace: work
    uid: c74fe988-ca1f-4fd7-a3f6-b3a589491e8c
- addresses:
  - 10.233.109.218
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: wr1.kryukov.local
  targetRef:
    kind: Pod
    name: my-first-deployment-cb8d9fbf5-7hkqh
    namespace: work
    uid: e025165b-c983-41f4-8fa7-abe9d2569783
- addresses:
  - 10.233.124.168
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: wr2.kryukov.local
  targetRef:
    kind: Pod
    name: my-first-deployment-cb8d9fbf5-skbbv
    namespace: work
    uid: b672a85b-0085-46de-be28-793b0ff52d2e
kind: EndpointSlice
metadata:
  annotations:
    endpoints.kubernetes.io/last-change-trigger-time: "2025-06-16T10:54:56Z"
  creationTimestamp: "2025-06-16T10:54:21Z"
  generateName: my-first-nginx-
  generation: 7
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.27.5
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
    kubernetes.io/service-name: my-first-nginx
  name: my-first-nginx-5m6cx
  namespace: work
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: my-first-nginx
    uid: 1a619219-0c6f-4547-896d-d16a7cd54513
  resourceVersion: "164424"
  uid: f99f367c-b913-43d8-a4fd-5404a139fd30
ports:
- name: http
  port: 80
  protocol: TCP
```

Массив `endpoints` содержит список ip адресов и портов подов. А также дополнительную информацию о состоянии пода.
В `metadata.ownerReferences` указано какой сервис является владельцем этого `EndpointSlice`.

Хотя, что то я полез далеко в глубину. Пока рано настолько глубоко закапываться в детали. Важно понять, что в нашем случае в соответствии сервису ставятся все поды, которые подходят по селектору (меткам). И дальше, глубоко в недрах Linux будет создано NAT преобразование, благодаря которому, по алгоритму round-robin соединения от клиентов будут равномерно распределяться между всеми подами.

Сейчас этот сервис живет внутри кластера и не выводится за его пределы. Что бы получить доступ к **сервису** извне, нам нужно создать ресурсы типа `Ingress` или воспользоваться механизмами, предоставляемыми `Gateway API`. Как минимум `Ingress` мы рассмотрим, когда будем разбираться с сетями kubernetes. А про Gateway API у меня есть несколько видео, рекомендуемых к просмотру после изучения базы kubernetes.

Также доступ к сервису со своей локальной машины можно получить через `kubectl port-forward`. Этот способ позволяет нам перенаправлять порты из контейнера на локальную машину. Вот пример команды:

```shell
kubectl -n work port-forward svc/my-first-nginx 8080:80 
```

Откройте новое окно терминала и выполните эту команду:

```shell
curl localhost:8080
```

Этот способ полезен для тестирования и отладки приложений, развернутых в Kubernetes. Но в обычной работе вы его использовать не будете.
Не забудьте нажать `Ctrl+C`, чтобы остановить перенаправление портов, когда закончите тестирование.