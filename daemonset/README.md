***DaemonSet***  

DaemonSet - сущность кластера k8s необходима для запуска/поддержки одного пода на каждой воркер ноде  
В отличии от Deploymenst/StatefulSet - обычно Service не указывается  
DaemonSet как и Deployment не гарантирует строгое именование подов в отличии от StatefulSet  

***Taints***  
По умолчанию на control node есть следующий taint - ```[map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]```  
Формат состоит из 3-х частей - ```key=[value]:Effect```
- key - ключ taint - наример node-role.kubernetes.io/control-plane
- value - значение taints, практически нигде не определяют
- Effect - действие:
  - NoSchedule - запрещает планирование на ноде, если поды были запущены до применения taint они продолжают работать
  - NoExecute - запрещает планирование на ноде, если поды были запущены до применения taint поды будут уделены - лучше не использовать!  
  - PreferNoSchedule - мягкая версия NoSchedule 

Узнать о taints:  
```ks get no -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints```  

**Создание taints**  

```kubectl taint nodes node-name test-taint=:NoExecute```

**Удаление taints**  

```kubectl taint nodes node-name test-taint=:NoExecute-```


***Tolerations***  
Механизм с помощью которого можно разрещить помещать поды на ноды с указанными Taints  
Размещать tolerations необходимо в спецификациях пода!  
Пример:  
```
spec:
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
``` 

***NodeSelector***

При необходимости запуска пода на конкретной ноды можно использовать NodeSelector использует метки  
Установка метки на ноду:  
```kubectl label nodes node-name label(key=value)```  
as example ``` kubectl label nodes worker1 special=ds-only```  

Проверить метки нод:  
```kubectl get nodes --show-labels```  

Удалить метку с ноды:  
```kubectl label nodes node-name key-```

Параметр nodeSelector используется в спецификацию nodeSelector


Документация о taints [Taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)  
Документация о daemonset - [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)  