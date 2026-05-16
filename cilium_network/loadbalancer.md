# Cilium Load Balancer

В bare-metal кластера Kubernetes сервисы типа ```LoadBalancer``` по умолчанию не получают внейшний IP-адрес - для этого нужен внешний контроллер (например MetalLB). Cilium предоставляет встроенную поддержку LoadBalancer через два механизма: L2 Announcments и BGP Contorl Plane.

## L2 Announcements

L2 Announcements - аналог MetalLB в режиме L2. Cilium назначает IP-адрес сервису и отвечает на ARP-запросы на этот IP от имени одной из нод кластера. Для клиентов в локальной сети это выглядит так, будто IP принадлежит ноде.  

### Принцип работы

  1. Администратор создаёт **CiliumLoadBalancerPool** - пул IP-адресов для выделения.  
  2. Администратор создаёт **CiliumL2AnnouncememntPolicy** - политику, определяющую на каких интерфейсах анонсировать IP.  
  3. При создании сервиса типа ```LoadBalancer```, Cilium выделяет IP из пула.  
  4. Между агентами CIlium происходит **leader election**(через Kubernetes Leases) - одна нода становится ответственно за данный IP.  
  5. Нода-лидер отвечает на ARP-запросы за выделенный IP, используя свой MAC-адрес.  
  6. Входящий трафик попадет на ноду-лидер, далее eBPF перенаправляет его на под.

```txt
Клиент в локальной сети
        │
        │ ARP: Who has 192.168.218.180?
        ▼
┌──────────────────┐
│  r2 (leader)     │ ◄── Отвечает: "Это мой MAC"
│  Cilium Agent    │
│  eBPF LB ────────┼──── Перенаправляет трафик на под
└──────────────────┘     (может быть на другой ноде)
```

## Включение L2 Announcements

L2 Announcements включается через Helm:

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --namespace kube-system \
  --reuse-values \
  --set l2announcements.enabled=true \
  --set externalIPs.enabled=true
```

После обновления дождёмся перезапуска агентов:

```sh
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout status daemonset/cilium -n kube-system --timeout=120s
```

```txt
daemon set "cilium" successfully rolled out
```

## Пул IP-адресов

Создадим пул IP-адресов для LoadBalancer сервисов. Адреса должны находится в той же подсети, что и ноды кластера(в моем слчае это 192.168.122.0/24), и не пересекатся с адресами нод или другими устройствами в сети. 

Файл ```manifests/01-lb-ippool.yml```:

```yml
---
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: main-pool
spec:
  blocks:
    - start: "192.168.122.10"
      stop: "192.168.122.15"
```
```sh
kubectl apply -f cilium_network/manifests/01-lb-ippool.yml
```

Проверим пул:

```sh
kubectl get ciliumloadbalancerippools
```
```sh
NAME        DISABLED   CONFLICTING   IPS AVAILABLE   AGE
main-pool   false      False         6               85s
```

Политика определяет, на каких сетевых интерфейсах Cilium будет отвечать на ARP-запросы.

Файл ```manifests/02-lb-l2-policy.yml```:

```sh
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-policy
spec:
  interfaces:
    - ^ens[0-9]+
  externalIPs: true
  loadBalancerIPs: true
```

Параметры: 

  - ```interfaces``` -  регулярное выражение для выбора сетевых интерфейсов. ```^ens[0-9]+``` соответствует ```ens0```, ```ens1``` и т.д.  
  - ```loadBalancerIPs``` - анонсировать IP адресов типа ```LoadBalancer```(Назначаются автоматически из ```CiliumLoadBalancerIPPool```).  
  - ```externalIP``` - анонсировать External IP сервисов. External IP - это IP-адрес, который можно вручную назначить **любому** сервису (включая ```ClusterIP```) через поле ```spec.externalIPs```. В отличии от ```LoadBalancer```, IP не выделеляется из пула автоматически - его укзаывает администратор в манифесте сервиса: 
```yml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: my-app
  name: my-svc
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
  exernalIPs:
  - 192.168.122.15
  selector:
    app.kubernetes.io/name: my-app
  type: ClusterIP
```

По умолчанию Kubernetes принимает такой IP, но не анонсирует его в сеть - нужно самостоятельно настраивать маршрутизацию. При ```externalIPs: true``` Cilium берет это на себя, отвечая на ARP-запросы за указынный IP. 

| | **Load Balancer** | **externalIPs** |
| :--- | :--- | :--- |
| Назначение IP | Автоматически из пула | Вручную в манифесте | 
| Тип сервиса| только ```Load Balancer``` | Любой (```ClusterIP```, ```NodePort```)|
| Применение | Основной способ| когда нужен конкретный IP без пула|

Применим нашу политику:  

```sh
kubectl apply -f manifests/02-lb-l2-policy.yml
```
```sh
ciliuml2announcementpolicy.cilium.io/default-l2-policy created
```

## Тестирование

Создадим тестовый под и сервис типа LoadBalancer.

Файл ```manifests/03-lb-test.yml```:

```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: lb-test
  labels: &labels
    app.kubernetes.io/name: lb-test
    app.kubernetes.io/version: "1.27.5"
spec:
  containers:
  - name: nginx
    image: nginx:1.27.5
    resources:
      limits:
        memory: "128Mi"
        cpu: "200m"
      requests:
        memory: "64Mi"
        cpu: "100m"
    ports:
      - containerPort: 80
        name: http
        protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: lb-test-svc
  labels: &labels
    app.kubernetes.io/name: lb-test
    app.kubernetes.io/version: "1.27.5"
spec:
  type: LoadBalancer
  selector: *labels
  ports:
  - port: 80
    targetPort: http
    protocol: TCP

```

```sh
kubectl apply -f manifests/03-lb-test.yml
```
```sh
pod/lb-test created
service/lb-test-svc created
```

Проверим, что сервис получил внешний IP:

```sh
 kubectl get svc lb-test-svc
```
```sh
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
lb-test-svc   LoadBalancer   10.100.203.226   192.168.122.10   80:30496/TCP   108s
```

IP ```192.168.122.10``` выделен из пула. Посмотрим leader election:

```sh
kubectl get leases -n kube-system | grep l2
```
```sh
cilium-l2announce-default-lb-test-svc   worker2                                                                     3m26s
```

Нода ```worker2``` стала лидером для этого сервиса. Она по умолчанию будет отвечать на ARP-запросы.

Проверим ARP с другой ноды
```sh
arping -c 2 -I ens3  192.168.122.10
```
```sh
ARPING 192.168.122.10
58 bytes from 52:54:00:82:38:90 (192.168.122.10): index=0 time=196.935 usec
58 bytes from 52:54:00:82:38:90 (192.168.122.10): index=1 time=728.899 usec

```

IP отвечает MAC-адресом ноды-лидера. 

Проверим HTTP-доступ:

```sh
curl -s -o /dev/null -w '%{http_code}' http://192.168.122.10 && echo ""
```
```sh
200
```

Сервис доступен из локальной сети по внешнему IP. 

## Отказоустойчивость

При использовании L2 Announcements весь трафик для конкретного IP проходит через одну ноду-лидер. Что произойдёт, если эта нода станет недостуна?

Cilium использует механизм **Kubernetes Leases** для leader election. Каждый сервис типа LoadBalancer получает отдельный Lease в namespace ```kube-system```:

```sh
kubectl -n kube-system cilium-l2announce-default-lb-test-svc get lease -o yaml
```
```sh
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2026-05-15T13:40:15Z"
  name: cilium-l2announce-default-lb-test-svc
  namespace: kube-system
  resourceVersion: "60294"
  uid: 52d9517d-dc5e-44f9-a2e0-8fd29083867f
spec:
  acquireTime: "2026-05-15T13:40:15.113620Z"
  holderIdentity: worker2
  leaseDurationSeconds: 15
  leaseTransitions: 0
  renewTime: "2026-05-15T13:54:30.168681Z"
```

Ключевые параметры:  

  - **leaseDurationSeconds: 15** - если лидер не обновляет lease в течении 15 секунд, он считается потерявшим лидерство.  
  - **holderIdentity** - текущая нода-лидер.  
  - **leaseTransitions** - счётчик переключения лидера.  

Сценарий отказа: 

Допустим, нода ```worker2``` - текущий лфидер для IP ```192.168.122.10```. При остановке cilium агента на worker2:

  1. Агент на worker2 перестаёт обновлять lease.  
  2. Через ```leaseDurationSeconds```(15 секунд) lease истекает.  
  3. Агент Cilium  на другой ноде захватывает lease.  
  4. Новый лидер отправляет gratuitous ARP, объявляя свой MAC-адрес для IP.
  5. Клиенты обноляют ARP-кеш и трафик переключается на новую ноду.

На практике переключение происходит быстрее, чем 15 секунд - другие агенты обнаруживают истечение lease и захватывают его при следующей проверке:

```sh
kubectl -n kube-system get lease cilium-l2announce-default-lb-test-svc -o jsonpath='{.spec.holderIdentity}'
```
```sh
worker2
```
```sh
kubectl -n kube-system get lease cilium-l2announce-default-lb-test-svc -o jsonpath='{.spec.leaseTransitions}'
```
```sh
0
```

#### Время переключения

Общее время недоступности складывается из:  

| **Этап** | **Время** |
| :--- | :--- | 
| Обнаружение отказа(истечение lease) | до 15 секунд |
| Захват lease новым лидером | < 1 секунды |
| Gratuitous ARP + обновление кешей | < 1 секунды |
| **Итого** | **~15-17 секунд** |

Это сопоставимо с поведением MetalLB в режиме L2. Для сценариев, требующих более быстрого переключения и балансировки трафика между нодами следует испольовать BGP с ECMP

### Очистка
```sh
kubectl delete -f 06-lb-test.yaml
```

## BGP Control Plane

Для production-окружений с сетевым оборудованием, поддерживающим BGP, Cilium предоставляет
**BGP Control Plane** — возможность анонсировать IP-адреса сервисов и подсети подов
через BGP-пиринг с маршрутизаторами.

### Преимущества BGP перед L2

| Характеристика | L2 Announcements | BGP |
|---|---|---|
| Область действия | Один L2-сегмент | Между сетями (L3) |
| Отказоустойчивость | Leader election (одна нода) | ECMP (несколько нод одновременно) |
| Балансировка | Весь трафик через одну ноду | Распределение по нескольким нодам |
| Требования | Нет | BGP-совместимый маршрутизатор |

### Архитектура BGP в Cilium

Cilium использует встроенный BGP-спикер на базе GoBGP. Каждый агент Cilium может
установить BGP-сессию с маршрутизатором и анонсировать:

- IP-адреса сервисов типа LoadBalancer.
- Подсети подов (Pod CIDR).
- External IP сервисов.

Конфигурация BGP выполняется через CRD-ресурсы:

- **CiliumBGPClusterConfig** — настройки BGP на уровне кластера.
- **CiliumBGPPeerConfig** — параметры пиринга с маршрутизатором.
- **CiliumBGPAdvertisement** — какие маршруты анонсировать.

Включение BGP через Helm:

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --namespace kube-system \
  --reuse-values \
  --set bgpControlPlane.enabled=true
```

> **Примечание:** Демонстрация BGP требует наличия маршрутизатора с поддержкой BGP
> (например, FRRouting, Mikrotik, Cisco), что выходит за рамки тестового стенда.
> Подробная документация доступна на [docs.cilium.io](https://docs.cilium.io/en/v1.19/network/bgp-control-plane/).