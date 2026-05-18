# PodDisruptionBudget

[PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) PDB ограничивает количество подов реплицируемого приложения, в случае создания другими приложениями [Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/).

Eviction удаляет под с ноды кластера в соответствии с определенными политиками (в том числе PDB) и ограничениями безопасности. Это подресурс API пода:
`/pod/<pod name>/evictions`.  

У PDB есть два параметра:

  - `maxUnavailable` - это максимальное количество подов, которые могут быть недоступны одновременно.  
  - `minAvailable` - это минимальное количество подов, которое должно быть достпно.  

Значение этих параметров могут быть заданы как целое число или процентное соотношение. Например, `minAvailable: 2` или `minAvailable: 80%`.

Если используются проценты, при вычислении количества подов, количество окргляется в большую сторону.  

Одновременно в PDB  можно использовать только один из параметров. Или `maxUnavailable` или `minAvailable`.  

Когда инициируется операция, которая может нарушить работу подов( например, kubectl drain), Contol Plane Kubernetes проиверяет все PDB в кластере. Если удаление пода нарушает условия PDB(например, уменьшает количество допуных подов ниже minAvailable или превышает maxUnavailable), операция блокируется до тех пор, пока не будет снова соблюдено. Это обеспечивает поэтапное удаление подов без критического снижения доступности сервиса.  

Пример `PodDisruptionBudget` в фале `pdb.yml`.

Тестовый `Deployment`: 

```yml
---
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testdp
  labels:
    app.kubernetes.io/name: &name testdp
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: *name
      app.kubernetes.io/instance: *instance
      app.kubernetes.io/version: *version
  template:
    metadata:
      labels:
        app.kubernetes.io/name: *name
        app.kubernetes.io/instance: *instance
        app.kubernetes.io/version: *version
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - *name
              - key: app.kubernetes.io/version
                operator: In
                values:
                - *version
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: whoami
        image: traefik/whoami
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        resources:
          limits:
            memory: "100Mi"
            cpu: "100m"
          requests:
            memory: "100Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: testdp-pdb
  labels:
    app.kubernetes.io/name: &name testdp
    app.kubernetes.io/instance: &instance testdp
    app.kubernetes.io/version: &version v0.0.1
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: *name
      app.kubernetes.io/instance: *instance
      app.kubernetes.io/version: *version
```

При помощи `spec.selector.matchLabels` определяется набор меток, установленных на поды, на которые будет распространятся PDB. В данном случае, PDB будет ожидать, чтобы наше приложение было доступно хотяб на 1 из 2 подов. Если хотя бы один из подов будет недоступн, PDB будет предотвращать выселение с ноды. 

Применим манифест:  

```sh
kubectl -n work apply -f pdb.yml
```

Смотрим состояние PDB:

```sh
kubectl -n work get pdb
```
```sh
NAME         MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
testdp-pdb   1               N/A               1                     88s
```

Сначала запретим распределение новых подов на ноду worker2:

```sh
kubectl cordon worker2
```
```sh
node/worker2 cordoned
```
```sh
kubectl get nodes
```
```sh
NAME      STATUS                     ROLES           AGE   VERSION
control   Ready                      control-plane   8d    v1.35.4
worker1   Ready                      <none>          8d    v1.35.4
worker2   Ready,SchedulingDisabled   <none>          8d    v1.35.4
```

Теперь принудительно очистим ноду от подов:  

```sh
kubectl drain worker2 --ignore-daemonsets --delete-emptydir-data
```
```sh
node/worker2 already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/cilium-envoy-d7xrw, kube-system/cilium-v8wrh, kube-system/node-local-dns-rt6dd
evicting pod work/testdp-f9d6d65fd-ds2ms
evicting pod kube-system/hubble-ui-67d8bff4c4-dfrjl
pod/hubble-ui-67d8bff4c4-dfrjl evicted
pod/testdp-f9d6d65fd-ds2ms evicted
node/worker2 drained
```

При этом при попытке выполнить drain для worker1 мы получим ошибку, что под testdp не может быть выселен с worker1 ноды, т.е. `PDB` не дает удалить под с ноды и соответственно завершить `drain`.

Отменим `cordon` на ноде:  

```sh
kubectl uncordon worker2
```
```sh
node/worker2 uncordoned
```
```sh
kubectl get nodes
```
```sh
NAME      STATUS   ROLES           AGE   VERSION
control   Ready    control-plane   8d    v1.35.4
worker1   Ready    <none>          8d    v1.35.4
worker2   Ready    <none>          8d    v1.35.4
```

Проверим, что под запустился успешно на worker2:  
```sh
 kubectl -n work get po
```
```sh
NAME                     READY   STATUS    RESTARTS   AGE
testdp-f9d6d65fd-v2hsd   1/1     Running   0          14m
testdp-f9d6d65fd-vv496   1/1     Running   0          6m35s
```

> `PodDisruptionBudget` не гарантирует, что указанное количество/процентное соотношение подов всегда будет в рабочем состоянии. Он может защитить только от добровольного выселения подов, но не от всех причин недоступности.  

Применять PDB необходимо "с умом". Не стоит использовать PDB ко всем приложениям. Например приложения с одной репликой. Вы не сможете перезапустить такое приложение если `minAvailable` будет равно 1 и более. Конечно может сделать `minAvailable: 0`. Но тогда зачем нужен PDB? 

Так же не используйте PDB для Jobs. Его под должен завершить работу в случае успешного завершения задания. 

Основные рекомендации по применению PDB можно найти в [документации](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)

## Нюансы

Попробуем уменьшить количество подов до 1:

```sh
kubectl -n work scale deployment testdp --replicas=1
```
```sh
kubectl -n work get po
```
```sh
NAME                     READY   STATUS    RESTARTS   AGE
testdp-f9d6d65fd-v2hsd   1/1     Running   0          28m
```

И мы увидим, что PDB  не ограничивает эту команду. Хотя, количество подов меньше, чем мы указали в PDB: `minAvailable: 1`.  

PDB не блокирует маштабирование. Как указано в документации, PDB ограничивает только eviction через Eviction API(например при drain узла), но не удаление Pod'ов контроллерами(Например Deployment при маштабировании).

Удалим PDB и `Deployment`: 

```sh
kubectl -n work delete -f pdb.yml 
```
```sh
deployment.apps "testdp" deleted
poddisruptionbudget.policy "testdp-pdb" deleted
```