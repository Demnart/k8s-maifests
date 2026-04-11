Как работает Deployment c точки зрения кластера
1. API Server и Etcd (Прием заказа)Когда вы выполняете kubectl apply -f deployment.yaml:API Server получает запрос, проверяет ваши права (RBAC) и валидирует структуру YAML.
Если всё в порядке, API Server записывает объект Deployment в etcd (базу данных кластера).
Важно: На этом этапе ничего не запускается. API Server просто подтвердил: «Я записал ваше пожелание».

2. Deployment Controller (Планировщик стратегии)Внутри kube-controller-manager постоянно работает Deployment Controller.
Он замечает (через Watch API), что в системе появился новый (или изменился старый) Deployment.
Его единственная задача — создать или обновить объект ReplicaSet.Он смотрит на spec.replicas и spec.selector в вашем Deployment 
и создает ReplicaSet, который будет отвечать за поддержку нужного количества копий.

3. ReplicaSet Controller (Менеджер копий)Теперь в игру вступает ReplicaSet Controller.
Он видит новый ReplicaSet и понимает: «Мне нужно запустить $N$ подов, а сейчас их 0».
Он отправляет запросы к API Server на создание объектов Pod.
Результат: В системе появляются поды со статусом Pending, у которых в поле nodeName пусто.

4. Scheduler (Выбор места)kube-scheduler постоянно мониторит API Server на наличие подов без назначенной ноды.
Он видит ваши новые поды.Смотрит на ресурсы (CPU/RAM), аффинити и другие ограничения.
Выбирает наиболее подходящую рабочую ноду (Worker Node).
Записывает имя выбранной ноды в объект пода через API Server.

5. Kubelet (Исполнитель на месте)На каждой ноде живет агент Kubelet.Kubelet на выбранной ноде видит, 
что ему «назначен» новый под.Он обращается к Container Runtime (например, containerd или Docker) 
через интерфейс CRI.Container Runtime скачивает образ и запускает контейнеры.Kubelet сообщает API Server, 
что под успешно запущен (Running).

Получить информацию о селекторах
ks get rs my-first-deployment-b9cbc74b7 --output=jsonpath={.spec.selector} | jq
Получить владельца
ks get rs my-first-deployment-b9cbc74b7 --output=jsonpath={.metadata.ownerReferences} | jq

Рестарт приложения

Связан с изменением полей манифеста, как пример изменения версии контейнера

 - Можно в ручную изменить image app: 
   kubectl set image deployment/my-first-deployment nginx:1.27.5 - не есть гуд!

Лучший вариат изменения которые вызовут рестарт деплоймента изменение манифеста

ApiObject:
- Pod
- Deployment
- Statefulset

Рестарт приложения - kubectl rollout restart ApiObject name

Стратегия рестарта

описываются в .spec.strategy.type:RollingUpdate - базовый тип стратегий  при котором создаются поды управляемые новой Replicaset постепенно
те сначала запускается под под управлением новой Replicaste и только после этого удаляются поды старой ReplicaSet
в .spec.strategy.type.RollingUpdate.maxSurge - максимальное количество подов, которое может быть указано сверх зарезервированного. Базовое значение 25%
в .strategy.rollingUpdate.maxUnavailable - максимальное количество подов, которые будут не доступны

Документация - https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/