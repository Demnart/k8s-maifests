# Deployment

- [Deployment](#deployment)
  - [Манифест](#манифест)
  - [Запуск Deployment](#запуск-deployment)
  - [Рестарт приложения](#рестарт-приложения)
  - [strategy](#strategy)

[Документация](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Управление подами... В предыдущем [видео про Pod](pod.md) мы рассмотрели как создать и запустить pod. И вы наверное обратили внимание на то, что:

- под не поддерживает горизонтальное масштабирование;
- при перезапуске приложения pod перезапускаются;

Т.е. используя только pod мы не сможем создать отказоустойчивую систему. Для этого нам нужен контроллер, позволяющий управлять несколькими pod'ами и обеспечивать их отказоустойчивость. Т.е брать на себя автоматизацию процесса управления жизненным циклом pod'ов. Таких контроллеров несколько: ReplicaSet, StatefulSet, Deployment и DaemonSet. В этом видео мы рассмотрим Deployment.

Deployment используется для управления подами. Приложения в подах, поддерживаемых Deployment, обычно являются stateless приложениями (не поддерживающими состояние). Т.е. свое состояния, если это необходимо, приложения сохраняют во внешних хранилищах: базах данных, файловых системах и т.д.

## Манифест

Какие значения можно использовать в манифесте, описано в документации по [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/).

Начнём писать манифест Deployment, для управления подом, который мы рассмотрели в предыдущем видео:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-first-deployment
  namespace: work
  labels:
    app: &app nginx
    version: &version 1.27.5
spec:
  replicas: 3
  selector:
    matchLabels:
      app: *app
      version: *version
  template:
```

Мы указываем версию API `apps/v1`, тип ресурса `Deployment`. Затем идет раздел `metadata`, в котором определяем имя и пространство имен, где будет создан наш Deployment.

На манифест Deployment тоже распространяются правила хорошего тона. Как и в случае Pod мы должны указать labels. Что мы и делаем ниже, где прописываем две метки: `app: nginx` и `version: 1.27.5`. Поскольку значения этих меток мы будем повторять дальше в манифесте, я определю "якорь" YAML и присвою ему значение: `app: &app nginx` и `version: &version 1.27.5`. дальше в тексте маничеста, что бы подставить значение якоря, достаточно просто указать имя якоря, например `*app` или `*version`.

В разделе `spec` мы начинаем описывать параметры самого Deployment. В первую очередь определяем количество одновременно запущенных подов: `replicas: 3`. Т.е.  будет запущено три экземпляра нашего приложения.

`selector` отвечает за то, какие поды должны быть выбраны для управления данным Deployment. В дальнейшем, при описании пода, мы присвоим ему аналогичные метки. Таким образом, Kubernetes будет знать, что эти поды принадлежат данному Deployment.

Сам под мы опишем в разделе `template`. А вот тут маленькая хитрость. Посмотрите манифест пода, который мы создали в предыдущем видео. Удалите из него два параметра: `apiVersion` и `kind`. Все что осталось, скопируйте и вставьте в манифест Deployment, в разделе `template` соблюдая отступы. В результате получается манифест Deployment.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-first-deployment
  namespace: work
  labels:
    app: &app nginx
    version: &version 1.27.5
spec:
  replicas: 3
  selector:
    matchLabels:
      app: *app
      version: *version
  template:
    metadata:
      labels:
        app: *app
        version: *version
    spec:
      initContainers:
        - name: init-myservice
          image: busybox:1.28
          command: 
            - 'sh'
            - '-c'
            - 'echo Initializing... && sleep 10'
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
      containers:
        - name: nginx
          image: nginx:1.27.5
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
```

При определении Deployment можно использовать дополнительные параметры. Некоторые из них мы рассмотрим по мере необходимости.

## Запуск Deployment

Теперь, когда у нас есть манифест для создания Deployment, можно применить его с помощью команды `kubectl apply`:

```shell
kubectl apply -f manifests/03-deployment.yaml
```

```txt
deployment.apps/my-first-deployment created
```

Посмотрим что получилось:

```shell
kubectl -n work get all
```

Используем специальный параметр all, который показывает все ресурсы в указанном namespace. В результате мы получим неожиданно много ресурсов:

```txt
NAME                                       READY   STATUS            RESTARTS   AGE
pod/my-first-deployment-676b75dd9c-j6bnw   0/1     Running           0          19s
pod/my-first-deployment-676b75dd9c-stlng   0/1     PodInitializing   0          19s
pod/my-first-deployment-676b75dd9c-zxqk5   0/1     Running           0          19s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-first-deployment   0/3     3            0           19s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-first-deployment-676b75dd9c   3         3         0       19s
```

Три пода - ожидаемо. Оди Deployment, тоже понятно. Но откуда тут какая то ReplicaSet? Давайте разбираться. Вспомним схему на которой я показывал ресурсы kubernetes и их взаимодействие.

Итак, мы положили манифест Deployment в кластер. В результате он сохранился в базе данных.

На следующем шаге сработал контроллер, отвечающий за Deployment. Он увидел что появился новый манифест и ... Нет, не начал создавать поды. Он создаёт ещё одну управляюзую конструкцию - ReplicaSet. Т.е. он создаёт манифест для ReplicaSet и помещает в базу данных.

Задача ReplicaSet - обеспечить работу заданного количества реплик Pod'ов, описываемых Deployment.

Далее контроллер, отвечающий за ReplicaSet видит что появился манифест с ReplicaSet и начинает создавать поды. Он создает манифесты подов. Столько манифестов, сколько указано в Deployment. В нашем случае - 3.

После того как в базе данных появились манифесты для Pod'ов, в работу вступает планировщик, распределяющий их по узлам кластера.

И в конце концов - kubelet запускает поды на соответствующих нодах кластера.

Обратите внимание на формирования имен ReplicaSet: `my-first-deployment-<hash>`. Они связаны с названием Deployment. А имена подов содержат в себе имя ReplicaSet, а также уникальный идентификатор пода. Т.е. понять, кто кем управляет можно по именам ресурсов.

Deployment понимает, каким ReplicaSet он управляет, при помощи labels, которые мы указали в манифесте Deployment. У ReplicaSet тоже есть метки, определяющие какими подами он управляет. Мы можем их увидеть в манифесте ReplicaSet:

```shell
kubectl -n work get replicaset.apps/my-first-deployment-676b75dd9c --output=jsonpath={.spec.selector} | jq
```

```json
{
  "matchLabels": {
    "app": "nginx",
    "pod-template-hash": "676b75dd9c",
    "version": "1.27.5"
  }
}
```

Кроме прямой связи через labels, от Deployment к ReplicaSet. И от ReplicaSet к Pods существует обратная связь. От Pod к ReplicaSet и от ReplicaSet к Deployment. Она осуществляется через поля `.metadata.ownerReferences`:

```shell
kubectl -n work get replicaset.apps/my-first-deployment-676b75dd9c --output=jsonpath={.metadata.ownerReferences} | jq
```

```json
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "Deployment",
    "name": "my-first-deployment",
    "uid": "3f202fec-4ca1-4035-8ebd-0b6743a8b7e9"
  }
]
```

Аналогичная обратная ссылка есть и у подов:

```shell
kubectl -n work get pod/my-first-deployment-676b75dd9c-j6bnw --output=jsonpath={.metadata.ownerReferences} | jq
```

```yaml
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "ReplicaSet",
    "name": "my-first-deployment-676b75dd9c",
    "uid": "feb84e15-22b7-4129-b931-1717cb8807da"
  }
]
```

Разумеется, для получения информации о Deployment можно использовать `kubectl describe`.

## Рестарт приложения

Рестарт приложения можно произвести несколькими способами:

- Изменив содержимое шаблона пода в Deployment.
- В ручную с помощью команды `kubectl rollout restart`.

В предыдущем видео мы пытались поменять манифест пода и получали сообщение об ошибке. И я говорил, что при, например изменении версии контейнера, придется перезапускать под. Deployment берет на себя ответственность за перезапуск подов.

В качестве примера, изменим версию nginx на предыдущую. Это можно сделать при помощи `kubectl set image deployment/my-first-deployment nginx=nginx:1.27.5`. Но это не лучший вариант. Мы управляем kubernetes при помощи манифестов. Манифесты где то храним, например в git. Предполагается, что в git находится то состояние кластера, которое нам необходимо. И если мы будем что то менять не используя манифесты, в git будут недостоверные данные. Поэтому, лучше один раз написать/изменить манифест в git, чем потом полдня потерять на поиск ошибки. *Обычно для управления состоянием кластера используют системы Continue Delivery (CD), такие как ArgoCD, Flux и другие. Они поддерживают состояния кластера в соответствии с дананными в git репозитории.*

Я создам новый манифест, а вы можете изменить текущий. В котором укажу новый образ контейнера. И применю его:

```shell
kubectl apply -f manifests/04-deployment-newcontainer.yaml
```

```txt
deployment.apps/my-first-deployment configured
```

Посмотрим, практически в реальном времени как происходил рестарт подов:

```shell
watch kubectl -n work get all
```

Выйти из режима просмотра можно нажав клавишу `Ctrl+C`.

Итоговый результат:

```shell
kubectl -n work get all
```

```txt
NAME                                      READY   STATUS    RESTARTS   AGE
pod/my-first-deployment-8c6989bc8-cr6sl   1/1     Running   0          2m5s
pod/my-first-deployment-8c6989bc8-mj4fr   1/1     Running   0          2m56s
pod/my-first-deployment-8c6989bc8-scdqb   1/1     Running   0          3m41s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-first-deployment   3/3     3            3           4m56s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-first-deployment-676b75dd9c   0         0         0       4m56s
replicaset.apps/my-first-deployment-8c6989bc8    3         3         3       3m41s
```

Что мы видим? Появился новый экземпляр `replicaset.apps/my-first-deployment-8c6989bc8`. Т.е. старый replicaset удалил все поды, которые он обслуживал. Новые поды были запущены новым replicaset. Но старый replicaset будет удалён автоматически только после того, как накопится 10 экземпляров (*значение по умолчанию*). *Накапливать такое количество не имеет смысла. Поэтому можно ограничить их, например тремя экземплярами, при помощи параметра `revisionHistoryLimit: 3`*.

Зачем так много? И зачем в обще их накапливать? Предполагается, что таким образом у вас будет возможность откатиться на предыдущий набор подов, если новый вас не устраивает. Я не буду показывать эту возможность потому что - это опять ручное управление, без сохранения состояния в git. Если вам не понравилось, то что получилось - откатитесь в git на предыдущий commit и примените предыдущий манифест Deployment.

Еще одна неприятность. Я поменял версию контейнера, но не поменял метки. Если я поменяю метки, то при деплое манифеста мы получим сообщение об ошибке: `The Deployment "my-first-deployment" is invalid: spec.selector: Invalid value: v1.LabelSelector{MatchLabels:map[string]string{"app":"nginx", "version":"1.27.4"}, MatchExpressions:[]v1.LabelSelectorRequirement(nil)}: field is immutable`. Т.е. нам всё равно при изменение Deployment придётся кго сначала удалять и создавать по новой. Поэтому я поменял только версию контейнера не изменяя метки.

Вернёмся к прежней версии:

```shell
kubectl apply -f manifests/03-deployment.yaml
```

Если вам, по каким либо причинам необходимо перезапустить приложение. Например, для того, что бы оно перечитало свои конфигурационные файлы. Воспользуйтесь командой:

```shell
kubectl -n work rollout restart deployment my-first-deployment
```

```txt
deployment.apps/my-first-deployment restarted
```

## strategy

Вы наверное обратили внимание, как контроллер Deployment обновляет приложение. По умолчанию он использует стратегию `.spec.strategy.type: RollingUpdate`. Это означает, что обновление происходит постепенно, старые поды заменяются новыми по одному. У этой стратегии есть два конфигурационных параметра:

- `.spec.strategy.rollingUpdate.maxUnavailable` - это необязательное поле, в котором указывается максимальное количество подов, которые могут быть недоступны в процессе обновления. Значение может быть абсолютным числом (например, 5) или процентом от желаемого количества модулей (например, 10%). Абсолютное число рассчитывается из процента путем округления в меньшую сторону. Значение не может быть равно 0, если `.spec.strategy.rollingUpdate.maxSurge` равно 0. Значение по умолчанию — 25%.
- `.spec.strategy.rollingUpdate.maxSurge` - это необязательное поле, в котором указывается максимальное количество подов, которые могут быть созданы сверх желаемого количества подов. Значение может быть абсолютным числом (например, 5) или процентом от желаемого количества модулей (например, 10%). Значение не может быть равно 0, если `MaxUnavailable` равно 0. Абсолютное число рассчитывается на основе процента путем округления в большую сторону. Значение по умолчанию — 25%.

Но в редких случаях может потребоваться полностью удалить все экземпляры приложения, прежде чем запускать новые поды. (*Привет любителям Python!*) В таких ситуациях можно использовать стратегию `Recreate`.