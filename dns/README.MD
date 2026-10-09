# Система DNS в кластере Kubernetes.

Основная система DNS - проект coreDNS - https://github.com/coredns/coredns отвечающий за систему 

При базовой установке с помощью инструмента Kubeadm в системе создается один `Deployment` `coredns` - который будет отвечать за преобразование имен.

В рамках моей настройки, при установке кластера был добавлен `Daemonset nodelocaldns` - https://github.com/kubernetes-sigs/node-local-dns - это кеширующий dns сервис и так как это `Daemonset` поды этого сервиса устанавливаются на каждой ноде кластера.

При запросе с ноды, обращение сначала происходит к локальному резолверу, если на нем нет данных, обращение происходит к `coredns` и если в `coredns` нет данных обращение происходит к резолв файлу контрольной ноды и тд.

## Правила именования сервисов

Представим, что мы хотим получить информацию о сервисе `coredns` из `namespace: work`. 
Для этого воспользуется инструментом для отладки сети `infoblox/dnstools`: 

```sh
kubectl run -it --rm --restart=Never --image=infoblox/dnstools dnstools
```

В самом контейнере попробуем получить host `coredns`:
```sh
dnstools# host coredns
Host coredns not found: 2(SERVFAIL)
```

Как видно из примера мы не получили информации о сервисе coredns. 
Для получения инфорнмации мы должны указать полностью квалифицировнного доменного имени сервиса. Это будет иметь следующий вид:

```sh
host kube-dns.kube-system.svc.cluster.local
```
```sh
kube-dns.kube-system.svc.cluster.local has address 10.96.0.10
```

Что означает выше указанныя запись:
- **kube-dns.** - имя сервиса coredns.
- **kube-system.** - `namespace` в котором создан сервис
- **svc.** - указываем, что речь идет о сервисе(На самом деле не считая `Statefulset` dns-имена присваиваются только сервисам).
- **cluster.local.** - имя кластера. 

Информация о том как именно должен происходит резолвинг находится в файле `/etc/resolv.conf`. 

Непосредственно в поде в моем кластере он имеет следующий вид: 

```sh
cat /etc/resolv.conf
```
```sh
search work.svc.cluster.local svc.cluster.local cluster.local
nameserver 169.254.25.10
options ndots:5
```

Кстати, если присмотреться к файлу resolv.conf можно понять, что можно сократить ранее сделанный запрос до: 

```sh
dnstools# host kube-dns.kube-system
```
```sh
kube-dns.kube-system.svc.cluster.local has address 10.96.0.10
```

Так как `.svc.cluster.local` будет подставлен из **resolv.conf** пода.

## Hostname пода

По умолчанию `hostname` пода соответствует имени, которое мы указали для пода. 
В качестве примера создан манифест пода `0-dns-tools.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnstools
  labels:
    app.kubernetes.io/name: dnstools
spec:
  containers:
  - name: dnstools
    image: infoblox/dnstools:latest
    command:
      - sleep
      - "36000"
    imagePullPolicy: IfNotPresent
    restartPolicy: Always
    resources:
      limits:
        memory: "64Mi"
        cpu: "100m"
      requests:
        memory: "32Mi"
        cpu: "50m"
```

При запуске этого контейнера и входе в него: 
```sh
kubectl apply -f 0-dns-tools.yaml
```
```sh
pod/dnstools created
```
```sh
kubectl exec -it dnstools -- sh
```

Мы можем получить его `hostname`:
```sh
dnstools# hostname
```
```sh
dnstools
```

Либо полное доменное имя FQDN с помощью команды:
```sh
hostname -f
```
```sh
dnstools# hostname -f
```
```sh
dnstools
```

В нашем случае, что короткое имя, что FQDN совпадают и равняются - `dnstools`

Если мы хотим дать системе внутри пода другое имя(FQDN) то необходимо внести незначительные правки в нашем манифесте, я создал новый манифест `1-dns-tools.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnstools
  labels:
    app.kubernetes.io/name: dnstools
spec:
  hostname: dns
  subdomain: tools
  containers:
  - name: dnstools
    image: infoblox/dnstools:latest
    command:
      - sleep
      - "36000"
    imagePullPolicy: IfNotPresent
    restartPolicy: Always
    resources:
      limits:
        memory: "64Mi"
        cpu: "100m"
      requests:
        memory: "32Mi"
        cpu: "50m"
```

Как видно в разделе spec нашего пода появилось два новых параметра `hostname` и `subdomain`. Применим этот манифест и проверим, что получилось: 
```sh
kubectl apply -f 1-dns-tools.yaml
```
```sh
pod/dnstools created
```

Проваливаемся в контейнер: 

```sh
kubectl exec -it dnstools -- sh
```

А теперь проведем провери: 

```sh
dnstools# hostname
```
```sh
dns
```

А теперь полное доменное имя:
```sh
hostname -f
```
```sh
dns.tools.work.svc.cluster.local
```

`Hostname` контейнера помещается в непосредственно в файле /etc/hosts пода:

```sh
cat /etc/hosts
```
```sh
# Kubernetes-managed hosts file.
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
fe00::0 ip6-mcastprefix
fe00::1 ip6-allnodes
fe00::2 ip6-allrouters
10.0.1.101      dns.tools.work.svc.cluster.local        dns
```

Можно указвать имя, но по факту это не имеет смысла.

## Траблшутинг

### Преобразование имет

При возникновении ошибок в кластере, когда не понятно, а что происходит с кластером, важно проверить работает ли вообще система DNS в кластере. 
По умолчанию в `namespace: default` имеется сервис `kubernetes`:

```sh
kubectl -n default get svc
```
```sh
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   10d
```

Проверить его можно как и ранее с помощью любого контейнера с сетевыми утилитами:

```sh
kubectl run  --image=infoblox/dnstools -- sh -c 'host kubernetes'
```
```sh
pod/sh created
```

А потом в логах можно увидеть запрос:

```sh
kubectl logs sh
```
```sh
kubernetes.default.svc.cluster.local has address 10.96.0.1
```

Если же, в ответе мы не получаем IP адрес, то проблема вероятнее всего связана с DNS. Если же преобразование имени срабатывает нужно расследовать причину.

Если нод более 1, то проверить систему преобразования имен необходимо на всех `worker` нодах.

### Ошибки NodeLocalDNS

Ошибка имеет вид:

```txt
dial tcp 10.96.0.10:53: i/o timeout
```

Означает, что кеширующий DNS-сервер не может получить доступ к основному `CoreDNS`. Обычно это означает, что на ноде с которой происходит запрос имеются проблемы с внутренней сетью Kubernetes.

