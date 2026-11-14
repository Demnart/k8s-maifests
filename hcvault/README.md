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

## Настройка приложения 

TBD