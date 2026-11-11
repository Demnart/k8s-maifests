# Gateway API

[Документация](https://kubernetes.io/docs/concepts/services-networking/gateway/).
[Документация на сайте проекта](https://gateway-api.sigs.k8s.io/).
[Реализация контроллеров](https://gateway-api.sigs.k8s.io/implementations/)

## Traefik

[Документация](https://docs.traefik.io/)

```sh
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

В [чарте]() отключаем:

- функционал ingress контроллера;
- генерацию gateway по умолчанию;

Включаем провайдер kubernetesGateway. Он по умолчанию отключен. И настраиваем формат логов.

```sh
kubectl apply -f manifests/01-application-treaefik.yaml
```