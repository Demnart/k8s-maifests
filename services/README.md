**Сервисы необходимы для получения единой точки входа к нашим подам**

При добавление сервиса с помощью API K8S он создает новый объект EndpointSlice владельцем которого является созданный сервис

EndpointSlice содержит в себе IP адреса наших подов, информацию на каких нодах расположены поды. Использует API -  discovery.k8s.io/v1

**Сервисы бывают разных типов**
- *ClusterIP* - сервис по умолчанию если мы не указываем его. Открывает доступ к поду по IP  адресу сервиса. Доступен только изнутри кластера
- *NodePort* - открывает выбранный порт на всех нодах, по которому можно получить доступ к поду. Основная проблема в том, что порт открывается на нодах, что не безопасно
- *LoadBalancer* - сервис для LA предоставленных со стороны облачных провайдеров 
- *ExternalName* - ссылка на подобие CNAME 


Ручной port forward для доступа к нашему поду без использования NodePort & Ingress || GatewayAPI
```kubectl -n work port-forward svc/my-first-nginx 8080:80```

Получить информацию о service/EndpointSlice: 
- ```kubectl get svc``` - получить информацию о всех сервисах
- ```kubectl describe svc my-first-nginx``` - получить описание конкретного сервиса
- ```kubectl get EndpointSlice ``` - получить информацию о всех EndpointSlice
- ```kubectl describe EndpointSlice my-first-nginx-s8xqx``` - получить описание конкретного EndpointSlice
- ```kubectl get EndpointSlice my-first-nginx-s8xqx -o yaml | less``` - манифест конкретного EndpointSlice

Документация -[Service](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/)  
Рекомендации по именованию labels [Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)  
