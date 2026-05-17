# Network Policies

Cilim поддерживает стандартные Kubernetes NetworkPolicy и расширенные CiliumNetworkPolicy. В этом разделе рассмотрим практическое применение сетевых политик и их отладку с помощью Hubble. 

## Подготовка тестовых приложений

Для демонстрации сетевых политик создадим три пода:

```sh
# Приложение - веб-сервер
kubectl run app1 --image=nginx:1.27.5 --restart=Never --labels="app=app1"

# Разрешённый клиент 
kubectl run dnstools --image=infoblox/dnstools:latest --restart=Never \
  --labels="app=dnstools"  --command -- sleep 3600

# Добавляем сервис
kubectl expose pod app1 --port=80 --target-port=80

# Ожидание готовности

kubectl wait --for=condition=Ready pod/app1 pod/dnstools --timeout=60s
```

Убедимся, что трафик проходит без ограничений:  

```sh
kubectl exec dnstools -- wget -qO- --timeout=3 http://app1
```
```sh
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

## Стандартные NetworkPolicy

#### Запрет всего входящего трафика

Базовый паттерн - запретить весь входящий трафик к поду, затем выборочно разрешить только нужный. 

Файл `manifests/np-deny-all.yml`:

```yml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-to-app1
spec:
  podSelector:
    matchLabels:
      app: app1
  policyTypes:
  - Ingress
```

Пустой список `ingress`(отсутсвие поля) означает: запретить весь входящий трафик.

```sh
kubectl apply -f manifests/np-deny-all.yml 
```

Проверим:  

```sh
kubectl apply -f manifests/np-deny-all.yml 
```
```sh
wget: download timed out
command terminated with exit code 1
```

#### Разрешение доступа от определенных подов

Добавим политики, разрешающую доступ от подов с label: `app: dnstools`:

Файл `manifests/np-allow-dnstools.yml`:

```yml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dnstools-to-app1
spec:
  podSelector:
    matchLabels:
      app: app1
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: dnstools
    ports:
    - protocol: TCP
      port: 80
```

Применяем: 

```sh
kubectl apply -f manifests/np-allow-dnstools.yml
```
```sh
kubectl exec dnstools -- wget -qO- --timeout=3 http://app1
```
```sh
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

Доступ восстановлен только для подов с label: `app: dnstools`

Удалим стандартные политики перед переходом к CiliumNetworkPolicy:

```sh
kubectl delete networkpolicy allow-dnstools-to-app1 deny-all-to-app1
```

## CiliumNetworkPolicy

CiliumNetworkPolicy расширяет возможности стандартных политик, добавляя:

  - **L7-фильтрацию** - HTTP-метод,путь, заголовки.  
  - **DNS-политики** - разрешение доступа к опредёленным доменным именам.  
  - **CIDR-политики** - фильтрация по IP-подсетям.  
  - **Entity-политики** -доступ к предопределённым сущностям(host, world, cluster).  


### L7-политики(HTTP)

L7-политики позволяют фильтровать трафик на уровне HTTP- по методу, пути, заголовкам. При приминении L7-политик трафик проходит через Cilium Envoy для инспекции. 

Пример: разрешить только `GET /` к подам с label `app: app1` от `app: dnstools`.

Файл `manifests/cnp-17-policy.yml`:

```yml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: 17-policy-app1
spec:
  endpointSelector:
    matchLabels:
      app: app1
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: dnstools
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/"
```
```sh
kubectl apply -f manifests/cnp-17-policy.yml
```

Проверим различные запросы:

```sh
# GET / - Разрешено
kubectl exec dnstools -- wget -qO- --timeout=3 http://app1
```
```sh
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
```sh
# GET /secret - запрещено (путь не совпадает)
kubectl exec dnstools -- wget -qO- --timeout=3 http://app1/secret
```
```sh
wget: server returned error: HTTP/1.1 403 Forbidden
command terminated with exit code 1
```
```sh
# POST / - запрещёно (метод не совпадает)
kubectl exec dnstools -- wget -qO- --timeout=3 --post-data='' http://app1
```
```sh
wget: server returned error: HTTP/1.1 403 Forbidden
command terminated with exit code 1
```

Cilium Envoy вовращает `403 Forbidden` для запросов, не соответствующих политике.

Удалим политику:

```sh
kubectl delete ciliumnetworkpolicy 17-policy-app1
```

### DNS-политики

DNS-политиуки позволяют разрешить исходящий трафик только к опредёленным доменным именам. Cilium перехватывает DNS-ответы и динамически обновляет правила на основе IP-адресов, полученных из DNS.

Пример: разрешить поду `dnstools`, доступ к `ya.ru`, но заблокировать доступ к другим внешним ресурсам.

> **Примечание** - В нашем кластере DNS-запросы обрабатывает NodeLocalDN(169.254.25.10), работающий на хосте. Поэтому для разрешения DNS необходимо добавить правило для entity `host`.

Файл `manifests/cnp-dns-policy.yml`
```yml
---
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: dnstools
  egress:
    - toEntities:
        - host
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    - toFQDNs:
        - matchName: ya.ru
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
            - port: "80"
              protocol: TCP

```
```sh
kubectl apply -f manifests/cnp-dns-policy.yml
```

Ключевые моменты:

  - Блок с `rules.dns` **обязателен** - Cilium должен видеть DNS-ответы, чтобы узнать IP-адреса для FQDN-правил.  
  - `toEntities: [host]` - разрешает доступ к NodeLocalDNS(169.254.25.10).  
  - `toFQDN` - разрешает трафик к IP-адресам, полученным из DNS для указанного домена.  
  - `toEndpoints: [{}]` - пустой селектор означает "любой под в кластере".  

Проверим:

```sh
# ya.ru - разрешено
kubectl exec dnstools -- wget -qO- --timeout=5 http://ya.ru 2>&1 | head -1
```
```sh
<!doctype html><html prefix="og: http://ogp.me/ns#" lang="ru"><meta http-equiv="X-UA-Compatible" content="IE=edge"><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
```
```sh
# google.com  - запрещёно. 
kubectl exec dnstools -- wget -qO- --timeout=5 http://google.com 2>&1 | head -1
```

DNS-запрос к google.com проходит (Cilium видит ответ), но TCP-соединение
блокируется, потому что google.com не указан в `toFQDNS`.

Удалим политику

```sh
kubectl delete ciliumnetworkpolicies dns-policy
```

## Отладка политик через Hubble

Hubble - ключевой инструмент для отладки сетевых политик. Он показывает каждый сетевой поток с вердиктом (FORWARDED/DROPPED) и причиной блокировки. 

### Просмотр заблокированного трафки

Для демонстрации применим deny-all политику и попробуем обратиться к app1:

```sh
kubectl apply -f manifests/np-deny-all.yml 
```

Попытка доступа (в фоне, с таймаутом):

```sh
kubectl exec dnstools -- wget -qO- --timeout=3 http://app1 &
```

Просмотр заблокированного трафика в Hubble:

```sh
cilium hubble port-forward &
hubble observe --namespace default --pod app1 --verdict DROPPED --last 5
```
```sh
May 17 13:11:16.570: default/dnstools:49724 (ID:47516) <> default/app1:80 (ID:3741) Policy denied DROPPED (TCP Flags: SYN)
May 17 13:11:17.599: default/dnstools:49724 (ID:47516) <> default/app1:80 (ID:3741) policy-verdict:none TRAFFIC_DIRECTION_UNKNOWN DENIED (TCP Flags: SYN)
May 17 13:11:17.599: default/dnstools:49724 (ID:47516) <> default/app1:80 (ID:3741) Policy denied DROPPED (TCP Flags: SYN)
May 17 13:11:18.623: default/dnstools:49724 (ID:47516) <> default/app1:80 (ID:3741) policy-verdict:none TRAFFIC_DIRECTION_UNKNOWN DENIED (TCP Flags: SYN)
May 17 13:11:18.623: default/dnstools:49724 (ID:47516) <> default/app1:80 (ID:3741) Policy denied DROPPED (TCP Flags: SYN)

```

Проверки HUBBLE:

```sh
# Только заблокированный трафик
hubble observe --verdict DROPPED

# Трафик конкретного пода
hubble observe --pod default/app1

# Трафик определённого namespace
hubble observe --namespace default

# Только HTTP-потоки
hubble observe --protocol http

# Комбинация фильтров
hubble observe --namespace default --pod app1 --verdict DROPPED --protocol http

# Последние N записей
hubble observe --last 10

# Наблюдение в реальном времени (follow)
hubble observe --namespace default --follow
```

## Очистка

Удалим созданные ресурсы:

```sh
kubectl delete pod/app1 pod/dnstools
kubectl delete svc/app1
kubectl delete networkpolicies deny-all-to-app1
```