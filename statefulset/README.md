# StatefulSet

В отличии от `Deployment`, [`StatefulSet`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) гарантирует уникальность имени каждого podа в кластере Kubernetes. И строго определяет порядок запуска, обновления и удаления подов.

## Пример конфигурации StatefulSet

Пример манифеста `StatefulSet`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: test-st
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 3
  serviceName: "test-st-headless"
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
```

Раздел `spec` по сути не отличается от такого же раздела в Deployment. Единственное дополнение - это указание так называемого headless service при помощи параметра `serviceName`. Этот сервис используется для формирования уникального сетевого имени каждого пода StatefulSet в пространстве имен кластера. *Подробнее о механизме работы headless service мы поговорим позднее.*

Сейчас мы не будем рассматривать работу с томами (накопителями) в кластере kubernetes. Но на будущее запомним, что для каждого пода в `StatefulSet` можно создать отдельный том при помощи параметра `volumeClaimTemplates`, который будет "привязан" к конкретному поду.

По аналогии с `Deployment`, будет создан сервис:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-st
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
  selector:
    app.kubernetes.io/name: *name
    app.kubernetes.io/instance: *instance
    app.kubernetes.io/version: *version
  ports:
    - port: 80
```

Также необходимо создать Headless сервис. Он отличается от обычного тем, что у него значение поля `clusterIP` установлено в `None`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-st-headless
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
  clusterIP: None
  selector:
    app.kubernetes.io/name: *name
    app.kubernetes.io/instance: *instance
    app.kubernetes.io/version: *version
  ports:
    - port: 80
```

Именно этот сервис указывается в манифесте `StatefulSet`.

Запустим StatefulSet с помощью команды:

```shell
kubectl -n work apply -f statefullset.yaml
```

```txt
service/test-st created
service/test-st-headless created
statefulset.apps/test-st created
```

Посмотрим содержимое namespace work:

```shell
kubectl -n work get all
```

```txt
NAME            READY   STATUS    RESTARTS   AGE
pod/test-st-0   1/1     Running   0          43s
pod/test-st-1   1/1     Running   0          39s
pod/test-st-2   1/1     Running   0          35s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/test-st            ClusterIP   10.233.47.69   <none>        80/TCP    43s
service/test-st-headless   ClusterIP   None           <none>        80/TCP    43s

NAME                       READY   AGE
statefulset.apps/test-st   3/3     43s
```

Первое отличие от `Deployment` - отсутствие `ReplicaSets`. `SatefulSet` самостоятельно управляет pod'ами.

Второе отличие - это имена подов. Как видите их имена содержат суффиксы: `0`, `1`, `2`. Т.е. мы точно знаем какое имя будет у каждого пода.

Посмотрим как работает обычный сервис. Запустим одноразовый под с контейнером, содержащим `curl`:


```shell
kubectl -n work run curl --rm -it --image=alpine/curl:8.14.1 -- /bin/sh
```

Парамер `--rm ` означает, что под будет удален после выполнения команды, а `-it` позволяет войти в этот под и выполнять команды в интерактивном режиме. Команда интерактивного режима указана после `--`.

У нас появится командная строка, в которой несколько раз выполним команду `curl test-st`. Таким образом мы обратимся к сервису `test-st` в том же namespace.

Я покажу только часть ответа приложения `StatefulSet` к которому мы обращаемся:

```txt
# curl test-st
Hostname: test-st-1
IP: 127.0.0.1
IP: ::1
IP: 10.233.124.133

# curl test-st
Hostname: test-st-0
IP: 127.0.0.1
IP: ::1
IP: 10.233.109.195

# curl test-st
Hostname: test-st-2
IP: 127.0.0.1
IP: ::1
IP: 10.233.109.201
```

Как видите, каждый раз мы получаем ответ от разных Pod'ов. Это происходит потому, что сервис `test-st` распределяет запросы между всеми доступными Pod'ами.

Теперь попробуем использовать headless сервис:

```txt
# curl test-st-0.test-st-headless
Hostname: test-st-0
IP: 127.0.0.1
IP: ::1
IP: 10.233.109.195

# curl test-st-1.test-st-headless
Hostname: test-st-1
IP: 127.0.0.1
IP: ::1
IP: 10.233.124.133

# curl test-st-2.test-st-headless
Hostname: test-st-2
IP: 127.0.0.1
IP: ::1
IP: 10.233.109.201
```

Как видно из примера, при помощи headless сервиса мы можем получить IP адрес конкретного Pod'а.

Не забудьте выйти из пода и завершить сессию при помощи команды `exit`.

```txt
# exit
Session ended, resume using 'kubectl attach curl -c curl -i -t' command when the pod is running
pod "curl" deleted
```

## Порядок запуска, обновления и удаления подов

Контроллер `StatefulSet` гарантирует определенный порядок запуска подов. Они запускаются в порядке возрастания индекса, начиная с 0. После того как запущен Pod с индексом 0, начинается запуск следующего Pod'. И так по порядку до последнего, третьего Pod'а.

Удаление подов происходит в обратном порядке. Сначала удаляется последний Pod `test-st-2`, а затем `test-st-1` и, наконец, `test-st-0`.

Если в процессе запуска, например, перед стартом `test-st-2`, по каким то причинам упадёт `test-st-0`. `test-st-2` не будет запущен до тех пор, пока не будет успешно запущен `test-st-0`.

При выполнении `rollout restart` - перезапуск подов происходит в обратном порядке. Сначала перезапускается последний Pod, а затем предыдущий.

Посмотрим как реагирует StatefulSet на команду `rollout restart`.

```shell
kubectl -n work rollout restart statefulset test-st
```

Если вы хотите, что бы порядок старта, удаление и рестарта подов `StatefulSet` не зависел от времени или других факторов, можно переопределить `.spec.podManagementPolicy`. *(Его значение по умолчанию: `OrderedReady`)*. Используйте `Parallel`, когда порядок старта и завершения не важен.

```shell
kubectl -n work delete -f manifests/06-statefullset.yaml
```

```txt
service "test-st" deleted
service "test-st-headless" deleted
statefulset.apps "test-st" deleted
```

```shell
kubectl -n work apply -f manifests/07-statefullset-parallel.yaml
```

```txt
service/test-st created
service/test-st-headless created
statefulset.apps/test-st created
```

Удалим тестовый `StatefulSet` и сервисы:

```yaml
kubectl -n work delete -f manifests/07-statefullset-parallel.yaml
```

```text
service/test-st created
service/test-st-headless created
statefulset.apps/test-st created
```