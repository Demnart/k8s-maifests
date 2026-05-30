# Создание чарта и metadata

Создаем чарт прилоежния:

```sh
cd helm
helm create openresty-art
```

## Структура чарта

[Documentation](https://helm.sh/docs/topics/charts/)

Редактируем содержимое файла Chart.yaml:

```yml
apiVersion: v2
name: openresty-art
description: My Vision of OpenResty
type: application
version: 0.1.0
appVersion: "1.27.1.2-0-alpine"
kubeVersion: ">=1.34.0"
```

из директории templates удаляем не нужные файлы.

```sh
cd templates
rm -rf {tests,hpa.yaml,serviceaccount.yaml}
```

Переименовываем и переносим автоматически созданные файлы. Мы их потом удалим, но по ходу правки будем заимствовать из них некоторые шаблоны:

```sh
mkdir ../../old-templates
mv deployment.yaml  httproute.yaml ingress.yaml service.yaml ../../old-templates
```

## Файл _helpers.tpl

Первое знакомство с шаблонами мы можем получить с помощью файла _helpers.tpl. Крайне не рекомендую его удалять. 

Пробежимся немного по файлу, для понимания как строятся шаблоны.  

Всё что начинается с двух фигурных скобок `{{` и заканчивается двумя фигурными скобками это шаблон `}}`

В шаблонах после фигурных скобок может идти тире `-`. Тире означает использовать функцию trim() - т.е. удалить значения указанные до начала шаблона или после него, либо и до шаблона и после. Пример из _helpers.tpl:

```tpl
{{- define "openresty-art.name" -}} # Удалить любые данные как перед так и в конце шаблона
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }} # удалить любые данные перед началом
{{- end }}
```

Для добавления комментария используется С-подобная конструкция:

```tpl
{{/*
 Здесь комментарий
*/}}
```

Рассмотрим на ранее приведенном примере шаблон:

```tpl
{{- define "openresty-art.name" -}} # Удалить любые данные как перед так и в конце шаблона
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }} # удалить любые данные перед началом
{{- end }}
```

Здесь мы создаем именнованный шаблон с помощью функции define(). В саму переменную `openresty-art.name` будет добавлено значение указанное нами в `default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" `. end же выступает как окончание именнованного шаблона.

В дальнейшем в любых файлах нашего чарта мы можем оброатиться к такому шаблону при помощи include. Например, создаем именнованный шаблон: 

```tpl
{{- define "openresty-art.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Verion | replace "+" "_" | trunc 63 | trimSuffix "-"}}
{{- end}}
```

После вызываем именнованный шаблон с помощью конструкции `include`:

```txt
тут будет что-то вставлено: {{ include "openerest-art.chart" . }}
```

После оработки, в данном месте будет подставлено содержимое вложенного шаблона:

```txt
Тут будет что-то вставлено: {{- printf "%s-%s" .Chart.Name .Chart.Verion | replace "+" "_" | trunc 63 | trimSuffix "-"}}
```

Итого в файле определено:  

  - **openresty-art.name** - имя чарта.  
  - **openresty-art.fullname** - имя приложения.  
  - **openresty-art.chart** - имя chart-а с версией.  
  - **openresty-art.labels** - общий набор labels, которые можно подставлять в metadata манифестов.  
  - **openresty-art.selectorLabels** - набор labels, которые можно использовать в селекторах. Например в селекторах servce.  
  - **openresty-art.serviceAccountName** - имя ServiceAccount. При условии, что оно определено в файле values.  

## Редактируем файл deployment.yaml - metadata

На данном этапе будем формировать шаблон для Deployment. 

Рекомендую разделеить в редакторе окно на две части. В левую поместить файл deployment.yml, в правую deployment-orig.yaml

Начнём c раздела metadata:

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "openresty-art.fullname" . }}
```

В данном месте будет генерировать имя деплоймента. При помощи [include](https://helm.sh/docs/howto/charts_tips_and_tricks/) подставится следующий шаблон:

```tpl
{{- define "openresty-art.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}
```

Попробуем разобраться, что тут написано. 

Для начала следует понять, к каким встроенным объектам мы можем обращаться в шаблонах helm.

## Встроенные объекты

[Documentation](https://helm.sh/docs/chart_template_guide/builtin_objects/)

  - **Release** - Описывает сам релиз.  
  - **Values** - Значение из файла values.yaml(параметры по умолчанию).  
  - **Files** - Доступ к файлам в чарте(кроме файлов в шаблоне).  
  - **Capabilities** - Информация о кластере kubernetes.  
  - **Template** - Информация о текущем файле шаблона.  
  - **Chart** - Содержимое файла Chart.yaml. Любые данные из Chart.yaml будут доступны здесь. Например, выражение {{ .Chart.Name }}-{{ .Chart.Version }} выведет на экран mychart-0.1.0.  
  - **Subcharts** - Это обеспечивает доступ родительскому объекту к элементам дочерних диаграмм (.Values, .Charts, .Releases и т. д.). Например, с помощью выражения .Subcharts.mySubChart.myValue можно получить доступ к значению myValue в диаграмме mySubChart.


Разберём пример приведенный выше шаблон построчно:

```tpl
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
```

Здесь происходит обращение к встроенному объекту Values и переменной указанной в этом файле fullnameOverride. В нашем случае переменная `fullnameOverride` в файле values.yaml - `fullnameOverride: ""` - пустая, т.е. для конструкции if это будет ложью и мы перейдем в блок `else`.

Если же `fullnameOverride: "qwerty"` будет иметь значение как я привел ранее, то произойдет следующее: 

 - Значение переменной fullnameOverride будет передано функции trim. Как мы видим для функции trim указан аргумент 63. Это максимальное число символо для поля и если значение переменной  будет превышать 63 символа, происходит обрезание значения до 63 символов. Почему обрезание проихсодит после 63 символов? Это связано с тем, что в Kubernetes поле в разделе `metadata` не может превышать больше 63 символов.  
 - После отработки функции trunc 63, значение передается функции trimSuffix. В примере фунация используется с аргументом "-". Обрезает информацию после встреи суффикса. Представим что в пременной будет значение - `fullnameOverride: "qwerty-1.0.1"`. Вся информация указанная после суффикса будет отброшена и на выходе мы получим значение `qwerty`.  


Как было указано выше в нашем примере переменная `fullnameOverride` пустая, поэтому мы перейдем к блоку `else`

```tpl
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
```
Мы можем в описании шаблона использовать свои переменные. Для этого используем `$имя_переменной`. Конкретно в нашем шаблоне - `$name` - создается переменная name, которой мы присваиваем значение с помощью конструкции `:=` далее у нас идет функия default и значения .Chart.Name .Values.nameOverride. Эту функцию необходимо читать с право налево. Т.е. если переменная .Values.nameOverride не определена, использовать .Chart.Name. В нашем случае переменная .Values.nameOverride не определена, поэтому helm пойдет в файл chart.yaml и возьмет значение name из этого файла. В нашем случае это - `openresty-art`. 

Далее у нас идет следующая строка - `if contains $name .Release.Name` - где мы открываем блок if в котором имеется функция contains(булевая) которая проверяет входит ли значение переменной name в подстроку .Release.Name и если это так, то мы забываем о переменной name и дальнейшее значение будет браться и обрабатываться уже из переменной .Release.Name. и прогоняем его через функции tunc/trimSuffix. 

Но если имя отлично от Relase.Name то мы перейдем в блок else где будет происходить следующее. Будет распечано  значение переменных в формате ".Release.Name-name" (printf форматирует переданную строку по шаблону, в нашем случае %s-%s, где в первую переменную %s попадет значение .Release.Name, а потом через дефис будет передано имя нашей переменной name).

## Раздел labels

Перейдем к разделую labels. По умолчанию нам предлагают следующую конструкци - `{{- include "openresty-art.labels" . | nindent 4 }}` её мы и используем.

Проверям файл _helpers.tpl и видим, что у нас будут подставлены следующие значения: 

```tpl
{{/*
Common labels
*/}}
{{- define "openresty-art.labels" -}}
helm.sh/chart: {{ include "openresty-art.chart" . }}
{{ include "openresty-art.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "openresty-art.selectorLabels" -}}
app.kubernetes.io/name: {{ include "openresty-art.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

Т.е. labels: 
  - helm.sh/chart - в которе подставится значение **openresty-art.chart**, а потом мы вызовем `openresty-art.selectorLabels` в котором будут следующие labels:  
  - app.kubernetes.io/name - в которое подставится значение **openresty-art.name**.  
  - app.kubernetes.io/instance - в которое подставится значение **.Release.Name**. 

Дальше мы смотрим, если  `.Chart.AppVersion` - определен, вставляем следующий label - **app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}**.

И в конце у нас будет добавлен label - `app.kubernetes.io/managed-by: {{ .Release.Service }}` где в качестве значения .Release.Service будет helm.

Так же мы видим что при указании labels в селекторе после передачи шаблона мы используем функцию nindent и передаем ей аргумент 4. Эта функция производит перевод на следующую строку и делает столько отступов сколько указано в передаваемом аргументе.

## Annotations

У меня в кластере установлен Stackater Reloader. И для его настройки необходимо в Deployment/Statefulset/DaemonSet добавлять annotation. Однако мы не можем быть уверены в том, что Stakater Reloader будет установлен в кластере куда будет загружаться наше приложение. 

Чтобы сделать красиво, мы создадим в файле values.yaml новый блок и назовем его application и в нем добавим значение reloader:false. Т.е. по умолчанию при использовании чарта аннотации не будут добавляться. 

Далее идем в Deployment и добавляем такой блок после labels:

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "openresty-art.fullname" . }}
  labels:
    {{- include "openresty-art.labels" . | nindent 4 }}
  {{- if .Values.application.reloader }}
  annotations:
    reloader.stakater.com/match: "true"
  {{- end }}
```

Сразу хочу обозначать, что такое объявление является не совсем корректным, но для базового понимания как мы можем добавить блок annotations при указании переменной .Values.application.reloader в значении true это более чем достаточно.  

## Проверка работы шаблонов

Проверим, работают ли наши шаблоны. Для этого воспользуемся командой [template](https://helm.sh/docs/helm/helm_template/)

Перейдем в директорию в который находится наш chart

```sh
cd chart
helm template app ./
```

Эта команда заставляет helm преобразовать шаблоны и выдать на стандартный вывод итоговый набор манифестов. Дополнительный параметр --debug, заставляет программу выводить отладочную информацию, которая будет полезной в случае обнаружения ошибок в шаблонах. 

В результате сформируется файл с манифестами appl.yaml. Откройте его и посмотрите начало определения Deployment:

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-openresty-art
  labels:
    helm.sh/chart: openresty-art-0.1.0
    app.kubernetes.io/name: openresty-art
    app.kubernetes.io/instance: app
    app.kubernetes.io/version: "openresty/openresty:1.27.1.2-0-alpine"
    app.kubernetes.io/managed-by: Helm
```



## Предопределение параметров по умолчанию.  


Изменить параметры по умолчанию, опредёленне в чарте можно двумя способами:

  1. При помощи параметра --set  
  2. Создав и применив собственный файл yaml c переопределенными параметрами.  

```sh
helm template app ./openresty-art  --set "application.reloader=true" --debug > app.yaml
helm template app ./openresty-art  -f my-values.yaml --debug > app.yaml
```

## Работа с приложением

Мы можем установить приложение: 

```sh
helm install app ./openresty-art/ --namespace work -f my-values.yaml
```
```sh
NAME: app
LAST DEPLOYED: Sat May 30 20:10:12 2026
NAMESPACE: work
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace work -l "app.kubernetes.io/name=openresty-art,app.kubernetes.io/instance=app" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace work $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace work port-forward $POD_NAME 8080:$CONTAINER_PORT
```
```sh
helm list --namespace work
```
```sh
NAME    NAMESPACE       REVISION        UPDATED                                         STATUS          CHART                   APP VERSION      
app     work            1               2026-05-30 20:10:12.380164999 +0300 EEST        deployed        openresty-art-0.1.0     1.27.1.2-0-alpine
```

Удалим приложение: 

```sh
helm uninstall app --namespace work
```