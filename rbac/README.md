# RBAC


В рамках текущего цикла поработаем с RBAC(Role Based Access Control) в Kubernetes.

Все действия в кластере по факту являются обращением к API kubernetes. Грубо говоря мы можем разделить обращения к API на две сущности - обращение от пользователя и обращение от программы. В кластере кубернетес невозможно создать как такового пользователя(учетную запись - логин/пароль)

## Аутентификация (Общее)

В кластере будет считаться, что если у пользователя есть сертификат, который прошел проверку центром сертификации (CA kubernetes) то он считается валидным и может обращаться к API Kubernetes. 

Если говорить о подах, то если доступ необходим, то поду предоставляется токен с помощюь ServiceAccount и с помощью выделенного токена происходит доступ к kube API. 

## Авторизация

Поговорим об авторизации - как было указано чаще всего для авторизации пользователя используется RBAC, те на базе ролей. 
Существует два типа ролей: 

- `Role` - обычная роль, обязательно привязывается к namespace. При помощи роли описывается к каким объектам API можно применить те или иные действия
- `ClusterRole` - Кластерная роль, действует на весь кластер. 

Далее мы должны привязать пользователя к одной из этих ролей. Есть `kind`: `RoleBinding` или `ClusterRoleBinding` и именно с помощью этого kind мы связываем пользователя с ролью. 


## Аутентификация приложения

Теперь поговорим о приложении. Аутентификация производится с помощью токена. Токен автоматически предоставляется любому поду. Поясню подробнее - при создании пода в него автоматически монитруется volume по пути `/var/run/secrets/kubernetes.io/serviceaccount/token`, в рамках указанной директории хранится файл CA кластера, а так же сам файл токена. Указанный токен имеет права default ServiceAccount. Default ServiceAccount объект API Kubernetes существующий в любом namespace, даже в абсолютно новом namespace добавляется этот дефолтный ServiceAccount. Мы можем проверить это. 
Сейчас в моем namespace `work` нет ни одного запущенного пода:

```sh
kubectl -n work get all
```
```sh
No resources found in work namespace.
```

Но при этом default `ServiceAccount` уже есть в этом `namespace`:
```sh
kubectl -n work get serviceaccount
```
```sh
NAME      AGE
default   7d22h
```

Если мы хотим дать приложению более расширенные права( отличающиеся от прав default ServiceAccount) в плане обращения к API kubernetes, нам необходимо создать отдельный объект  kubernetes API типа `kind: ServiceAccount` и потом при создании `kind: RoleBinding\ClusterRoleBindig` связать созданную роль( где указаны нужные нам права) с `ServiceAccount` который мы создали. **Важно** - чтобы нашо приложение работало с правами, которые мы предоставили, в `pod/deployment/statefulset` и тд мы должны добавить имя созданного нами `ServiceAccount` в манифест.

## Выдача пользователю прав на доступ к Kubernetes API в конкретном namespace (Ручной вариант не рекомендуется!!!)

Для выдачи прав доступа к kubernetes API нам необходимо сначала создать сертификат. Для этого сначала генерируем ключ:

```sh
openssl genrsa -out username.key 4096
```

Где username - имя нашего пользователя. 
Далее займемся генерацией конфигурационного файла из которого мы создадим csr-запрос, который будет подписан CA нашего кластера.

```sh
cat conf.cfg
```
```sh
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[ dn ]
CN = artiom
DN = rogov
DC = local

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```

Далее на базе файла конфигурации и приватного ключа, создаем csr запрос:

```sh
openssl req -config conf.cfg -new -key artiom.key -nodes -out artiom.csr
```

Далее нам необходимо предоставить наш csr запрос в base64 формате. Делается это следующим образом: 

```sh
cat artiom.csr | base64 | tr -d '\n'
```

Дело в том, что kubernetes когда речь идет не о текстовых данных, хранит данные в формате base64. Для облегчения жизни мы не будем просто копировать значение нашего csr переведенного в base64, а создадим переменную среды окружения:

```sh
export BASE64_CSR=$(cat artiom.csr | base64 | tr -d '\n')
```

Далее мы создаем файл подписи сертификата csr.yaml уже как объекта Kubernetes API: 

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: artiom-csr
spec:
  request: ${BASE64_CSR}
  usages:
  - digital signature
  - key encipherment
  - client auth
  signerName: kubernetes.io/kube-apiserver-client # Обязательное поле, в данном случае это встроенный signer который подписывает клиенсткие сертификаты CA кластера
```

Далее с помощью утилиты `envsubst` мы передаем значение переменной окружающей среды - `BASE64_CSR` в наш файл и отправляем запрос на выпуск сертификата в `Kubernetes API`:

```sh
cat artiom-csr.yaml | envsubst | kubectl apply -f -
```

После этого мы можем проверить наши csr запросы в кластере с помощью команды: 

```sh
kubectl get csr
```
```sh
NAME         AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
artiom-csr   15s   kubernetes.io/kube-apiserver-client   kubernetes-admin   <none>              Pending
```

И состояние у него Pending - ожидает решение. 

Далее подтверждаем выпуск сертификата командой: 

```sh
kubectl certificate approve artiom-csr
```
```sh
certificatesigningrequest.certificates.k8s.io/artiom-csr
```

После этого проверяем наш запрос:

```sh
kubectl get csr
```
```sh
NAME         AGE     SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
artiom-csr   5m49s   kubernetes.io/kube-apiserver-client   kubernetes-admin   <none>              Approved,Issued
```

Как видим в Condition сертификат заапрувлен и выпущен. 
Для получения сертификата нам необходимо обратиться к конкретному полю нашего подтвержденного сертрификата:

```sh
kubectl get csr artiom-csr -o jsonpath='{.status.certificate}' | base64 -d > artiom.crt
```

Можем подробнее проверить наш сертификат: 
```sh
openssl x509 -in artiom.crt -text | less
```

## Генерация конфигурации для получения доступа к кластеру(.kube/config)

Для создания конфига в первую очередь мы должны узнать IP:PORT нашего кластера к которому мы будем подключаться. 
для этого выполним команду: 

```sh
kubectl cluster-info
```
```sh
Kubernetes control plane is running at https://192.168.122.65:6443
CoreDNS is running at https://192.168.122.65:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
KubeDNSUpstream is running at https://192.168.122.65:6443/api/v1/namespaces/kube-system/services/kube-dns-upstream:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Теперь можно приступить к созданию конфигурационного файла.
- **Шаг 1. Добавление информации о кластере.**
 ```sh
kubectl config --kubeconfig=./config set-cluster k8s --server=https://192.168.122.65:6443 --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true
```

  Параметры:
  - `--kubeconfig=./config` - указывает файл конфига для kubectl. Если не указать этот параметр, то будет изменен дефолтный .kube/config и мы не сможем получить доступ к кластеру. 
  - `set-cluster k8s` - устанавливает имя класта в файле конфигуации. Имя может быть любым, в данном случае в конфиге имя кластера будет k8s. 
  - `--server=https://192.168.122.65:6443` - Адрес нашего мастера кластера кубернетес
  - `--certificate-authority=/etc/kubernetes/pki/ca.crt` - указание пути до CA файла нашего кластера
  - `--embed-certs=true` - параметр, без которого в данных сертификата будет путь, а не сам сертификат. Т.е. этот параметр обязателен если мы хотим передать конфиг другому пользователю.

  Проверим получившийся файл:
```sh
cat config
```
```sh
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJQlBNQlJaS0l1aE13RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TmpBNU1qRXhNalV6TURkYUZ3MHpOakE1TVRneE1qVTRNRGRhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUURqdEJHVDA5SHpjWEFBdHpDRWMwRGEvMUlFNCtDVzY1dGFlWGxuUUg0c2NJT2ZCazhOUWpjblFPS1gKM0Y0V2F3QUtMb1VEajFCQW04OUZaZ1ZCNkF6NU13MjBlN2o5NFNlU0pUR2UzZkJsNDBseVBreXlHcWl5cVNiSwpuMVFwMGU2WURVbzJ5dy9GUFhPYlcyV0xERlZNeUIrSm43SmNDOERBc3Vqa3Z3RndEV2lxS2t4WU92Y0tJV1pGCkRScWxhRXpMakpJbjdWS0VqZTFUaVJqb29TdXpONU43S2xmd3pDaEQ1NkhLUnFLeGNuNm1uMVZ4ZEVYMGx2NVUKd2tQU0F6bU4yVlhFbVQyZ1E3V1l4Ym1FbmxCZjRjSEhFMmkzeC9EN1NpMjViTFU2emtzcEVTL0NCUEpjcEVHMAo4cVlhcjB1QmY4S0Q4TU9QK1YwWnJ1eEc0OVp4QWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJSTXZRaW13L0NSYkQ5cFJySGVBRVNxcHNlcnB6QVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQW0xZk1UeDV2eQpQL0tyNE1pY050MDVubWl4M3dnRHJ4aTdjQzc3VFQ5UlJvOWpkMUcvS2N6dVU3QWQ2Y2VyZDNpVUcyZTNNbDhpCjM4TkUvUVpSZERkQ2JyTzAzSFJ2a2RDU0o2aXhrU2xXa0NieHdqdVAyNnNHY2F3T2UzOVlHRWl2RG1uSDlUQW8KRDBpV2NqRFVqNTJ3cmdEQ3NINlZkangxTVVxSDhpRXp5U3pqV3V5OVNyeGxpVHo2Y2p2V0pIUGRpZFdxK1dlVApWRmg3SnFyZGpIZXFCY1BTRjh2Q25NK3hhQ0tLMEdGNm9Sd21RdVR2YVp6RGFCSU5YLzg2djNBRUJYZm1Jc0R0CldVZ3VVNmVXVk5VMTUxY3VRKzQ3Nk9sazZuUDZFOGdBUDdHTUMvYy9zK0FFam5TcVVUalVQME10aWxNRXBlZlgKZHk4WlZRTXNUVEpICi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: 192.168.122.65
  name: k8s
contexts: null
current-context: ""
kind: Config
users: null
```

- **Шаг 2. Добавление информации о пользователе**

```sh
kubectl config --kubeconfig=./config set-credentials artiom --client-key=artiom.key --client-certificate artiom.crt --embed-certs=true
```

Где:
  - `--kubeconfig=./config` - генерируемый нами конфиг файл.
  - `set-credentials artiom` - создание пользователя artiom в конфигурационном файле. Это имя не передается в кластер kubernetes и может быть любым. 
  - `--client-key=artiom.key` - путь до приватного ключа.
  - `--client-certificate` - путь до сертификата.
  - `--embed-certs=true` - настройка, чтобы вместо пути были данные сертификата. 

- **Шаг3. Определение контекста**

Контекст связывает между собой в конфиге данные кластера и данные пользователя: 

```sh
kubectl config --kubeconfig=./config set-context default --cluster=k8s --user=artiom --namespace kubetest
```

- **Шаг 4. Установить контекст по умолчанию**

```sh
kubectl config --kubeconfig=./config use-context default
```

На этом этапе создание конфигурационного файла закончено. 


## Проверка созданного конфигурационного файла.

Для проверки созданного конифига выполним команду: 

```sh
kubectl --kubeconfig=./config -n kubetest get pods
```
```sh
Error from server (Forbidden): pods is forbidden: User "artiom" cannot list resource "pods" in API group "" in the namespace "kubetest"
```

Если получена подобная ошибка то все хорошо, это значит, что сертификат выпущен корректно, но у нашего пользователя просто нет прав для листинга подов в namespace kubetest. Права на доступ к ресурсам выдаются уже с помощью политик `RBAC`.

## Создание роли для пользователя

Для того, чтобы дать нашему пользователю права/ограничения для доступа к kubernetes API нам необходимо создать роль и привязать ее к нашему пользователю.

Посмотрим на роль и привязку в файле `user.yaml`:
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kubetest
  name: artiom-role-01
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: artiom-rolebinding
  namespace: kubetest
subjects:
  - kind: User
    name: artiom
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: artiom-role-01
  apiGroup: rbac.authorization.k8s.io
```

Где в `Role`: 

- `apiGroups` - группы API к которым разрешен доступ, это то значение которое мы указываем в `apiVersion`
- `resources` - это ресурсы, это значения которые мы указываем в `kind`
- `verbs` - это то, что нам разрешено делать, например получать список подов.

Где в `RoleBinding`:

- `subjects` - указываются данные для кого мы делаем связь. В нашем случае речь идет о пользователе `kind: User` с именем `artiom` которое было указано у нас в выпускаемом сертификате в поле CN
- `roleRef` - с какой ролью мы связываем `kind: Role` и указываем имя нашей роли - `name: artiom-role-01`


Выше приведенный пример роли с ее политиками разрешено все - пример. **Так делать не нужно!!!**

Лучшая практика - это политика все запрещено и только при необходимости выдавать конкретные права доступа к конкретным API,kind и только на конкретные действия.

Удалим созданную `Role` и `RoleBindig`:
```sh
kubectl delete -f user.yam
```
```sh
role.rbac.authorization.k8s.io "artiom-role-01" deleted
rolebinding.rbac.authorization.k8s.io "artiom-rolebinding" deleted
```

Теперь пойдем от изначальной ситуации, когда для нашего пользователя нет никаких прав:

```sh
Error from server (Forbidden): pods is forbidden: User "artiom" cannot list resource "pods" in API group "" in the namespace "kubetest"
```

### Pods

Пользователь, которому мы передеали конфиг обращается с указанной ошибкой к нам и мы уже настраиваем роль непосредственно для его запросов.
Изменим нашу роль в соответствии с получаемой ошибкой. Что есть в ошибке: 

1. API group "" - аналог apiVersion: v1
2. resource "pods" - аналог kind: Pod
3. cannot list - невозможен листинг -> vert list

Т.е. новая роль будет иметь такой вид: 
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kubetest
  name: artiom-role-01
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list"]
```

`RoleBinding` в этом случае не меняется поэтому я его не трогаю. 

Применяем обновленню  роль: 
```sh
ks -n kubetest apply -f user.yaml
```
```sh
role.rbac.authorization.k8s.io/artiom-role-01 created
rolebinding.rbac.authorization.k8s.io/artiom-rolebinding created
```

Теперь проверям: 

```sh
kubectl  -n kubetest get po
```
```sh
No resources found in kubetest namespace.
```

Отлично, теперь мы можем проверять поды под нашим пользователем artiom.

### Services

Теперь попробуем проверить сервисы: 

```sh
kubectl -n kubetest get svc
```
```sh
Error from server (Forbidden): services is forbidden: User "artiom" cannot list resource "services" in API group "" in the namespace "kubetest"
```

И тут снова получаем ошибку, тк прав для доступа к сервисам нашему пользователю не выдано. По  аналогии с подами добавляем в нашу роль нужные права:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kubetest
  name: artiom-role-01
rules:
  - apiGroups: [""]
    resources: ["pods","services"]
    verbs: ["list"]
```

Те к ранее созданным apiGroups в блоке `resources` мы добавляем через запятую после подов `"services"` и применям: 

```sh
kubectl -n kubetest apply -f user.yaml
```
```sh
role.rbac.authorization.k8s.io/artiom-role-01 configured
rolebinding.rbac.authorization.k8s.io/artiom-rolebinding unchanged
```

Теперь при проверке из конфига пользователя доступен листинг сервисов:
```sh
kubectl -n kubetest get svc
```
```sh
No resources found in kubetest namespace.
```

Права на листинг сервисов теперь тоже доступен. 

### Deployments

Далее при попытке проверить Deployments получаем ошибку:
```sh
kubectl -n kubetest get deployments
```
```sh
Error from server (Forbidden): deployments.apps is forbidden: User "artiom" cannot list resource "deployments" in API group "apps" in the namespace "kubetest"
```

Обновляем нашу роль. Хочу заметить, что для deployments уже используется другая API группа - `apps`, т.е. обновленная роль будет иметь следующий вид:
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kubetest
  name: artiom-role-01
rules:
  - apiGroups: [""]
    resources: ["pods","services"]
    verbs: ["list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["list"]
```

Применяем обновленную роль: 
```sh
kubectl -n kubetest apply -f user.yaml
```
```sh
role.rbac.authorization.k8s.io/artiom-role-01 configured
rolebinding.rbac.authorization.k8s.io/artiom-rolebinding unchanged
```

И теперь при проверке Deployments тоже больше нет ошибок: 

```sh
kubectl -n kubetest get deployments
```
```sh
No resources found in kubetest namespace.
```

И по факту в дальнейшем мы либо на базе получаемых ошибок выдаем дополнительные права, либо изначально знаем, какие именно права мы предоставим пользователю и создаем роль с нужными правами. 

## Роль после выдачи всех нужных прав

Итоговый пример роли, как она может выглядеть:
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kubetest
  name: artiom-role-01
rules:
  - apiGroups: [""]
    resources: ["pods","services", "replicationcontrollers"]
    verbs: ["list", "create", "get", "update", "delete"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["list", "get"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: ["apps"]
    resources: ["deployments","daemonsets","replicasets","statefulsets"]
    verbs: ["list", "create", "get", "update", "delete", "patch", "deploy"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["list", "create", "get", "update", "delete"]
  - apiGroups: ["batch"]
    resources: ["jobs","cronjobs"]
    verbs: ["list", "create", "get", "update", "delete"]
```

## Поговорим о verbs

`verbs` - по факту то, что разрешено длеать с ресурсом. На примере: 
- `get` - получить конкретный ресурс, например один под по имени
- `list` - получить коллекцию ресурсов, например все поды в namespace
- `watch` - для наблюдения за одним или коллекцией ресурсов. 
- `update` - применяется для update ресурса, этот же verb применяется когда мы что-то сделали в манифесте и при этом применяем его снова с помощю `apply -f` - по факту применяется `update`
- `create` - создание ресурса
- `delete` - удаление ресурса
- `patch` - для обновления ресурса(обычно используется как команда kubectl )
- `deploy` - для деплоя


## Поговорим о приложениях.

