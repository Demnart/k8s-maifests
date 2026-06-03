# Application aviability

В этой части мы произведем настройку нашего сервиса для доступа к приложению, а так же настройку http-route до приложения. 

## Service

Для начала в файле values.yaml перенесем параметры service чуть вышы. Так как сервис напрямую не влияет на работу приложения, я не добавлял его в блок application:

```yaml
service:
  # Service type: ClusterIP, NodePort.
  type: ClusterIP
  port: 80 
  name: ""
  nodePort: ""
  targetPort: 80
```

Далее перейдем к настройке шаблона для Service:


```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: {{ include "openresty-art.fullname" . }}-svc
  labels:
    {{- include "openresty-art.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
  {{- include "openresty-art.selectorLabels" . | nindent 4 }}    
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    {{- if .Values.service.name }}
    name: {{ .Values.service.name }}
    {{- end }}
    {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
    nodePort: {{ .Values.service.nodePort }}
    {{- end }}
```

Из шаблона нас интересует конструкция `{{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}` которая позволяет два значения и только если оба значения истина мы добавим nodePort:

  - .Values.service.type "NodePort" - тип Service должен быть `NodePort`.  
  - .Values.service.nodePort - NodePort указан. Если данной переменной нет, то и добавлять поле nodePort бессмысленно. Порт будет взят автоматически исходя из настроек кластера.  


## HTTProute

Теперь перейдем к настройке роутинга. Напоминаю, что в кластере используется CNI Cilium с заменой kube-proxy. Так же в кластере настроен Cilium Gateway(контроллер) и создан gatewayclass, на который ссылается gateway:


```sh
kubectl get gatewayclass
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       11d
```
```sh
kubectl -n gateway-infra get gateway
```
```sh
NAME             CLASS    ADDRESS          PROGRAMMED   AGE
cilium-gateway   cilium   192.168.122.10   True         11d
```

Теперь перейдем к настройке httproute. Для начала найдем в файле values.yaml переменные для httproute:

```sh
httpRoute:
  # HTTPRoute enabled.
  enabled: false
  # HTTPRoute annotations.
  annotations: {}
  # Which Gateways this Route is attached to.
  parentRefs:
  - name: cilium-gateway
    port: 80
    # namespace: default
  # Hostnames matching HTTP header.
  hostnames:
  - app.local
  # List of rules and filters applied.
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /headers
```

Первый параметр который важен для нас это httpRoute.enabled. По умолчанию этот параметр false и что-бы шаблон httproute заработал нам необходимо указать этот параметр в true. 

Перейдем к рассмотрению шаблона httproute.yaml:

```yaml
{{- if .Values.httpRoute.enabled -}}
{{- $fullName := include "openresty-art.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  namespace: {{ .Release.Namespace }}
  name: {{ $fullName }}-route
  labels:
    {{- include "openresty-art.labels" . | nindent 4 }}
  {{- with .Values.httpRoute.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  parentRefs:
    {{- with .Values.httpRoute.parentRefs }}
      {{- toYaml . | nindent 4 }}
    {{- end }}
  {{- with .Values.httpRoute.hostnames }}
  hostnames:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  rules:
    {{- range .Values.httpRoute.rules }}
    {{- with .matches }}
    - matches:
      {{- toYaml . | nindent 8 }}
    {{- end }}
    {{- with .filters }}
      filters:
      {{- toYaml . | nindent 8 }}
    {{- end }}
      backendRefs:
        - name: {{ $fullName }}-svc
          port: {{ $svcPort }}
          weight: 1
    {{- end }}
{{- end }}

```

Самая первая строка - `{{- if .Values.httpRoute.enabled -}}` условный оператор if который создаст манифест только в том случае если `.Values.httpRoute.enabled` будет true.

Дальше:

```yaml
{{- $fullName := include "openresty-art.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
```

Мы создаем две переменных $fullName и $svcPort. Для fullName у нас будет браться значение из именнованного шаблона описанного в _helpers.tpl , а вот значение порта мы уже будем брать из переменной которую объявляли ранее для Service - `.Values.service.port `. А дальше мы уже используем знакомые нам конструкции, которые мы использовали ранее с deployment.yaml. 


> Важно! Для корректной работы HTTProute в итоговом манифесте необходимо указать namespace в котором находится созданный gateway. Для проверки helm-приложения и использую собственный файл с переменными my-values.yaml и в нем в блоке httproute, переменная namespace расскоментирована и в ней указан конкретный namespace, в котором у меня находится gateway.