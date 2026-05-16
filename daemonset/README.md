# DaemonSet

- [DaemonSet](#daemonset)
  - [Пример конфигурации DaemonSet](#пример-конфигурации-daemonset)
  - [Зараженные ноды (Taints и Tolerations)](#зараженные-ноды-taints-и-tolerations)
  - [NodeSelector](#nodeselector)


[`DaemonSet`](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) в Kubernetes используется для запуска одного экземпляра podа на каждой ноде кластера. Он идеально подходит для задач, которые требуют выполнения агентов или сервисов на всех или некоторых узлах кластера.

## Пример конфигурации DaemonSet

[Пример манифеста `DaemonSet`](daemonset.yml):

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-ds
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
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
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
              name: http
              protocol: TCP
          resources:
            limits:
              cpu: 100m
              memory: 100Mi
            requests:
              cpu: 100m
              memory: 100Mi
          livenessProbe:
            httpGet:
              path: /
              port: http
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: http
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 10
```

Манифест `DaemonSet` полностью копирует структуру манифеста `Deployment`. Единственное отличие - в отсутствии поля `.spec.replicas`, так как для `DaemonSet` это поле не применимо.

Обратите внимание, что в манифесте я не определяю `Service`. Это сделано намеренно. Предполагается, что приложение, которое запускается при помощи `DaemonSet`, предназначено только для сбора и отсылки некой информации. В качестве примера можно привести системы сбора логов в k8s. Там на каждой ноде запускается агент, который читает логи, находящиеся в директории `/var/log`. И отправляет их куда-то. 

Или сбор метрик при помощи node-exporter. Точно так же, на каждой ноде кластера запускается агент, который собирает данные о машине и отдаёт их по запросу агентов сбора метрик. Агенты обращаются непосредственно к отдельному экземпляру node-exporter, не использую `Service`. (*Как он узнаёт этот IP - это отдельная история, о которой мы поговорим позже.*)

Применим манифест:

```shell
kubectl -n work apply -f daemonset.yml
```

```txt
daemonset.apps/test-ds created
```

Посмотрим, что мы получили:

```shell
kubectl -n work get all
```

```txt
NAME                READY   STATUS    RESTARTS   AGE
pod/test-ds-mg5kw   1/1     Running   0          110s
pod/test-ds-npfx9   1/1     Running   0          110s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/test-ds   2         2         2       2            2           <none>          110s
```

## Зараженные ноды (Taints и Tolerations)

Посмотрим внимательно на поды:

```shell
kubectl -n work get pods -o wide
```

```txt
NAME            READY   STATUS    RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
test-ds-mg5kw   1/1     Running   0          3m33s   10.233.109.235   worker1              <none>           <none>
test-ds-npfx9   1/1     Running   0          3m33s   10.233.124.184   worker2              <none>           <none>
```

Вспомним, какие ноды есть в нашем кластере:

```shell
kubectl get nodes
```

```txt
NAME                STATUS   ROLES           AGE   VERSION
control             Ready    control-plane   36d   v1.33.1
worker1             Ready    worker          36d   v1.33.1
worker2             Ready    worker          36d   v1.33.1
```

Что то не сходится. По идее поды DaemonSet должны быть запущены на всех нодах, а здесь видно, что они запущены только на двух.

При назначении пода на ноду, на работу планировщика влияют различные факторы. В нашем случае на control ноды кластера были установлены Taints (зараза). Посмотреть какие taints установлены можно командой:

```shell
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

```txt
NAME                TAINTS
control             [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
worker1             <none>
worker2             <none>
```

Как видно из вывода команды, на ноде `control` установлен taints: `node-role.kubernetes.io/control-plane`.

По умолчанию планировщик не распределяет поды на "зараженные" ноды. Поэтому поды нашего DaemonSet были созданы только на worker нодах.

Для того, чтобы поды могли запускаться на "зараженной" ноде, необходимо добавить толерантность к конкретному типу заразы.

Зараза состоит из трёх частей: `key=[value]:Effect`. В нашем случае это

* `key` - ключ taint (например, node-role.kubernetes.io/control-plane). [Список "стандартных" ключей](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#taints-and-tolerations). Если нужно - можно определить свой ключ.
* `value` - значение taint. Не обязателен к определению. Если не указано, то любое значение будет считаться совпадением.
* `Effect` - действие.
  * `NoSchedule` - запрещает планирование под на ноде. Поды, запущенные до применения taint не удаляются.
  * `NoExecute` - запрещает планирование под на ноде. Поды, запущенные до применения taint будут удалены с ноды.
  * `PreferNoSchedule` - это «предпочтительная» или «мягкая» версия NoSchedule. Планировщик будет пытаться не размещать на узле поды, но это не гарантировано.
  
Что бы игнорировать taint `node-role.kubernetes.io/control-plane:NoSchedule` для подов DaemonSet необходимо добавить в манифест толерантность к конкретному типу заразы в спецификации пода. В нашем случае это будет выглядеть так:

```yaml
spec:
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
```

Если мы не указываем значение ключа (value), operator должен быть установлен в `Exists`. Итоговый манифест мы поместим в файл `daemonset-tolleration.yml`.

Применить изменения без удаления текущего манифеста не получится. Поэтому сначала удалим старый манифест:

```shell
kubectl -n work delete -f daemonset.yml
```

```txt
daemonset.apps "test-ds" deleted
```

Применим новый манифест:

```shell
kubectl -n work apply -f daemonset-tolleration.yml
```

```txt
daemonset.apps/test-ds created
```

Проверим, что все работает корректно:

```shell
kubectl -n work get pods -o wide
```

```txt
NAME            READY   STATUS    RESTARTS   AGE   IP               NODE                NOMINATED NODE   READINESS GATES
test-ds-bcrll   1/1     Running   0          67s   10.233.81.107    control             <none>           <none>
test-ds-vx8t8   1/1     Running   0          67s   10.233.109.240   worker1             <none>           <none>
test-ds-zxmdg   1/1     Running   0          67s   10.233.124.181   worker2             <none>           <none>
```

Как видим все получилось.

Добавим свою собственную заразу на вторую worker ноду.

```shell
kubectl taint nodes worker2 test-taint=:NoExecute
```

```txt
node/worker tainted
```

Посмотрим на поды:

```shell
kubectl -n work get pods -o wide
```

```txt
NAME            READY   STATUS    RESTARTS   AGE   IP               NODE                NOMINATED NODE   READINESS GATES
test-ds-bcrll   1/1     Running   0          67s   10.233.81.107    control             <none>           <none>
test-ds-vx8t8   1/1     Running   0          67s   10.233.109.240   worker1             <none>           <none>
```

Поскольку мы использовали effect `NoExecute`, под находившийся ранее на этой ноде был удален.

Уберем taint с ноды:

```shell
kubectl taint nodes worker2 test-taint=:NoExecute-
```

```txt
node/worker2 untainted
```

```shell
kubectl -n work get pods -o wide
```

```txt
NAME            READY   STATUS    RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
test-ds-bcrll   1/1     Running   0          6m43s   10.233.81.107    control             <none>           <none>
test-ds-ndqnp   1/1     Running   0          45s     10.233.124.182   worker1             <none>           <none>
test-ds-vx8t8   1/1     Running   0          6m43s   10.233.109.240   worker2             <none>           <none>
```

Effect `NoExecute` опасный. Он удалить с ноды все поды, не имеющии толерантности к taint. На самом деле на этой норде были некоторые служебные поды, которые не желательно удалять. Поэтому рекомендуется использовать `NoSchedule`.

Удалим DaemonSet:

```shell
kubectl -n work delete -f daemonset-tolleration.yml
```

```txt
daemonset.apps "test-ds" deleted
```

## NodeSelector

Использовать taints для распределение подов DaemonSet не лучший способ управления. Например нам необходимо разместить поды на строго определённых нодах кластера. В этом случае можно использовать `nodeselector`. В качестве параметра, используемого для отбора нод, можно указать метки (labels), установленные на нодах.

Давайте пометим ноды c1 и worker2 меткой `special: ds-only`:

```shell
kubectl label nodes control special=ds-only && \
kubectl label nodes worker2 special=ds-only
```

```txt
node/control labeled
node/worker2 labeled
```

Посмотрим нак каких нодах установлены метки `special: ds-only`:

```shell
kubectl get nodes --show-labels | grep 'special=ds-only'
```

```txt
control   Ready    control-plane   36d   v1.33.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=control,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=,special=ds-only
worker2   Ready    worker          36d   v1.33.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker,special=ds-only
```

Добавим `nodeSelector` в определение пода:

```yaml
spec:
  nodeSelector:
    special: ds-only
```

Применим манифест:

```shell
kubectl -n work apply -f daemonset-ns.yml
```

```txt
daemonset.apps/test-ds created
```

Посмотрим список подов в кластере:

```shell
kubectl -n work get pods -o wide
```

```txt
NAME            READY   STATUS    RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
test-ds-2s5k7   1/1     Running   0          6m37s   10.233.124.141   worker2             <none>           <none>
test-ds-n9bvn   1/1     Running   0          6m37s   10.233.81.106    control             <none>           <none>
```

Удалим метки с нод:

```shell
kubectl label nodes control special- && \
kubectl label nodes worker2 special-
```

```txt
node/control unlabeled
node/worker2 unlabeled
```

Посмотрим на поды DaemonSet после удаления меток:

```shell
kubectl -n work get ds
```

```txt
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR     AGE
test-ds   0         0         0       0            0           special=ds-only   10m
```

Удалим манифест. Всё равно в кластере нет ни одной ноды с необходимым набором меток.

```shell
kubectl -n work delete -f 10-daemonset-ns.yml 
```

```txt
daemonset.apps "test-ds" deleted
```

Толерантность и nodeSelector определяются в шаблоне пода. Это значит, что их можно использовать не только в DaemonSet'ах, но и в Deployment'ах, StatefulSets и других контроллерах Pod.