# Argo CD Projects

[Argo CD Project](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/) - Проекты обеспечивают логическую группировку приложений. Проекты предоставляют следующие возможности:

  - ограничение того, куда могут быть развернуты приложения (целевые кластеры и пространства имен).  
  - ограничение того, что может быть развернуто (доверенные репозитории Git).  
  - ограничение того, какие типы объектов могут или не могут быть развернуты (например, RBAC, CRD, DaemonSets, NetworkPolicy и т. д.).  
  - определение ролей проекта для обеспечения RBAC приложения (привязанных к группам OIDC и/или токенам JWT).  

# Создание проекта

`Argo CD project` может быть создан как через веб-интерфейс Argo CD так и с помощью манифестов. 

Попробуем создать проект `system-utils` который будет отвечать за установку в кластере системных утилит. 

Файл `project-system.yml`:

```yml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: system-utils
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Cluster utilities, monitoring, logging and etc.
  sourceRepos:
    - 'https://github.com/Demnart/k8s-maifests'
  destinations:
    - namespace: '*'
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
    - group: '*'
      kind: 'namespace'
    - group: 'scheduling.k8s.io'
      kind: 'PriorityClass'
    - group: 'rbac.authorization.k8s.io'
      kind: '*'
    - group: 'storage.k8s.io'
      kind: '*'
    - group: 'apiregistration.k8s.io'
      kind: 'ApiService'
    - group: 'admissionregistration.k8s.io'
      kind: 'ValidatingWebhookConfiguration'
```

Для создания проекта используется `Kind: AppProject`. В этом манифесте мы так же впервые встречаемся с такой сущностью как `finalizers`. Эта сущность Kubernetes нужна для удаления ресурсов. Например если мы удалим вышеуказанный манифест именно этот finalizer(постовляемый argocd) произведет удаление проекта.

Далее в spec:

  - **description:** - Текстовое описание проекта.  
  - **sourceRepos** - источники данных(репозитории, helm-charts и т.д.).  
  - **destinations** - в каких именно кластерах будут размещаться приложения/проекты. В нашем случае мы разрешаем размещение в любых namespaces в кластере.  
  - **clusterResourceWhitelist/clusterResourceBlacklist** - список разрешённых/запрещённых ресурсов которые можно создать.  

Как говорилось выше можно создавать/редактировать Projects как в веб-интерфейсе, так и с помощью манифестов.

Тк