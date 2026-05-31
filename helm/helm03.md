# Helm 

## Раздел spec deployment


В values.yaml переносим replicaCount в раздел application и добавим revisionHistoryLimit:

```yml
application:
  reloader: false
  replicaCount: 1
  revisionHistoryLimit: 3
```

В шаблоне deployment.yaml добавляем соответстующие шаблоны:

```yml
spec:
  replicas: {{ .Values.application.replicaCount }}
  revisionHistoryLimit: {{ .Values.application.revisionHistoryLimit }}
```

За ним изменим раздел `selector.matchLabels`. Тут просто подставим готовый именнованный шаблон, при помощди которого опредяем labels селектора подов.

Аналогичный шаблон, подставляем в разделе `template.metadata.labels`:

```yml
  selector:
    matchLabels:
      {{- include "openresty-art.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "openresty-art.selectorLabels" . | nindent 8 }}
```

## Аннотации пода:

Добавим аннотации к шаблону пода. Обычно такие аннотации нужны для работы систем метрик например Prometheus.

Воспользуемся шаблоном из deployment-orig.yaml:

```yaml
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Единственное что подправим с нашей стороны это путь, т.к. podAnnotations мы переместили в раздел application:

```yaml
      {{- with .Values.application.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Конструкция [with](https://helm.sh/ru/docs/chart_template_guide/control_structures/#modifying-scope-using-with) позволяет управлять областью видимости переменных. 

Попробуем сгенерировать наше манифесты приложения с помощю `helm template` с пустой аннотацией, которая сейчас у нас применена по умолчанию(Это связано с тем, что сейчас в файле values.yaml application.podAnnotations пустая)

```sh
helm template app ./openresty-art/  > app.yaml
```

И в готовом манифесте шаблон пода будет следующим: 

```yaml
  template:
    metadata:
      labels:
        app.kubernetes.io/name: openresty-art
        app.kubernetes.io/instance: app
```

А теперь попробуем использовать файл my-values.yaml в котором в переменной `application.podAnnotations` есть данные. 

Файл my-values.yaml:

```yaml
fullnameOverride: "art"

application:
  reloader: true
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "80"
    prometheus.io/path: "/metrics"
```

В нашем примере добавил в в переменную `application.podAnnotations` аннотацию пода для работы Prometheus. Это теоретический пример т.к. для openresty сейчас нет никакого смысла применять аннотации. 

Проверим какой манифест соберется при использовании файла с нашими переменными. Не забудьте перейти в директорию где находится наш чарт:

```sh
cd helm/chart
helm template app ./openresty-art/ -f my-values.yaml > app.yaml
```

И результат который мы получаем:

```yaml
  template:
    metadata:
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "80"
        prometheus.io/scrape: "true"
      labels:
        app.kubernetes.io/name: openresty-art
        app.kubernetes.io/instance: app
```

## Оператор with 

Поговорим немного об операторе with в helm. Как выше было указано оператор with переключает контекст. Корнем контекста у нас является точка `.` - это текущий контекст или корень, через который мы обращаемся к данным. Когда мы находимся в началае шаблона, точка содержит в себе вообще всё `.Values`,`.Chart`,`.Release` и др.  

Поговорим на примере по конструкции - `{{- with .Values.application.podAnnotations }}`. Что делает эта конструкция? Исходя из того, что я описал выше, with переводит контекст. Т.е. подобная конструкция говорит о том, что внутри коренем `.` становится `.Values.application.podAnnotations`

Далее в добавленном нами шаблоне идёт:

```yaml
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Где:

  - **annotations** - это жестко указанный нами ключевое слово annotations.  
  - **{{- toYaml . | nindent 8 }}** - содержимое, что хранится у нас в `.Values.application.podAnnotations` Т.е. наши значения:

```yml
    prometheus.io/scrape: "true"
    prometheus.io/port: "80"
    prometheus.io/path: "/metrics"
```

Хочу обратить ваше внимание, что podAnnotations не подставляется, подставляется только его содержимое. 

  - При этом сама конструкция {{- }} - удалить все символы перед подставляемым шаблоном.  
  - {{- toYaml . }} - преобразовать всё что указано в корне в YAML формат.  
  - {{- toYaml . | nintend 8 }} - для каждой строки сделать отсуп в 8 символов


## Раздел Spec пода

В разделе spec добавим блок для imagePullSecrets. Напоминаю, что данная конструкция необходима для добавления секрета в котором будет хранится информация о приватном репозитории, а именно данные аутентификации к приватному репозиторию. 

```yaml
    spec:
      {{- with .Values.application.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Данный шаблон подставит нам imagePullSecrets в том случае если в переменной будет определено значение(Секрет). Т.е. если в нашем vaules/my-values будет указано имя секрета, например MySecret, шаблон в спецификации пода добавит ключевое слово imagePullSecrets, а в качетсве содержимого будет указана информация, которая возмьмется из корня, который благодаря with теперь будет `.Values.application.imagePullSecrets`.

Пример. В файле `my-values.yaml` добавим: 

```yaml
fullnameOverride: "art"

application:
  reloader: true
  podAnnotations: {}
  imagePullSecrets:
  - name: MySecret
```

А теперь проверим, как будет выглядеть манифест: 

```sh
 helm template app ./openresty-art/ -f my-values.yaml > app.yam
```

Созданный манифест:

```yaml
    spec:
      imagePullSecrets:
        - name: MySecret
      containers:
```

## spec.containers

Переходим к описанию спецификаций нашего контейнера. В качестве имени в оригинальном шаблоне helm указывается конструкция:

```yaml
      containers:
        - name: {{ .Chart.Name }}
```

Мы же укажем:

```yaml
      containers:
      - name: {{ include "openresty-art.fullname" . }}
```

Для того, чтобы при необходимости мы могли развернуть в рамках одного namespace множество деплойментов переопределяя `openresty-art.fullname`.

Далее переходим к указанию image.

```yaml
        image: "{{ .Values.application.image.repository }}:{{ .Values.application.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.application.image.pullPolicy }}
```

Перед этим в файле values.yaml перенесем блок image в созданный нами блок application:

```yaml
application:
  reloader: false
  replicaCount: 1
  revisionHistoryLimit: 3
  podAnnotations: {}
    # prometheus.io/path: /metrics
    # prometheus.io/port: "80"
    # prometheus.io/scrape: "true"
  imagePullSecrets: []
  # - name: MySecret
  image:
    repository: openresty/openresty
  # This sets the pull policy for images.
    pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
    tag: ""
```

Как формируется наш image в манифесте. Мы берем значение из `.Values.application.image.repository` ставим двоеточие и берем значение tag из `.Values.application.image.tag`, либо берем значение default `.Chart.AppVersion`. Кстати мы можем самостоятельно указать tag, даже не в переменной, а непосредственно в шаблоне. Т.е. заменить `default .Chart.AppVersion` на `default 1.27.1.2-0-alpine`.

Переходим к добавлению probes. В файле values.yaml пробы представлены в виде:

```yaml
probes:
  livenessProbe:
    httpGet:
      path: /
      port: http
  readinessProbe:
    httpGet:
      path: /
      port: http
```

Перенесем их в наш блок application:

```yaml
nameOverride: ""
fullnameOverride: ""

application:
  reloader: false
  replicaCount: 1
  revisionHistoryLimit: 3
  podAnnotations: {}
    # prometheus.io/path: /metrics
    # prometheus.io/port: "80"
    # prometheus.io/scrape: "true"
  imagePullSecrets: []
  # - name: MySecret
  image:
    repository: openresty/openresty
  # This sets the pull policy for images.
    pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
    tag: ""
  probes:
    livenessProbe:
      httpGet:
        path: /
        port: http
    readinessProbe:
      httpGet:
        path: /
        port: http
```

В Deployment.yaml добавление проб будет иметь вид:

```yaml
      containers:
      - name: {{ include "openresty-art.fullname" . }}
        image: "{{ .Values.application.image.repository }}:{{ .Values.application.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.application.image.pullPolicy }}
        {{- with .Values.application.probes }}
          {{- toYaml . | nindent 8 }}
        {{- end }}
```

С помощью with переводим контекст на `.Values.application.probes`, т.е. содержимое `probes` становится корнем, а затем с помощью `{{- toYaml . | nindent 8 }}`, добавляем наши пробы.

Таким же образом поступаем и с ресурсами:

```yaml
      containers:
      - name: {{ include "openresty-art.fullname" . }}
        image: "{{ .Values.application.image.repository }}:{{ .Values.application.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.application.image.pullPolicy }}
        {{- with .Values.application.probes }}
          {{- toYaml . | nindent 8 }}
        {{- end }}
        {{- with .Values.application.resources }}
        resources:
          {{- toYaml . | nindent 10 }}
        {{- end }}
```

Единственное, что в ресурсах мы указываем ключевое слово resources, т.к. мы не можем описывать requests и limits без него.

