# Gateway API

Gatewai API - стандарт Kubernetes для управления входящим трафиком, пришедший на замену Ingress. Cilium реализует Gateway API с помощью встронного прокси, без необходимости устанвливать отдельный Ingress Controller(nginx, traefik и т.д.).  

## Gateway API vs Ingress

| **Характеристика** | **Ingress** | **Gateway API** |
| :--- | :--- | :--- |
| Ролевая модель | Один ресурс | GatewayClass -> Gateway -> Route (разделение ответсвтенности) |
| Протоколы | HTTP/HTTPS | HTTP, HTTPS, gRCP, TCP, TLS |
| Маршрутизация | Хост + путь | Хост, путь , заголовки, query-параметры |
| Стандартизация | Аннотации зависят от реализации | Единный API для всех реализаций |
| Статус | Stable, но ограничен | Stable (v1), активно развивается |

## Предварительные требования

Для работы Gateway API необходимы:  

  - **CRD Gateway API** - определение ресурсов.  
  - **Включённая поддержка в Cilium** - параметр ```gatewayAPI.enabled```.  
  - **L2 Announcement или BGP** - для назначения IP шлюзу(Gateway создаёт сервис типа LoadBalancer).  

## Установка CRD

Gateway API CRD не входит в состав Kubernetes и устанавливается отдельно:  

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_gateways.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
```
```sh
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
```

Проверяем:  

```sh
kubectl  api-resources | grep gateway
```
```sh
gatewayclasses                      gc                                  gateway.networking.k8s.io/v1        false        GatewayClass
gateways                            gtw                                 gateway.networking.k8s.io/v1        true         Gateway
grpcroutes                                                              gateway.networking.k8s.io/v1        true         GRPCRoute
httproutes                                                              gateway.networking.k8s.io/v1        true         HTTPRoute
referencegrants                     refgrant                            gateway.networking.k8s.io/v1beta1   true         ReferenceGrant
```

## Включение Gateway API в Cilium 

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --namespace kube-system \
  --reuse-values \
  --set gatewayAPI.enabled=true
```

После обновления необходимо перезапустить агенты Cilium и оператор, чтобы они применили новыую конфигурацию и обнаружили CRD:  

```sh
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout status daemonset/cilium -n kube-system --timeout=120s
```

У меня возникли проблемы при рестарте Daemon set из-за отсутсвия Cilium Envoy CDR. Для решения проблемы добавил недостающие CDR файл ```manifists/cilium-envoy-crds.yml```
```yml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: ciliumenvoyconfigs.cilium.io
spec:
  group: cilium.io
  names:
    kind: CiliumEnvoyConfig
    listKind: CiliumEnvoyConfigList
    plural: ciliumenvoyconfigs
    singular: ciliumenvoyconfig
    shortNames:
    - cec
  scope: Namespaced
  versions:
  - name: v2
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        x-kubernetes-preserve-unknown-fields: true
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: ciliumclusterwideenvoyconfigs.cilium.io
spec:
  group: cilium.io
  names:
    kind: CiliumClusterwideEnvoyConfig
    listKind: CiliumClusterwideEnvoyConfigList
    plural: ciliumclusterwideenvoyconfigs
    singular: ciliumclusterwideenvoyconfig
    shortNames:
    - ccec
  scope: Cluster
  versions:
  - name: v2
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        x-kubernetes-preserve-unknown-fields: true
```

После применения указанных CDR:

```sh
kubectl apply -f manifests/cilium-envoy-crds.yml
```

Проверка:

```sh
kubectl api-resources | grep cilium
```
```sh
ciliumcidrgroups                    ccg                                 cilium.io/v2                        false        CiliumCIDRGroup
ciliumclusterwideenvoyconfigs       ccec                                cilium.io/v2                        false        CiliumClusterwideEnvoyConfig
ciliumclusterwidenetworkpolicies    ccnp                                cilium.io/v2                        false        CiliumClusterwideNetworkPolicy
ciliumendpoints                     cep,ciliumep                        cilium.io/v2                        true         CiliumEndpoint
ciliumenvoyconfigs                  cec                                 cilium.io/v2                        true         CiliumEnvoyConfig
ciliumidentities                    ciliumid                            cilium.io/v2                        false        CiliumIdentity
ciliuml2announcementpolicies        l2announcement                      cilium.io/v2alpha1                  false        CiliumL2AnnouncementPolicy
ciliumloadbalancerippools           ippools,ippool,lbippool,lbippools   cilium.io/v2                        false        CiliumLoadBalancerIPPool
ciliumnetworkpolicies               cnp,ciliumnp                        cilium.io/v2                        true         CiliumNetworkPolicy
ciliumnodeconfigs                                                       cilium.io/v2                        true         CiliumNodeConfig
ciliumnodes                         cn,ciliumn                          cilium.io/v2                        false        CiliumNode
ciliumpodippools                    cpip                                cilium.io/v2alpha1                  false        CiliumPodIPPool
```

А теперь повторный рестарт Daemonset:

```sh
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout status daemonset/cilium -n kube-system --timeout=120s
```
```sh
daemon set "cilium" successfully rolled out
```
```sh
kubectl rollout restart deployment/cilium-operator -n kube-system
kubectl rollout status deployment/cilium-operator -n kube-system --timeout=120s
```
```sh
deployment "cilium-operator" successfully rolled out
```

## Ресурсы Gateway API

Gateway API состоит из трёх уровней ресурсов:

```txt
┌──────────────────────────┐
│  GatewayClass            │  Кто предоставляет шлюз (Cilium)
│  (администратор кластера)│
└───────────┬──────────────┘
            │
┌───────────▼──────────────┐
│  Gateway                 │  Точка входа (IP, порты, протоколы)
│  (администратор сети)    │
└───────────┬──────────────┘
            │
┌───────────▼──────────────┐
│  HTTPRoute / GRPCRoute   │  Правила маршрутизации
│  (разработчик)           │
└──────────────────────────┘
```

## GatewayClass

GatewayClass определяет, какой контроллер обрабатывает шлюзы. Для Cilium создадим GatewayClass с контроллером ```io.cilium/gateway-controller```. 

Файл ```gateway/01-gateway-class.yml```:

```yml
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
```
```sh
gatewayclass.gateway.networking.k8s.io/cilium created
```

Проверяем

```sh
kubectl get gatewayclass
```
```sh
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       36s
```

Статус `ACCEPTED: TRUE` - Cilium готов обрабатывать шлюзы этого класса.

### Gateway

Gateway - точка входа трафика. При создании Gateway, Cilium автоматически создаёт сервис типа `LoadBalancer`, который получает API из пула.

Файл `02-gateway.yml`:  

```yml
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
```

Параметры:

  - **gatewayClassName: cilium** - ссылка на GatewayClass.  
  - **listeners**- список слушателей(порты и протоколы).  
  - **allowedRoutes.namespaces.from: Same** - принимать маршруты только из того же namespace.  

```sh
kubectl apply -f gateway/02-gateway.yml
```
```sh
gateway.gateway.networking.k8s.io/cilium-gateway created
```

Проверим Gateway:

```sh
kubectl get gateway
```
```sh
NAME             CLASS    ADDRESS          PROGRAMMED   AGE
cilium-gateway   cilium   192.168.122.10   True         31s`
```

## Тестовые приложения

Для проверки маршрутизации создадим два простых приложения: 

Файл `gateway/03-gateway-apps.yml`:

```yml
---
# Приложение 1
apiVersion: v1
kind: Pod
metadata:
  name: app1
  labels:
    app.kubernetes.io/name: &name1 app1
    app.kubernetes.io/version: &version "1.27.5"
spec:
  initContainers:
  - name: init
    image: busybox:1.37
    command: ['sh', '-c', 'echo "<h1>App1</h1>" > /html/index.html']
    volumeMounts:
    - name: html
      mountPath: /html
    resources:
      limits:
        memory: "64Mi"
        cpu: "100m"
      requests:
        memory: "32Mi"
        cpu: "50m"
  containers:
  - name: nginx
    image: nginx:1.27.5
    ports:
      - containerPort: 80
        name: http
        protocol: TCP
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    resources:
      limits:
        memory: "128Mi"
        cpu: "200m"
      requests:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: html
    emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: app1-svc
  labels:
    app.kubernetes.io/name: &name1 app1
spec:
  selector:
    app.kubernetes.io/name: *name1
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
  type: ClusterIP
---
# Приложение 2
apiVersion: v1
kind: Pod
metadata:
  name: app2
  labels:
    app.kubernetes.io/name: &name2 app2
    app.kubernetes.io/version: &version "1.27.5"
spec:
  initContainers:
  - name: init
    image: busybox:1.37
    command: ['sh', '-c', 'echo "<h1>App2</h1>" > /html/index.html']
    volumeMounts:
    - name: html
      mountPath: /html
    resources:
      limits:
        memory: "64Mi"
        cpu: "100m"
      requests: 
        memory: "32Mi"
        cpu: "50m"
  containers:
  - name: nginx
    image: nginx:1.27.5
    ports:
      - containerPort: 80
        name: http
        protocol: TCP
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    resources:
      limits:
        memory: "128Mi"
        cpu: "200m"
      requests:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: html
    emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: app2-svc
  labels:
    app.kubernetes.io/name: &name2 app2
spec:
  selector:
    app.kubernetes.io/name: *name2
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
  type: ClusterIP
```
```sh
kubectl apply -f gateway/03-gateway-apps.yml
```
```sh
pod/app1 created
service/app1-svc unchanged
pod/app2 configured
service/app2-svc unchanged
```

### Маршрутизация по имени хоста

HTTPRoute позволяет направлять трафик в разные приложения на основе имени хоста (Заголовок Host).

Файл `gateway/04-gateway-multi-route.yml`:

```yml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app1-route
spec:
  parentRefs:
  - name: cilium-gateway
  hostnames:
  - "test1.rogov.lan"
  rules:
  - matches:
      - path:
          type: PathPrefix
          value: "/"
    backendRefs:
    - name: app1-svc
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app2-route
spec:
  parentRefs:
  - name: cilium-gateway
  hostnames:
  - "test2.rogov.lan"
  rules:
  - matches:
      - path:
          type: PathPrefix
          value: "/"
    backendRefs:
    - name: app2-svc
      port: 80 
```
```sh
kubectl apply -f gateway/04-gateway-multi-route.yml
```
```sh
httproute.gateway.networking.k8s.io/app1-route created
httproute.gateway.networking.k8s.io/app2-route created
```

Проверяем созданные HTTP-роуты: 

```sh
kubectl get httproutes
```
```sh
NAME         HOSTNAMES             AGE
app1-route   ["test1.rogov.lan"]   11s
app2-route   ["test2.rogov.lan"]   11s
```

### Проверка

DNS-имена `test1.rogov.lan` и `test2.rogov.lan` должны указывать на IP Gateway(`192.168.122.10`). Можно настроить DNS или добавить записи в `/etc/hosts`:

```sh
192.168.122.10 test1.rogov.lan test2.rogov.lan
```

Проверим маршрутизацию с помощью заголовка `HOST`:

```sh
curl -s http://192.168.122.10 -H "Host: test1.rogov.lan"
```
```sh
<h1>App1</h1>
```

Аналогично и для второго приложения:  

```sh
curl -s http://192.168.122.10 -H "Host: test2.rogov.lan"
```
```sh
<h1>App2</h1>
```

## Маршрутизация по пути

HTTPRoute  поддерживает маршрутизацию по URL-пути. Создадим маршрут, который направляет `/app1` в одно приложение, а `/app2` - в другое.  

Файл `05-path-route.yml`:

```yml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: path-route
spec:
  parentRefs:
  - name: cilium-gateway
  hostnames:
  - "test3.rogov.lan"
  rules:
  - matches:
      - path:
          type: PathPrefix
          value: "/app1"
    backendRefs:
      - name: app1-svc
        port: 80
    filters:
      - type: URLRewrite
        urlRewrite:
          path:
            type: ReplacePrefixMatch
            replacePrefixMatch: "/"
  - matches:
      - path:
          type: PathPrefix
          value: "/app2"
    backendRefs:
      - name: app2-svc
        port: 80
    filters:
      - type: URLRewrite
        urlRewrite:
          path:
            type: ReplacePrefixMatch
            replacePrefixMatch: "/"
```

## Маршрутизация по заголовкам

HTTPRoute может направлять трафик на основе HTTP-заголовков.

Файл `06-header-route.yml`:

```yml
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: header-route
spec:
  parentRefs:
  - name: cilium-gateway
  hostnames:
  - "test4.rogov.lan"
  rules:
  - matches:
      - headers:
          - name: X-Version
            value: v2
    backendRefs:
      - name: app2-svc
        port: 80
  - backendRefs:
      - name: app1-svc
        port: 80
```

Запросы с заголовком `X-Version: v2` направляются в app2, всё остальное - в app1

## Очистка

```sh
kubectl delete -f gateway/05-path-route.yml 
kubectl delete -f gateway/04-gateway-multi-route.yml
kubectl delete -f gateway/03-gateway-apps.yml
kubectl delete -f gateway/02-gateway.yml 
```

> **Примечание**: GatewayClass и CRD оставляем - они понадобятся в дальнейшей работе кластера