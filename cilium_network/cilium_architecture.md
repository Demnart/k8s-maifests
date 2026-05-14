# Архитектура Cilium

## Компоненты Cilium

Cilium состоит из нескольких компонентов, каждый из которых отвечает за свою часть
функциональности:

```txt
┌────────────────────────────────────────────────────┐
│                  Kubernetes API                    │
└─────────────────────┬──────────────────────────────┘
                      │
           ┌──────────┴──────────┐
           │  Cilium Operator    │    Управление IPAM,
           │  (Deployment, 2шт)  │    координация агентов
           └─────────────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
┌───┴───┐         ┌───┴───┐        ┌───┴───┐
│Cilium │         │Cilium │        │Cilium │    Агент на каждой ноде,
│Agent  │         │Agent  │        │Agent  │    загрузка eBPF программ
│(r1)   │         │(r2)   │        │(r3)   │
└───┬───┘         └───┬───┘        └───┬───┘
    │                 │                │
┌───┴───┐         ┌───┴───┐        ┌───┴───┐
│Cilium │         │Cilium │        │Cilium │    L7 прокси
│Envoy  │         │Envoy  │        │Envoy  │    (HTTP, gRPC)
└───────┘         └───────┘        └───────┘
```

## Cilium Agent

DaemonSet, запускается на каждой ноде кластера. Основные функции:  

  - Взаимодействует с Kubernetes API - отслеждивает создание/удаление подов, сервисов, политик.  
  - Генерирует и загружает eBPF-программы в ядро.  
  - Управляет сетевыми интерфейсами подов.  
  - Обрабатывает IPAM(назначение IP-адресов подам).  
  - Обеспечивает балансировку сервисов(замена kubeproxy).  

## Cilium Operator

Deployment(по умолчанию 2 реплики). Отвечает за задачи уровня кластера:  

  - Управление пулом IP-адресов (IPAM в режиме cluster-pool).  
  - Сборка мусора: удаление устаревших CilimIdentity & CiliumEndpoint.  
  - Синхронизация ресурсов между нодами.  

## Cilium envoy

DaemonSet, запускается на каждой ноде. Используется для:  

  - Обработка L7-трафика (HTTP, gRPC) при использовании L7 Network Policies.  
  - Проксирование трафика, который не может быть обработан только средствами eBPF.  


## CDR ресусры

Cilium создаёт набор Custom Resource Definition:  
```sh
kubectl api-resources | grep cilium
```
```sh
ciliumcidrgroups                    ccg                                 cilium.io/v2                      false        CiliumCIDRGroup
ciliumclusterwidenetworkpolicies    ccnp                                cilium.io/v2                      false        CiliumClusterwideNetworkPolicy
ciliumendpoints                     cep,ciliumep                        cilium.io/v2                      true         CiliumEndpoint
ciliumidentities                    ciliumid                            cilium.io/v2                      false        CiliumIdentity
ciliuml2announcementpolicies        l2announcement                      cilium.io/v2alpha1                false        CiliumL2AnnouncementPolicy
ciliumloadbalancerippools           ippools,ippool,lbippool,lbippools   cilium.io/v2                      false        CiliumLoadBalancerIPPool
ciliumnetworkpolicies               cnp,ciliumnp                        cilium.io/v2                      true         CiliumNetworkPolicy
ciliumnodeconfigs                                                       cilium.io/v2                      true         CiliumNodeConfig
ciliumnodes                         cn,ciliumn                          cilium.io/v2                      false        CiliumNode
ciliumpodippools                    cpip                                cilium.io/v2alpha1                false        CiliumPodIPPool
```  

Основные:  

  - **CiliumEndpoint** - предоставляет под с точки зрения Cilium(IP, Identity, состояния).  
  - **CiliumIdentity** - числовой идентификатор, вычесленный из labels пода. Используется вместо IP-адресов в сетевых политиках.  
  - **CiliumNode** - информация о ноде: IPAM, маршрутизация.  
  - **CiliumNetworkPolicy** - расширенные сетевые политики(L7, DNS, CIDR).  

## Cilium CLI

Cilium CLI(cilium) - утилита для управления и диагностики Cilium из командной строки:  

### Установка

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

Проверим версию:  

```sh
cilium version
```
```sh
cilium-cli: v0.19.2 compiled with go1.25.5 on linux/amd64
cilium image (default): v1.19.1
cilium image (stable): v1.19.3
cilium image (running): 1.19.3
```

Проверим статус:  

```sh
cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium-envoy             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium                   Running: 3
                       cilium-envoy             Running: 3
                       cilium-operator          Running: 2
                       clustermesh-apiserver    
                       hubble-relay             
Cluster Pods:          2/2 managed by Cilium
Helm chart version:    1.19.3
Image versions         cilium             quay.io/cilium/cilium:v1.19.3@sha256:2e61680593cddca8b6c055f6d4c849d87a26a1c91c7e3b8b56c7fb76ab7b7b10: 3
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.36.6-1776000132-2437d2edeaf4d9b56ef279bd0d71127440c067aa@sha256:ba0ab8adac082d50d525fd2c5ba096c8facea3a471561b7c61c7a5b9c2e0de0d: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.19.3@sha256:205b09b0ed6accbf9fe688d312a9f0fcfc6a316fc081c23fbffb472af5dd62cd: 2
```

Все компоненты в статусе OK. Hubble Relay пока отключен - установим его в следующем разделе.  

## Детальная диагностика

Для более подробной информации можно обратиться к агенту Cilium внутри пода:  

```sh
kubectl exec -n kube-system ds/cilium -- cilium-dbg status  --verbose
```
```sh
KVStore:                 Disabled   
Kubernetes:              Ok         1.35 (v1.35.3) [linux/amd64]
Kubernetes APIs:         ["cilium/v2::CiliumCIDRGroup", "cilium/v2::CiliumClusterwideNetworkPolicy", "cilium/v2::CiliumEndpoint", "cilium/v2::CiliumNetworkPolicy", "cilium/v2::CiliumNode", "core/v1::Pods", "networking.k8s.io/v1::NetworkPolicy"]
KubeProxyReplacement:    True   [ens3   192.168.0.34 fe80::d050:ceb5:118c:1ae1 (Direct Routing)]
Host firewall:           Disabled
SRv6:                    Disabled
CNI Chaining:            none
CNI Config file:         successfully wrote CNI configuration file to /host/etc/cni/net.d/05-cilium.conflist
Cilium:                  Ok   1.19.3 (v1.19.3-f5eb641b)
NodeMonitor:             Listening for events on 6 CPUs with 64x4096 of shared memory
Cilium health daemon:    Ok   
IPAM:                    IPv4: 4/254 allocated from 10.0.0.0/24, 
IPv4 BIG TCP:            Disabled
IPv6 BIG TCP:            Disabled
BandwidthManager:        Disabled
Routing:                 Network: Tunnel [vxlan]   Host: Legacy
Attach Mode:             TCX
Device Mode:             veth
Masquerading:            IPTables [IPv4: Enabled, IPv6: Disabled]
Controller Status:       23/23 healthy
Proxy Status:            OK, ip 10.0.0.41, 0 redirects active on ports 10000-20000, Envoy: external
Global Identity Range:   min 256, max 65535
Hubble:                  Ok              Current/Max Flows: 4095/4095 (100.00%), Flows/s: 4.60   Metrics: Disabled
Encryption:              Disabled        
Cluster health:          3/3 reachable   (2026-05-14T11:50:47Z)   (Probe interval: 1m56.754608943s)
Name                     IP              Node                     Endpoints
Modules Health:          Stopped(23) Degraded(0) OK(80)
```

Из вывода видно:  

  - **KubeProxyReplacement: True** - Cilium полностью заменяет Kube-proxy.  
  - **Routing:Network: Tunnel [vxlan]** - Используется VXLAN overlay для связаности между нодами.  
  - **IPAM IPv4: 4/254 allocated from 10.0.0.0/24** - на данной ноде выделено 4 ip из подсети /24.  


## Информация о нодах

```sh
kubectl get ciliumnodes -o wide
```
```sh
NAME      CILIUMINTERNALIP   INTERNALIP      AGE
control   10.0.0.41          192.168.0.34    36m
worker1   10.0.2.99          192.168.0.175   16m
worker2   10.0.1.28          192.168.0.194   16m
```

Каждой ноде выделена своя подсеть /24 из общего пула ```10.0.0.0/16```:  
  
  - control -> ```10.0.0.0/24```  
  - worker1 -> ```10.0.2.0/24```
  - worker2 -> ```10.0.1.0/24```      

## Endpoints и Identity

```sh
kubectl get ciliumendpoints -A
```
```sh
NAMESPACE     NAME                       SECURITY IDENTITY   ENDPOINT STATE   IPV4         IPV6
kube-system   coredns-5f84cc8d6f-97cxd   38651               ready            10.0.1.19    
kube-system   coredns-5f84cc8d6f-vzsp5   38651               ready            10.0.2.225   
```

Оба пода CoreDNS имеют одинаковый **SecurityIdentity 38651** - он вычислен из их labels. При пересоздании пода(даже с другим IP ), идентификтор сохраняется, пока labels не изменяется.  

```sh
kubectl get ciliumidentities
```
```sh
NAME    NAMESPACE     AGE
38651   kube-system   42m 
```

## Network Policies

Cilium поддерживает стандартные Kubernetes NetworkPolicy и расширенные CiliumNetworkPolicy.  

### Стандартные NetworkPolicy (L3/L4)

Стандартные NetworkPolicy работают на уровне L3(IP)/L4(Порты). Cilium реализует их через eBPF - без Iptables.  

Пример: разрешить входящий трафик к подам с label ```app: backend``` только от подов с label ```app: frontend```  на порт 8080:  

```yml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

## CiliumNetworkPolicy(L7)

CiliumNetworkPolicy расширяет возможности стандартных политик, добавляя фильтрацию на уровне L7 - HTTP, gRPC, Kafka, DNS.  

Пример: разрешить только GET-запросы к пути ```/api/public``` для подов с label ```app: api```:  

```yml
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-get-public-api
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/api/public"    
```

При применении L7-политик трафик перенаправляется через Cilium Envoy для инспеции.  

## CiliumClusterwideNetworkPolicy

Политики уровня кластера - применяются ко всем namespace:  

```yml
---
kind: CliumClusterwideNetworkPolicy
apiVersion: cilium.io/v2
metadata:
  name: deny-external-access
spec:
  endpointSelector:
    matchLabels:
      envionment: production
  inressDeny:
  - fromCIDR:
    - "0.0.0.0/0"
```

## Как Cilium применяет политики

В отличии от традиционных CNI, Cilium не оперирирует IP-адресами при применении политик. Вместо этого:  

  1. Из labels пода вычисляется числовой **SecurityIdentity**.  
  2. Identity хранится в BPF map на каждой ноде.  
  3. При поступлении пакета eBPF-программа извлекает identity отправителя и проверяет его по policy map.  
  4. Решения (allow/deny) принимается за O(1) - один lookup в хеш-таблице.  

Это означает, что при пересоздании пода с новым IP - политики продолжают работать , потому , что Identity определяется labels, а не подом.   