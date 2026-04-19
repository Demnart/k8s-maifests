***Кластер***  
Состоит из 1 контрольной ноды и 2 воркер нод: 

```sh
NAME      STATUS   ROLES           AGE   VERSION
control   Ready    control-plane   36m   v1.35.4
worker1   Ready    <none>          35m   v1.35.4
worker2   Ready    <none>          35m   v1.35.4
```

***nodeName***

Простейщий параметр который влияет на то, где именно будут размещены поды  
Указывается в спецификации пода:  
```yaml
spec:  
  containers:  
  nodeName: worker1
```

Удобно для запуска подов для отладки.

***Affinity***

Существует 3 разновидности:  
- Node Affinity - определяет предпочтение размещения подов на нодах кластера
- Pod Affinity - определяет правила размещения подов рядом с другими подами
- Pod Anti-affinity - определяет правила, не разрешающие размещения подов на нодах, где уже запущены такие поды

**Запрет размещения подов**  

Самый распростарнённый вариан испольлзования - это Pod Anti-affinity. Когда нам необходимо запретить размещения двух подов одной управляющей конструкции  
Deployment || StatefulSet на одной ноде кластера. 

Пример ```01-podeaffinity.yml```  
```yaml
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - testdp
              - key: app.kubernetes.io/version
                operator: In
                values:
                - v0.0.1
```

В примере в спецификации пода мы определяем раздел ```affinity```, в котором объявляем ```podAntiAffinity```  
Дальше объясняем планировщику насколько "важен" данный вариант ```podAntiAffinity```. Возможны два варианта:  
- ```requiredDuringSchedulingIgnoredDuringExecution``` - обязательный к испольнению. Усли не удаётся подобрать ноду, удовлетворяющую данному соответсвию - под создан не будет
- ```preferredDuringSchedulingIgnoredDuringExecution``` - мягкий вариант. Попытается найти нод, подходящую для выполнения соответствия, Если нода не будет найдена - всё равно запустит под, на какой-либо ноде  

Затем указываем список правил. У нас будет одно правило, определяющее по каким меткам подов мы будем ориентироваться.

```topologyKey``` - определям ноды на которых будет работать affinity. Мы указываем kubernetes.io/hostname этот label всегда устанвливается на ноды кластера автоматически  

В манифесте у нас указано количество реплик 2. Те по одной реплике на ноду.  
Попробуем увеличить количество реплик с помощью команды:

```kubectl -n work scale --replicas=3 depoloyment testdp``` - У нас все так же останеться 3 пода, но при этом 3 под будет в состоянии Pending, тк ```podAntiAffinity``` запрещает размещать на ноде более одного пода со списком меток ```app.kubernetes.io/name: testdp``` и ```app.kubernetes.io/version: v0.0.1```  

```kubectl -n work get pods```  
```sh
NAME                      READY   STATUS    RESTARTS   AGE
testdp-7d948bd89d-56wdm   0/1     Pending   0          3m53s
testdp-7d948bd89d-ntr6r   1/1     Running   0          5m54s
testdp-7d948bd89d-spjdn   1/1     Running   0          5m54s
```

Мы видим, что 2 пода запущены. А вот 3-й находится в состоянии ```Pending```. Посмотрим на него подробнее:  
```kubectl -n work events --for pod/testdp-7d948bd89d-56wdm```  
```sh
LAST SEEN               TYPE      REASON             OBJECT                        MESSAGE
2m44s (x2 over 7m55s)   Warning   FailedScheduling   Pod/testdp-7d948bd89d-56wdm   0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) didn't match pod anti-affinity rules. no new claims to deallocate, preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
```  

Удалим более не нужны Deployment  

```kubectl -n work delete deployments testdp```  
```deployment.apps "testdp" deleted```  

***NodeAffinity***

При помощи nodeAffinity мы можем опредлять на какие ноды кластера будут размещатся под.  
Для эксперимента мы поделим наш кластер на две зоны ```Africa``` и ```Australia```. Для этого поставим соответсвующие метки на ноды кластера.
```sh
kubectl label nodes worker1 topology.kubernetes.io/zone=Africa
kubectl label nodes worker2 topology.kubernetes.io/zone=Australia
```  
```sh
node/worker1 labeled
node/worker2 labeled
```  

Наш пример ```02-nodeaffinity.yml``` 

Пример как выглядит nodeAffinity:
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: topolgy.kubernetes.io/zone
              operator: In
              values:
              - Africa
```

Проверим где был размещен под:  
```kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'```  
```sh
Name                      Node
testdp-7c4d8d5689-92l5n   worker1
```  

Увеличим количество подов до 2 с помощью:  
```kubectl scale --replicas=2 deployment testdp```  
и как видим 2 под не может быть расположен нигде:  
```sh
ks get po
NAME                      READY   STATUS    RESTARTS   AGE
testdp-7c4d8d5689-92l5n   1/1     Running   0          3m18s
testdp-7c4d8d5689-q6mwx   0/1     Pending   0          44s
```  

*** Вес ***  

При использовании правил affinity можно указывать какой из правил имеет больше преимущество. Для этого правилам можно указать их ```weight```(вес).
Посмотим направил ```affinty``` в новом примере ```03-nodeaffinityweight.yml```:  
```yaml
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: kubernetes.io/os
                  operator: In
                  values:
                  - linux
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              preference:
                matchExpressions:
                - key: topology.kubernetes.io/zone
                  operator: In
                  values:
                  - Africa
            - weight: 50
              preference:
                matchExpressions:
                - key: topology.kubernetes.io/zone
                  operator: In
                  values:
                  - Australia
```  

Распределение идет по максимальному весу, те на той ноде где больше вес будут распределены поды.  

применим манифест:  
```kubectl -n work apply -f 03-nodeaffinityweight.yml```  

Проврим на какой ноде был размещен под:  

```kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'```  
```sh
Name                      Node
testdp-6796b688d7-699v4   worker2
```  
При добавлении второго пода:  
```kubectl -n work scale --replicas=2 deployment testdp```  
Можно заметить, что под был размещен на ноде с label Africa:  
``` kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'```  
```sh
Name                      Node
testdp-6796b688d7-699v4   worker2
testdp-6796b688d7-jvxkr   worker
```


***Pod Topology Spread Constaints```

В предыдущем примере мы распределяли поды по зонам.Но, как вы заметили, поды сначала добавлялись в одной зоне. Потом когда в этой зоне не осталось свободных нод, поды начали создаваться во второй зоне.  

Как сделать, чтобы поды равномерно распределялись между нодами зоны? Причём, что бы не было перекосов, когда в одной зоне подов значительно больше, чем в другой, при одинаковом количестве нод в каждой зоне.  

[Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) - опредяет набор правил управления распределения подов между доменами сбоя, такими как регионы, зоны, узлы и другие пользовательские домены топологии.

Изменим наш манифест. Удалим из него раздел ```afffinity```. В спецификации пода добавим ```topologySpreadContstraints```.  

```yaml
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: *name
            app.kubernetes.io/instance: *instance
            app.kubernetes.io/version: *version
        nodeAffinity: Ignore
        nodeTaintsPolicy: Honor
```  

Параметры ```topologySpreadConstraints```:  
- ```maxSkew``` - максимальная разница количества подов между доменами топологии  
- ```topologyKey``` - метка на ноде кластера, которая используется для определения доменов топологии  
- ```whenUnsatisfiable``` - что делать с подом, если он не соответствует ограничению.   
  - ```DoNotSchedule``` - (по умолчанию) запрещает планировщику запускать под на ноде  
  - ```ScheduleAnyway``` - разрещает запускать под на ноде.  
- ```labelSelector```- определяет список меток подов, попадающих под это правило  
- ```nodeAffinityPolicy``` - определяет будут ли учитываться ```nodeAffinity```/```nodeSelector``` пода при расчёте неравномерности распределения пода  
  - ```Honor``` - (по умолчанию) в расчёт включаются только ноды, соответствующие ```nodeAffinity```/```nodeSelector```  
  - ```Ignore``` - в расчёты включены все ноды  
- ```nodeTaintsPolicy```- аналогично ```nodeAffinityPolicy```, только учитываются ```Taints```.  
  - ```Honor```- Включаются ноды без установленных ```Taints```, а так же ноды для которых у пода есть ```Toleration```  
  - ```Ignore```- (по умолчанию) в расчёты включены все ноды  

Запускаем Deployment:  
```kubectl apply -f 04-podtopolyspreadcontstraints.yml```  
```deployment.apps/testdp created```  
``` kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'```  
```sh
Name                      Node
testdp-5668f9f5f5-ftdrt   worker1
```  
Под был размещен на "Африке". Добавим еще под.  
``` kubectl -n work scale --replicas=2 deployments testdp```  
```kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'```
```sh
Name                      Node
testdp-5668f9f5f5-5gg6s   worker2
testdp-5668f9f5f5-ftdrt   worker1
```  

Увеличим количество подов до 4-х:  
```kubectl -n work scale --replicas=4 deployments testdp```  
Проверяем:  
```kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'```  
```sh
Name                      Node
testdp-5668f9f5f5-5gg6s   worker2
testdp-5668f9f5f5-csx2x   worker2
testdp-5668f9f5f5-ftdrt   worker1
testdp-5668f9f5f5-zhcv9   worker1
```
