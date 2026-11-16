# Hashicorp Vault

`Vault` - Инструмент для хранения секретов от комании Hashicorp Vault.  Подробную информацию об этом продукте возможно получить на странице - [HC Vault](https://www.hashicorp.com/en/products/vault)

## Возможные пути реализации

1. Установить `vault-agent` как `sidecar` контейнер, который будет размещаться рядом с основным контейнером, будет получать секерыт из хранилища `Vault` и затем передавать их с помощью томов, как пример emptyDir

2. Воспользоваться `secret-store-csi-driver` у которого есть возможность работать с `vault`. 


## Установка Hashicorp Vault

Установку я делал с помощью [helm chart](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm), но со своими значениями - файл `values.yaml`.
Основные изменения связаны с режимом работы и выделением pv для storage:

- Указан namespace vault
- Добавлен режим работы standalone - т.е. полноценная установка 1 реплики `statefulset` vault c ключем seal. Такой режим выбран из-за того, что кластер маленький для HA но при этом хочется опробовать Vault максимально приближенный к проду
- Указан StorageClass для выделения места под data storage vault
- Включил Vault UI чтобы иметь доступ к графическому управлению.
- Вырезан полностью функционал Ingress, но при этом httproute не настроен тк будет настроен отдельно с добавлением `ListenerSet`.

Итак последовательность шагов по установке:

- Добавить в репозиторий vault: 
```sh
helm repo add hashicorp https://helm.releases.hashicorp.co
```
- Установить helm с использованием наших values:
```sh
helm install vault hashicorp/vault -f values.yaml
```

## Шаги после установки

После установки в кластере в namespace vault появятся следующие элементы:
```sh
kubectl -n vault get all
```
```sh
NAME                                        READY   STATUS    RESTARTS   AGE
pod/vault-0                                 1/1     Running   0          21m
pod/vault-agent-injector-55467999d6-xb6sh   1/1     Running   0          22m

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/vault                      ClusterIP   10.110.113.110   <none>        8200/TCP,8201/TCP   22m
service/vault-agent-injector-svc   ClusterIP   10.98.46.200     <none>        443/TCP             22m
service/vault-internal             ClusterIP   None             <none>        8200/TCP,8201/TCP   22m
service/vault-ui                   ClusterIP   10.109.42.81     <none>        8200/TCP            22m

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vault-agent-injector   1/1     1            1           22m

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/vault-agent-injector-55467999d6   1         1         1       22m

NAME                     READY   AGE
statefulset.apps/vault   1/1     22m
```

Сейчас у меня под `statefulset` pod/vault-0 в статусе Running, но при установке он автоматически не поднимется. Если проверить логи то в них обнаружиться следующее:

```sh
kubectl logs pod/vault-0
```
```sh
{"@level":"info","@message":"security barrier not initialized","@module":"core","@timestamp":"2026-11-14T14:25:36.948288Z"}
{"@level":"info","@message":"seal configuration missing, not initialized","@module":"core","@timestamp":"2026-11-14T14:25:36.948388Z"}
```

Эти сообщения возникают из-за того, что `vault` не был инициализирован. Инициализация `vault` делается единожды после установки. Во время инициалзиации вам будет предоставлен ключ для операции `Unseal`, а так же root-token. 

> **Важно!!!** И ключ и токен необходимо сохранить в надежном месте. В случае утери ключа или токена придется производить переустановку Vault. 

Для того, чтобы инициализировать `Vault` необходимо выполнить следующую команду:

```sh
kubectl -n vault exec -it vault-0 -- vault operator init
```

После ввода этой команды, мы получим 5 ключей для `Unseal` и `root токен`. Для процедуры `Unseal` необходимо будет ввести 3 ключа из предоставленных 5.

Так как кластер используется как собственная лабораторная, я чутка облегчил себе жизнь выполнив такую команду:

```sh
vault operator init -key-shares=1 -key-threshold=1
```

При использовании подобной команды будет выдан только один ключ и для `Unseal` так же понадобится только один ключ.

Итак ключ/и были получены и теперь необходимо произвести процедуру `Unseal` следующим образом:

```sh
kubectl -n vault exec -it vault-0 -- vault operator unseal
```

После выполнения команды будет запрошен ключ `Unseal`. После ввода ключа, сразу же выведется статус `Vault`
```sh
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         2.0.4
Build Date      2026-08-03T16:14:36Z
Storage Type    file
Cluster Name    vault-cluster-c3a0e711
Cluster ID      7efb723b-e99c-a3a8-13b8-22a9ee42f10c
HA Enabled      false
```

При необходимости получить статус vault можно с помощью команды:

```sh
kubectl -n vault exec vault-0 -- vault status
```

## Доступ к UI

Для доступа к `UI` были созданы два ресурса: `Listenerset` и `httproute`. Файлы манифестов `httproute.yaml` и `listenerset.yaml`. После их применения не забудьте добавить доменное имя в `/etc/hosts` с указнием ip адреса `gateway`.

Секреты, политики и другое можно настраивать через веб-интерфейс. Я добавлю скриншоты настройки через UI когда буду использовать CRD - External Secret. Сейчас же настройку политик и сам секрет я буду настраивать через CSI и непосредственно в самом контейнере Vault с помощью CLI команд.

## Использование vault в приложении

Для начала начала работы зайдем в контейнер vault с помощью команды:

```sh
kubectl -n vault exec -it vault-0 -- /bin/sh
```

После входа в контейнер выполним команду:
```sh
vault login
```

У нас запросят токен, это тот самый токен который мы получили при инициализации `vault`, подставим его и после применения получим такую картину:
```sh
Key                  Value
---                  -----
token                ***.*************Wky80Dlmpdlw
token_accessor       4mXonfTn3dkIkfXFVja751X6
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

Что прримечательно, полученный нами root-токен не имеет срока истечения(TTL).

Следующие команды позволят нам настроить аутентификации и версию хранения `secrets`. 

Для начала создадим `secret` с типом хранения `kv-v2`. Это можно сделать с помощью команды:

```sh
vault secrets enable -path=secret kv-v2
```
```sh
Success! Enabled the kv-v2 secrets engine at: secret/
```

Далее добавим возможность аутентификации с помощью `Service Account` kubernetes:
```sh
vault auth enable kubernetes
```
```sh
Success! Enabled kubernetes auth method at: kubernetes/
```

После добавления метода аутентификации, можем проверить как именно мы можем производить аутентификацию в Vault с помощью команды:
```sh
vault auth list
```
```sh
Path           Type          Accessor                    Description                Version
----           ----          --------                    -----------                -------
kubernetes/    kubernetes    auth_kubernetes_21e09bc9    n/a                        n/a
token/         token         auth_token_3f67a178         token based credentials    n/a
```
Далее мы настраиваем конфигурацию для аутентификации:

```sh
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    issuer="https://kubernetes.default.svc.cluster.local"
```

Так как vault у нас находится непосредственно в кубернетес мы указываем host следующим образом - `https://kubernetes.default.svc:443` если же vault находился бы вне кластера нам необходимо было бы указать корректный адрес по которому мы можем получить доступ к API kubernetes. 

Далее попробуем добавить `secret`:

```sh
vault kv put secret/application application="HelloPassword" user="vasiliy" password="MegaPassword"
```
```sh
===== Secret Path =====
secret/data/application

======= Metadata =======
Key                Value
---                -----
created_time       2026-11-14T16:11:33.807031629Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

Чуть углубимся в то, что делает выше указанная команда - `vault kv put secret/application`. Данная команда создает в нашем секретном хранилище типа `kv-v2 secret/` файл с именем application. В рамках данного файла создаются три записи `application,user,password` со значенями которые мы указали.

При необходимости мы можем получить данные из созданного `secret` с помощью команды:

```sh
vault kv get secret/application
```
```sh
===== Secret Path =====
secret/data/application

======= Metadata =======
Key                Value
---                -----
created_time       2026-11-14T16:11:33.807031629Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

======= Data =======
Key            Value
---            -----
application    HelloPassword
password       MegaPassword
user           vasiliy
```

Так как мы авторизованы в vault мы имеем полный доступ к секрету, который мы же и создали. 

Теперь перейдем к настройке политики доступа к этому секрету:
```sh
vault policy write internal-app - <<EOF
path "secret/data/application" {
  capabilities = ["read"]
}
EOF
```
```sh
Success! Uploaded policy: internal-app
```

Мы создаем политику с названием `internal-app`, при использовании которой мы получим возможность читать - `capabilities = ["read"]` ранее добавленные нами секреты.

При необходимости мы можем получить информацию о созданной политике с помощью команды:

```sh
vault policy read internal-app
```
```sh
path "secret/data/application" {
  capabilities = ["read"]
  }
```

Далее мы создадим роль для типа авторизации kubernetes, в которой будут указаны следующие данные:

- Имя сервис аккаунта используемое приложением
- Namespace из которого должен прийти запрос
- Политика, которая будет применена

```sh
vault write auth/kubernetes/role/application \
bound_service_account_names=application-sa \
bound_service_account_namespaces=work \
policies=internal-app \
ttl=120m
```

Те если в vault поступит запрос от приложения, которое использует `serviceaccount` с именем `application-sa`, `serviceaccount` расположен в namespace `work`, то ему будет присвоена политика `internal-app ` позволяющая читать наш созданный секрет.


## Secrets Store CSI Driver

Secrets Store CSI Driver необходим для безопасной доставки внешних секретов, паролей, ключей и сертификатов внутрь подов Kubernetes в виде подключаемых томов (volumes). По факту это набор CRD который мы устанавливаем в кластер и потом можем настраивать. 

Я устанавливаю `SSCSID` с помощью `helm чарта и argocd` - файл `argocd-app/csi.yaml`. В этом файле добавляем в проект `default` необходимые CRD указав в качестве репозитория url чарта Secrets Store CSI Driver.

Почему проект `default`. CRD из чарта устанавливаются в kube-system, кроме `CRD` в чарте так же поставлется `daemonSet` - `csi-secrets-store-secrets-store-csi-driver` который устанавливает поды на каждой ноде + так же устанавливаются `Role/RoleBinding` и `ClusterRole/ClusterRoleBinding`. 

Тк проект `default` в `WhiteList` имеет `kind: *`  и `group: *` это позволяет не заморачиваться с настройкой разрешений и установить все необходимое. 

Сам манифест имеет вид:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: storage-csi
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    path: ''
    repoURL: https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
    targetRevision: 1.6.2
    chart: secrets-store-csi-driver
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: ''
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      enabled: true

```

## Настройка приложения 

Само приложение, которое мы будем использовать `manifests/openresty-deployment.yaml` это `openresty` который выведет дефолтную страницу, которую мы ему передадим.

Сам манифест имеет вид:

```yaml
---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: application-sa
---
kind: Service
apiVersion: v1
metadata:
  name: openresty
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    k8s-app: openresty

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: openresty
  labels:
    k8s-app: openresty
spec:
  replicas: 2
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      k8s-app: openresty
  template:
    metadata:
      labels:
        k8s-app: openresty
    spec:
      #### Добавлен для работы vault
      serviceAccountName: application-sa
      initContainers:
        - name: prepare
          image: busybox:1.38.0
          imagePullPolicy: "IfNotPresent"
          command:
            - sh
            - -c
            - |
              sed -e "s/\$APPLICATION_FROM_VAULT/$(cat /mnt/secrets-store/application)/" \
                  -e "s/\$USER_FROM_VAULT/$(cat /mnt/secrets-store/user)/" \
                  -e "s/\$PASSWORD_FROM_VAULT/$(cat /mnt/secrets-store/password)/" \
                  /mnt-template/index.html > /mnt/index.html
              echo "Done"
          volumeMounts:
            - name: secrets-store-inline
              mountPath: "/mnt/secrets-store"
              readOnly: true
            - name: config
              mountPath: "/mnt"
            - name: config-template
              mountPath: "/mnt-template"
      containers:
        - name: openresty
          image: openresty/openresty:centos-rpm
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http
          readinessProbe:
            httpGet:
              path: /index.html
              port: http
            initialDelaySeconds: 5
            periodSeconds: 15
          livenessProbe:
            httpGet:
              path: /index.html
              port: http
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          resources:
            limits:
              cpu: "0.2"
              memory: "400Mi"
            requests:
              cpu: "0.1"
              memory: "200Mi"
          volumeMounts:
            - name: config
              mountPath: /usr/local/openresty/nginx/html/
      volumes:
        - name: config-template
          configMap:
            name: openresty
        - name: config
          emptyDir:
            medium: Memory
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "vault-application"
```

Что важно в манифесте: 

- Создается `ServiceAccount` с именем `application-sa`, таким же именем, которое мы настроили в нашей роли в vault в методе аутентификации `kubernetes`. 
- `ServiceAccount` применяется к нашим подам в разделе `spec`
- В `Deployement` есть три `volumes`, первый это шаблон index.html страницы созданной для openresty, второй `emptyDir` для передачи готового файла index.html в под `openresty`, `emptyDir` используем типа `Memory` те данные будут хранится в памяти и секреты не появятся в основном контейнере openresty нигде на дисковой подсистеме. Есть третий `volume` с именем `secrets-store-inline` это наш `csi` из которого мы получим секерты из `Vault`.


Шаблон нашего `index.html` представлен как `ConfigMap` файл `manifests/configmap.yaml`:

```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: openresty
data:
  index.html: |
    <html>
    <head>
        <title>Тестовая страница</title>
        <meta charset="UTF-8">
    </head>
    <body>
      <h1>Тестовая страница</h1>

      <p>Вот тут мы подставляем данные из hashicorp vault:</p>
      <p>Приложение: <b>$APPLICATION_FROM_VAULT</b> </p>
      <p>Пользователь: <b>$USER_FROM_VAULT</b> </p>
      <p>Пароль: <b>$PASSWORD_FROM_VAULT</b> </p>

    </body>
    </html>
```

И самый важный в данном случае, это файл `manifests/csi.yaml`:

```sh
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: vault-application
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.vault.svc:8200"
    roleName: "application"
    objects: |
      - objectName: "application"
        secretPath: "secret/data/application"
        secretKey: "application"
      - objectName: "user"
        secretPath: "secret/data/application"
        secretKey: "user"
      - objectName: "password"
        secretPath: "secret/data/application"
        secretKey: "password"
```

В этом файле мы создаем `kind: SecretProviderClass` это объект добавленного ранее нами `Secrets Store CSI Driver`, т.е. без установки этого CRD у нас бы ничего не сработало. 

Теперь подробнее по содержимому:

- **provider: vault** - тип провайдера, тк используется `Vault` указываем vault.
- **vaultAddress: "http://vault.vault.svc:8200"** - адрес сервиса `vault` который был добавлен при установки `HC Vault`. Те имя сервиса `vault.` его namespace `vault.` и тип `svc.` по факту если бы захотели FQDN то адрес бы имел вид `vault.vault.svc.cluster.local`
- **roleName: "application"** - роль которую мы добавили в метод аутентификации kubernetes
- **objects** - массив с данными, где:
  - **secretPath** - путь до секрета в `Vault`. Помним, что из-за особенностей типа хранилища kv-v2, всегда нужно указывать в пути `/data/`
  - **secretKey** - ключ который мы добавили в созданный секрет, по факту по ключу мы получаем значения
  - **objectName** - объект, который будет создан, грубо говоря в нашем случае будет создан файл с содержимым секрета указанного в `secretKey`

### Запуск приложения

Запустить приложение можно двумя способами: 

1. Просто выполнить команду:

```sh
kubectl -n work apply -f manifests/
```

В этом случае применятся все файлы распололженные в директории manifests

2. С помощью `argo-cd`. Для этого в директории `argo-app/application-app.yaml` было создано приложение argo-cd с таким содержимым:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: application
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'https://github.com/Demnart/k8s-maifests'
    path: hcvault/manifests
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: work
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      enabled: true
    syncOptions:
      - CreateNamespace=true
```

После чего примеяем приложение в кластере:

```sh
kubectl apply -f argo-app/application-app.yaml 
```

## External Secrets Operator

External Secrets Operator (ESO) — это Kubernetes-оператор, который синхронизирует секреты из внешних систем управления (хранилищ) в стандартные нативные объекты Secret внутри кластера Kubernetes.

Установку ESO производил с помощью helm по инструкции [ESO getting started](https://external-secrets.io/latest/introduction/getting-started/)

После установки `ESO` можем проверить ресурсы, которые были установлены с помощью команды: 

```sh
kubectl -n external-secrets get all
```
```sh
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/external-secrets-68c5b95f44-6v2h5                   1/1     Running   0          77m
pod/external-secrets-cert-controller-5c986cfd69-2gvmm   1/1     Running   0          77m
pod/external-secrets-webhook-74f5789fdc-24jps           1/1     Running   0          77m

NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/external-secrets-webhook   ClusterIP   10.96.210.241   <none>        443/TCP   77m

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/external-secrets                   1/1     1            1           77m
deployment.apps/external-secrets-cert-controller   1/1     1            1           77m
deployment.apps/external-secrets-webhook           1/1     1            1           77m

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/external-secrets-68c5b95f44                   1         1         1       77m
replicaset.apps/external-secrets-cert-controller-5c986cfd69   1         1         1       77m
replicaset.apps/external-secrets-webhook-74f5789fdc  
```

Помимо деплойментов, так же был установлен набор CRD который можно проверить таким образом: 

```sh
kubectl get crd | grep external
```
```sh
acraccesstokens.generators.external-secrets.io                                Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
beyondtrustworkloadcredentialsdynamicsecrets.generators.external-secrets.io   Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
cloudsmithaccesstokens.generators.external-secrets.io                         Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
clusterexternalsecrets.external-secrets.io                                    Cluster      v1(storage)                     2026-11-16T18:05:45Z
clustergenerators.generators.external-secrets.io                              Cluster      v1alpha1(storage)               2026-11-16T18:05:45Z
clusterpushsecrets.external-secrets.io                                        Cluster      v1alpha1(storage)               2026-11-16T18:05:45Z
clustersecretstores.external-secrets.io                                       Cluster      v1(storage)                     2026-11-16T18:05:45Z
ecrauthorizationtokens.generators.external-secrets.io                         Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
externalsecrets.external-secrets.io                                           Namespaced   v1(storage)                     2026-11-16T18:05:45Z
fakes.generators.external-secrets.io                                          Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
gcraccesstokens.generators.external-secrets.io                                Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
generatorstates.generators.external-secrets.io                                Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
githubaccesstokens.generators.external-secrets.io                             Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
gitlabdeploytokens.generators.external-secrets.io                             Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
grafanas.generators.external-secrets.io                                       Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
mfas.generators.external-secrets.io                                           Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
passwords.generators.external-secrets.io                                      Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
pushsecrets.external-secrets.io                                               Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
quayaccesstokens.generators.external-secrets.io                               Namespaced   v1alpha1(storage)               2026-11-16T18:05:45ZG
secretstores.external-secrets.io                                              Namespaced   v1(storage)                     2026-11-16T18:05:45Z
sshkeys.generators.external-secrets.io                                        Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
stssessiontokens.generators.external-secrets.io                               Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
uuids.generators.external-secrets.io                                          Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
vaultdynamicsecrets.generators.external-secrets.io                            Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
webhooks.generators.external-secrets.io                                       Namespaced   v1alpha1(storage)               2026-11-16T18:05:45Z
```

После установки `ESO` так же как и для `CSI` необходимо в `vault` создать политики и роль для типа авторизации kubernetes.

### Vault policy and role for ESO

для начала провалимся в pod нашего `sts vault-0`:

```sh
kubectl -n vault exec -it vault-0 -- /bin/sh
```

После авторизиемся с помощью нашего токена: 

```sh
vault login
```
```sh
Token (will be hidden): 
```

Так же как и для `CSI` сначала создадим политику с именем `eso-read`: 
```sh
vault policy write eso-read - <<'EOF'
path "secret/data/*" {
  capabilities = ["read"]
}
path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
EOF
```

По факту данная политика дает нам доступ ко всем данным(секретам) расположенным в созданном хранилище secret.

Теперь настроим новую роль в методе авторизации kubernetes:

```sh
kubectl -n vault exec vault-0 -- vault write auth/kubernetes/role/eso \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-read \
  audience=vault \
  ttl=1h
```

> ***Важно!!!** в рамках текущей установки используется ServiceAccount созданный при установке ESO с помощью helm. Это сделанно намеренно, т.к. кластер локальный и в качестве storage я буду использовать ClusterSecretStore, т.е. ресурс доступный для всего кластера. Если будет использоваться ресурс SecretStore то он namespace зависимый и рекомендуется создавать отдельный ServiceAccount для него и указывать конкретный namespace откуда мы ожидаем его получить.

Итак политика и роль для авторизации `kubernetes` готовы, теперь можно приступить к непосредственному написанию манифестов из CRD ESO.

### CluseterSecretStore and ExternalSecret

Для начала нам нужно создать так называемый `backend` т.е. указать оператору `ESO` где именно он должен брать секреты. 

Описание `ClusterSecretStore` находится в файле `external-secret/clusterSecretStore.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso"
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
            audiences: ["vault"]
```

Итак что нас интересует в данном манифесте:

- `kind: ClusterSecretStore` - тип нашего ресурса ClusterSecretStore - распотраняется на весь кластер, т.е. любой созданный в любом namespace объект типа `ExternalSecret` сможет указать его как своего родителя.
- `spec.provider.vault` - указываем, что мы используем Vault как хранилище к которому необходимо подключиться. 
- `server: "http://vault.vault.svc:8200"` - адрес нашего Vault, т.к. Vault расположен непосредственно в кластере, то указываем FQDN его сервиса для подключения
- `path: "secret"` - путь до нашего хранилища секретов. 
- `version: "v2"` - версия хранилища, вспоминаем, что мы создавали хранилище типа `kv-v2`
- `spec.provider.vault.auth` - указываем:
  - `kubernetes` - тип авторизации настроенной в vault
  - `mountPath: "kubernetes"` - точка монтирования, совпадает с типом.
  - `role: "eso"` - роль добавленную для авторизации `kubernetes` с привязанной политикой
  - `serviceAccountRef` - указываем имя ServiceAccount `name: external-secrets` и namespace `namespace: external-secrets`. `audiences: ["vault"]` добавлен, для проверки jwt-токенов, он так же был добавлен нами при создании роли.

Добавляем `ClusterSecretStore` в кластер:
```sh
kubectl apply -f external-secret/clusterSecretStore.yaml
```
```sh
clustersecretstore.external-secrets.io/vault-backend created
```

Проверить, что ClusterSecretStore создан можно командой:

```sh
kubectl get clustersecretstore.external-secrets.io/vault-backend
```
```sh
NAME            AGE   STATUS   CAPABILITIES   READY
vault-backend   51m   Valid    ReadWrite      True
```

> **Важно!** в CAPABILITIES видно ReadWrite **НО** это возможности самого оператора ESO. Что в реальности будет возможно делать с секретами определяется политикой Vault. Созданная нами политика позволяет только читать данные, читать/получать метаданные. 

Итак, `ClusterSecretStore` настроен, а именно успешно подключился к нашему vault и получил по выданной роли доступ к хранилищу секретов secret. 

Но как нам получить нужные нам секреты? Для этого необходимо создать еще один объект но уже типа `ExternalSecret`.

Файл манифеста `external-secret/external-secret.yaml`:
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: vault-secret
  namespace: work
spec:
  refreshInterval: "180s"
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: vault-secret
  dataFrom:
    - extract:
        key: "kubernetes/test/env-secret"
```

В манифесте:

- `namespace: work` - ExternalSecret namespace зависимый ресурс, т.е. мы указываем в каком конкретном namespace необходимо его создать
- `spec.refreshInterval: "180s"` - раз в сколько секунд оператор будет обращаться в `vault` для проверки секретов.
- `spec.secretStoreRef` - в этом блоке мы указываем `родителя` который подключился к `Vault`
- `spec.target` - имя секрета, который создаться в API Kubernetes
- `spec.dataFrom.extract.key` - путь до нашего файла с секретами в Vault. Путь имеет вид `"kubernetes/test/env-secret"` потому, что в `ClusterSecretStore` мы уже ранее указали путь до хранилища `secret`. В ExternalSecret при таком варинте подключения, будут добавлены все ключи/значения хранящиеся в файле env-secret

Применим манифест:

```sh
kubectl apply -f external-secret/external-secret.yaml 
```
Проверить добавленный ExternalSecret можно так:
```sh
kubectl -n work get ExternalSecret
```
```sh
NAME           STORETYPE            STORE           REFRESH INTERVAL   STATUS         READY   LAST SYNC
vault-secret   ClusterSecretStore   vault-backend   180s               SecretSynced   True    55s
```

А теперь самое важное, как именно мы можем получить в наш под секреты? 
Для этого я немного переделал deployment openresty, который ранее использовался для `CSI` - файл манифеста - `manifests/openresty-es-deployment.yaml`

```yaml
kind: Service
apiVersion: v1
metadata:
  name: openresty
  namespace: work
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    k8s-app: openresty
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: openresty
  namespace: work
  labels:
    k8s-app: openresty
spec:
  replicas: 2
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      k8s-app: openresty
  template:
    metadata:
      labels:
        k8s-app: openresty
    spec:
      initContainers:
        - name: prepare
          image: busybox:1.38.0
          imagePullPolicy: "IfNotPresent"
          envFrom:
            - secretRef:
                name: vault-secret
          command:
            - sh
            - -c
            - |
              sed -e "s/\$APPLICATION_FROM_VAULT/$APP_NAME/" \
                  -e "s/\$USER_FROM_VAULT/$APP_USER/" \
                  -e "s/\$PASSWORD_FROM_VAULT/$APP_PASSWORD/" \
                  /mnt-template/index.html > /mnt/index.html
              echo "Done"
          volumeMounts:
            - name: config
              mountPath: "/mnt"
            - name: config-template
              mountPath: "/mnt-template"
      containers:
        - name: openresty
          image: openresty/openresty:1.27.1.2-13-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http
          readinessProbe:
            httpGet:
              path: /index.html
              port: http
            initialDelaySeconds: 5
            periodSeconds: 15
          livenessProbe:
            httpGet:
              path: /index.html
              port: http
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          resources:
            limits:
              cpu: "0.2"
              memory: "400Mi"
            requests:
              cpu: "0.1"
              memory: "200Mi"
          volumeMounts:
            - name: config
              mountPath: /usr/local/openresty/nginx/html/
      volumes:
        - name: config-template
          configMap:
            name: openresty
        - name: config
          emptyDir:
            medium: Memory
```

Итак основные отличия: 

1. ServiceAccount не нужен, мы указываем его непосредственно в ClusterSecretStore
2. Не нужно монтировать отдельный том для секретов, хотя если это необходимо, это возможно сделать, чтобы наши секреты в поде хранились в качестве файлов.
3. Подключение к поду, производится как и с обычными секретами:
```yaml
          envFrom:
            - secretRef:
                name: vault-secret
```

Т.е. мы берем все данные из секрета и они помещаются как env переменные в InitContainer. Так же как и в deployment `CSI` наши секреты не будут видны в основном контейнере, т.к. подключаются только в InitContainer.

Для проверки, применим ConfigMap и наш новый Deployment:
```sh
kubectl -n work apply -f manifests/configmap.yaml,manifests/openresty-es-deployment.yaml 
```

После чего можно проверить все созданные ресурсы: 

```sh
kubectl -n work get all
```
```sh
NAME                             READY   STATUS    RESTARTS   AGE
pod/openresty-7f9cf77565-glnhx   1/1     Running   0          19s
pod/openresty-7f9cf77565-gpnbz   1/1     Running   0          19s

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/openresty   ClusterIP   10.104.135.237   <none>        80/TCP    19s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/openresty   2/2     2            2           19s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/openresty-7f9cf77565   2         2         2       19s
```

А теперь зайдем на любую ноду кластера и выполним curl на IP адрес сервиса:
```sh
curl 10.104.135.237
```
```sh
<html>
<head>
    <title>Тестовая страница</title>
    <meta charset="UTF-8">
</head>
<body>
  <h1>Тестовая страница</h1>

  <p>Вот тут мы подставляем данные из hashicorp vault:</p>
  <p>Приложение: <b>App from vault</b> </p>
  <p>Пользователь: <b>Vasea</b> </p>
  <p>Пароль: <b>MEGAPASSWORD</b> </p>

</body>
</html>

```

Данные помещенные в `Vault` успешно были переданы к нам в под.

> **Важно!** - ESO обновляет Secret раз в refreshInterval (у тебя 180 секунд). Дальше всё зависит от способа подключения:
>  - переменные окружения (env/envFrom) не обновляются, пока под не перезапустится;
>  - volume обновляется сам примерно за минуту, но приложение должно перечитать файл.

Если нужен автоматический перезапуск при смене секрета, обычно ставят Reloader (stakater) и добавляют аннотацию на Deployment.