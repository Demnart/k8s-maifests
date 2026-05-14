# Hubble

## Что такое Hubble

[Hubble]() - инструмент наблюдаемости сети, встроенный в Cilium. Hubble собирает данные о сетевых потоках(flows) непосредственно из eBPF-хуков в ядре - без дополнительного overhead.  

Основные возможности:  

  - Наблюдение за сетевым трафиком в реальном времени(L3/L4/L7).  
  - Фильтрация потоков по namespace, pod, verdict(FORWARDED/DROPPED), протоколу.  
  - Визулизация зависимостей между сервисами (Service Map).  
  - Диагностика проблем с сетевыми политикамид.  

## Компоненты Hubble

```txt
┌──────────────────────────────────────────────────────────┐
│                    Hubble UI (веб)                       │
│                  Визуализация потоков                    │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────┴───────────────────────────────┐
│                   Hubble Relay                           │
│           Агрегация данных со всех нод                   │
└────┬──────────────┬──────────────┬───────────────────────┘
     │              │              │
┌────┴────┐    ┌────┴────┐    ┌────┴────┐
│ Hubble  │    │ Hubble  │    │ Hubble  │ Встроен в Cilium
│ Server  │    │ Server  │    │ Server  │ Agent на каждой ноде
│  (r1)   │    │  (r2)   │    │  (r3)   │
└─────────┘    └─────────┘    └─────────┘
```

  - **Hubble Server** - работает внутри Cilium Agent на каждой ноде. Собирает flow-данные из eBPF event streams.  
  - **Hubble Relay** - Deployment, агрегирует данные со всех нод через gRPC.  
  - **Hubble UI** - веб-интерфейс с графом зависимостей сервисов.  
  - **Hubble CLI** - утилита командной строки для запроса потоков.  

## Установка Hubble

Для установки Hubble воспользовался следуюещей командой:  

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium --namespace kube-system --reuse-values --set hubble.relay.enabled=true --set hubble.ui.enabled=true
```

Upgrade - тк мы уже устанавливали в кластер Cilium и по факту расширяем нашу установку добавляя в неё Hubble.  

После установки Hubble проверим статус Cilium:  

```sh
cilium status
```
```sh
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium-envoy             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Deployment             hubble-relay             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui                Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 3
                       cilium-envoy             Running: 3
                       cilium-operator          Running: 2
                       clustermesh-apiserver    
                       hubble-relay             Running: 1
                       hubble-ui                Running: 1
Cluster Pods:          4/4 managed by Cilium
Helm chart version:    1.19.4
Image versions         cilium             quay.io/cilium/cilium:v1.19.3@sha256:2e61680593cddca8b6c055f6d4c849d87a26a1c91c7e3b8b56c7fb76ab7b7b10: 3
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.36.6-1776000132-2437d2edeaf4d9b56ef279bd0d71127440c067aa@sha256:ba0ab8adac082d50d525fd2c5ba096c8facea3a471561b7c61c7a5b9c2e0de0d: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.19.3@sha256:205b09b0ed6accbf9fe688d312a9f0fcfc6a316fc081c23fbffb472af5dd62cd: 2
                       hubble-relay       quay.io/cilium/hubble-relay:v1.19.3@sha256:5ee21d57b6ef2aa6db67e603a735fdceb162454b352b7335b651456e308f681b: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.3@sha256:db1454e45dc39ca41fbf7cad31eec95d99e5b9949c39daaad0fa81ef29d56953: 1
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.3@sha256:661d5de7050182d495c6497ff0b007a7a1e379648e60830dd68c4d78ae21761d: 1
```

## Hubble CLI

### Установка

```sh
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/main/stable.txt)
curl -L --fail --remote-name-all \
  https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
```

Проверяем версию:  

```sh
hubble version
```
```sh
hubble v1.19.3@HEAD-400168b compiled with go1.26.2 on linux/amd64
```

## Подключение к Hubble Relay

Hubble CLI подключается к Hubble Relay через port-forward. Cilium CLI предоставляет удобную команду:  

```sh
cilium hubble port-forward &
```

Проверяем подключение:  

```sh
hubble status
```
```sh
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 12,285/12,285 (100.00%)
Flows/s: 11.92
Connected Nodes: 3/3
```

Hubble подключен ко всем 3 нодам и собирает потоки.  

## Наблюдение за трафиком

Создадим тестовые ресурсы для демонстрации:  
```sh
kubectl run nginx-test --image=nginx --restart=Never
kubectl wait --for=condition=Ready pod/nginx-test --timeout=60s
kubectl expose pod nginx-test --port=80 --name=nginx-test-svc 
kubectl run dnstools --image=infoblox/dnstools --restart=Never --command -- sleep 3600  
kubectl wait --for=condition=Ready pod/dnstools --timeout=60s
```

Генерируем HTTP-трафик:

```sh
kubectl exec dnstools -- wget -qO- http://nginx-test-svc > /dev/null
```

Наблюдаем потоки в namespace ```default```:  

```sh
hubble observe --namespace default --last 10
```
```sh
May 14 13:51:31.303: default/dnstools:54628 (ID:64874) -> 169.254.25.10:53 (host) to-stack FORWARDED (UDP)
May 14 13:51:31.305: default/dnstools:54628 (ID:64874) <- 169.254.25.10:53 (host) to-endpoint FORWARDED (UDP)
May 14 13:51:31.305: default/dnstools:36226 (ID:64874) -> default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: SYN)
May 14 13:51:31.305: default/dnstools:36226 (ID:64874) <- default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: SYN, ACK)
May 14 13:51:31.305: default/dnstools:36226 (ID:64874) -> default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: ACK)
May 14 13:51:31.305: default/dnstools:36226 (ID:64874) -> default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 14 13:51:31.306: default/dnstools:36226 (ID:64874) <- default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
May 14 13:51:31.306: default/dnstools:36226 (ID:64874) -> default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 14 13:51:31.306: default/dnstools:36226 (ID:64874) <- default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: ACK, FIN)
May 14 13:51:31.306: default/dnstools:36226 (ID:64874) -> default/nginx-test:80 (ID:5665) to-endpoint FORWARDED (TCP Flags: ACK)
```

Из вывода видно полный цикл HTTP-запроса:  
  
  1. DNS-запрос от ```dnstools``` к NodeLocalDNS (```169.254.25.10:53```).  
  2. TCP-соединение от ```dnstools```(ID:64874) к ```nginx-test```(ID:5665) на порт 80.  
  3. Все пакеты в статусе ```FORWARDED``` - доставлены успешно.  


## Полезные фильтры

```sh
hubble observe --pod defatult/nginx-test
```

Только отброшенные пакеты:  

```sh
hubble observe --verdict DROPPED
```

Фильтр по протоколу:  

```sh
hubble observe --protocol TCP --namespace default
```

Наблюдение в реальном времени(follow):  

```sh
hubble observe --namespace default --folow
```

## Просмотр заблокированного траффика