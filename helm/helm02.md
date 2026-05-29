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

