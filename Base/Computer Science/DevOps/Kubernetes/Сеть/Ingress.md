# Ingress

[[Kubernetes|Карта тем]] · [[Service]] · [[Облачный кластер]]

## Ресурс и контроллер

**Ingress** — API-объект с правилами HTTP(S)-маршрутизации по host и path к backend Services. **Ingress controller** — компонент, который читает эти правила и реализует их в proxy/балансировщике. Создать ресурс Ingress без подходящего контроллера недостаточно.

![](https://ucarecdn.stepik.net/33b22086-0e28-4bb3-9aed-6d8e7e04cfcb/)

Несколько HTTP-приложений могут использовать общую точку входа, а не отдельный LoadBalancer для каждого. Это упрощает управление доменами, TLS и маршрутизацией. Экономия адресов и стоимости зависит от конкретной инфраструктуры.

![](https://ucarecdn.stepik.net/8a6675dd-131a-4a26-92f9-d3ca82701636/)

![](https://ucarecdn.stepik.net/5d7af360-198f-4211-b48e-1901715a9e85/)

```text
Клиент → внешний адрес контроллера → host/path rule → Service → ready Pod
```

Примеры реализаций: community ingress-nginx, NGINX Ingress Controller, HAProxy и Traefik. Первые два названия относятся к разным проектам. Правила и расширения контроллеров не полностью взаимозаменяемы.

> [!warning] Статус ingress-nginx
> Для community **Ingress NGINX** официально объявлено завершение сопровождения в марте 2026: без новых выпусков и исправлений безопасности. Приведённые ниже команды и изображения сохраняют описание существующей конфигурации, но не являются рекомендацией разворачивать этот контроллер в новом production-кластере. Это не означает удаления API Ingress или прекращения поддержки всех NGINX-контроллеров. [Официальное уведомление Kubernetes](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/).

## Общий backend и Ingress-манифест

Namespace `example-app`, Deployment и `example-service` описаны в [[Управление ресурсами]] и [[Service]]. Эти ресурсы одинаковы для локального и облачного варианта и не дублируются. Предпосылка следующего примера — существующий подходящий IngressClass `nginx`; наличие класса проверяется, а не предполагается по имени образа.

`ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: example-app
spec:
  ingressClassName: nginx
  rules:
    - host: hello-world.example
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-service
                port:
                  number: 80
```

`ingressClassName` выбирает класс контроллера. `host` должен совпасть с Host запроса; `Prefix /` направляет подходящие пути к Service. `backend.service.port` — порт Service `80`, не `targetPort: 8080` Pod. Backend Service находится в namespace Ingress.

```bash
kubectl get ingressclass
kubectl apply -f ingress.yaml
kubectl -n example-app get pods
kubectl -n example-app get svc
kubectl -n example-app get ingress
kubectl -n example-app describe ingress example-ingress
```

Если общие манифесты сохранены в отдельную папку `ingress/`, допустимо `kubectl apply -f ingress/`. В ней должны находиться namespace, Deployment, Service и Ingress, а не только один файл правил.

## Локальный вариант Minikube

В существующих стендах используется addon:

```bash
minikube addons enable ingress
kubectl -n ingress-nginx get pods
minikube ip
```

С учётом предупреждения о сопровождении необходимо проверить реализацию addon конкретной версии Minikube. У controller Pod ожидается Running/Ready; Jobs admission-create/admission-patch могут иметь Completed — это не ошибка контроллера.

Адрес из `minikube ip` сопоставляется с доменом в hosts. Пример строки **только для Node с таким фактическим IP**:

```text
192.168.49.2 hello-world.example
```

```bash
curl http://hello-world.example/ingress
```

При HTTP-контракте примера ожидается `Hi there, I love ingress!`. Docker driver, вложенная виртуализация и ОС хоста могут требовать дополнительного туннеля; наличие адреса в статусе Ingress не гарантирует маршрут к нему с компьютера. См. [[Локальный кластер]].

## Облачный вариант Yandex Cloud

Контроллер размещается отдельно от приложения, например в namespace `ingress-nginx`. Исторический вариант установки через Marketplace:

![](https://ucarecdn.stepik.net/7ff5b3a3-a54a-4b3e-bf69-fae55d439528/)

Поиск приложения Ingress в Marketplace:

![](https://ucarecdn.stepik.net/fc589fd8-18a3-43d0-996f-04fc18703af3/)

Выделенный namespace для ресурсов контроллера:

![](https://ucarecdn.stepik.net/f990e576-dd7c-4143-9e58-b81b3ca15b0d/)

Состояние установленного приложения:

![](https://ucarecdn.stepik.net/74a8cce9-1e0b-4aa3-829a-5204b86342c9/)

> [!note] Исторические экраны
> Названия продуктов, defaults и доступные версии Marketplace меняются. Перед использованием проверяются поддержка контроллера, настройки RBAC, Service и сети; «оставить все настройки по умолчанию» не является универсальной эксплуатационной рекомендацией. Установка приложения в интерфейсе не меняет правил общего Ingress-манифеста.

```bash
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx get svc
```

Если controller опубликован Service типа LoadBalancer, его `EXTERNAL-IP`/hostname используется для DNS или локального hosts. Пример с **документационным**, не реальным адресом:

```text
192.0.2.10 hello-world.example
```

Затем применяется тот же `ingress.yaml`, проверяются `get ingress` и запрос:

```bash
curl http://hello-world.example/cloud
```

Ожидаемый ответ приложения — `Hi there, I love cloud!`. Требуются работающий LoadBalancer, разрешённый трафик security groups и корректный маршрут клиента. VPN может менять DNS или маршруты и затруднять проверку.

## TLS и ограничения

В показанном манифесте **TLS не настроен**: нет `spec.tls`, сертификата и настройки HTTPS-клиента. Ingress API позволяет описать TLS, но выпуск сертификата и его ротация требуют отдельных механизмов, например cert-manager. Ingress также не является универсальным способом публикации произвольного TCP/UDP без расширений контроллера.

Для диагностики последовательно проверяются controller, IngressClass, события Ingress, Host/path, Service port и ready endpoints. [[Диагностика и отладка]] описывает разделение сетевых и прикладных проблем.
