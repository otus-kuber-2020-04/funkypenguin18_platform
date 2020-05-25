# funkypenguin18_platform

# HW 4

### k8s - networks

####01
- В существующий манифест web-pod.yaml добавлены описания livenessProbe и readinessProbe
- На его его основании создан манифест для Deployment web-deployment.yaml

####02
- Добавим стратегию развёртывания в наш манифест с параметрами     maxUnavailable: 0    maxSurge: 100%
- Протестированы различные варианты значений maxUnavailable и maxSurge, получились различные стратегии обновления подов:

    deployment с "простоем" - мы сначала удаляем все `pods` и только потом создаём новые.

    ```yaml
    maxUnavailable: 100%
    maxSurge: 0
    ```

    Blue-Green deployment - имеем две версии приложения одновременно.

    ```yaml
    maxUnavailable: 0
    maxSurge: 100%
    ```

    Canary deployment - pods постепенно заменяются со старой версией на новые

    ```yaml
    maxUnavailable: 1
    maxSurge: 0
    ```

    Недопустимая комбиинация, т.к. не позволит выполнить `Deployment`

    ```yaml
    maxUnavailable: 0
    maxSurge: 0
    ```

    Random mode =) - мы позволяем удалять и создавать `pods` в случайном порядке.

    ```yaml
    maxUnavailable: 100%
    maxSurge: 100%
    ```

####03
- Добавлен манифест, создающий Service с типом ClusterIP.
- Настроим minikube для использования IPVS вместо iptables, согласно инструкции из домашнего задания.

####04
- Установлен MetalLB, все сущности созданы


```bash
kubectl --namespace metallb-system get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-57f648cb96-w9f44   1/1     Running   0          175m
pod/speaker-9wjmf                 1/1     Running   0          175m

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/speaker   1         1         1       1            1           beta.kubernetes.io/os=linux   175m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           175m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-57f648cb96   1         1         1       175m
```

- Создан манифест metallb-config.yaml для настройки MetalLB с помощью ConfigMap.

- Создан новый Service, используйющий MetalLB в качетве LoadBalancer.
  IP 172.17.255.1 назначен нашему сервису web-svc-lb

####05
Задание со * | DNS через MetalLB
Созданы манифесты ./coredns/coredns-svc-lb.yaml и  ./coredns/coredns-svc-udp-lb.yaml

####06
Создание Ingress
- Применен манифест https://raw.githubusercontent.com/kubernetes/ingressnginx/master/deploy/static/provider/baremetal/deploy.yaml
- Создан и применен манифест nginx-lb.yaml

####07
Создание правил Ingress
- созданы и применены манифесты web-svc-headless.yaml и web-ingress.yaml

####08
Задание со * | Ingress для Dashboard
- созданы манифесты crd.yaml  dashboard-ingress.yaml  sa.yaml, помещены в директорию ./dashboard

####09
Задание со * | Canary для Ingress
- созданы манифесты  01-namespace.yaml  02-web-deploy.yaml  03-web-svc.yaml  04-web-ingress-default.yaml  04-web-ingress.yaml, помещены в директорию ./canary


# HW 3

### k8s - security

####01
- созданы сервисные учетные записи bob и dave
- роли bob прикреплена роль админ в рамках кластера, dave без доступа к кластеру (без прикрепления ролей)

####02
- создан namespace prometheus
- в нём создана сервисная учетная запись carol
- всем сервисным учетным записям неймспейса prometheus прикреплена роль просмотра pod-ов всего кластера

####03
- создан dev
- созданы сервисные учетные записи jane и ken
- к сервисной учетной записи jane прикреплена роль admin в рамках пространства dev
- к сервисной учетной  записи ken прикреплена роль view в рамках пространства dev

# HW 2

### ReplicaSet

Создано описание rs для hipster-frontend.
Replicaset не умеет обновлять поды если изменился шаблон (template). Шаблон применяется для новых контейнеров.

Создан deployment для hipster-paymentservice.
Попробовали поскейлить и поудалять поды.

### Deployment со *

#### Тестирование разных вариантов maxSurge и maxUnavailable.
##### maxUnavailable = **0**; maxSurge = **100%**

Вначале создаются новые поды. По готовности нового пода удаляется старый под.

##### maxUnavailable = **0**; maxSurge = **0**

Невалидная спецификация. Оба этих поля не могут равняться нулю одновременно.

##### maxUnavailable = **100%**; maxSurge = **100%**

Одновременно создаются новые поды и удаляются старые. Данная стратегия может привести к простою сервиса.

#####  maxUnavailable = **100%**; maxSurge = **0**

Похоже на стратегию Recreate. Вначале старые поды удаляются, после удаления создаются новые поды.
В отличие от Recreate, который дожидается полного удаления старых подов, при данных значениях maxUnavailable  и maxSurge новые поды созадаются сразу после изменения статуса старых подов на terminating.

Созданы paymentservice-deployment-bg.yaml и paymentservice-deployment-reverse.yaml

### Probes

Добавлен frontend-deployment.yaml с пробами.

### DaemonSet *

Создан daemonset с node-exporter, который запускается на всех нодах кластера.

```
kubectl get ds -w  
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
node-exporter   6         6         6       6            6           kubernetes.io/os=linux   78s
```

Проверена доступность метрик после проброса порта

```
часть выхлопа curl localhost:9100/metrics:

go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
```

# HW 1

1. Установлены kubectl, docker, minikube на centos 7, проверено подключение к кластеру

2. Проверить почему восстанавливаются под-ы из kubesystem и поведение при kubectl delete pod --all -n kube-system

* apiserver и другие поды будут восстановлены, т.к. это static pod, который загружается из /etc/kubernetes/manifests и он управляется напрямую kubelet, отличия есть в:
* coredns будет восстановлен, т.к. управляется через Deployment, который создает соответствующий ReplicaSet,
* kube-proxy управляется DaemonSet controller, поэтому также будет восстановлен после удаления.

3. Подготовлен, собран и запушен на DockerHub `Dockerfile` удовлетворяющий следующим требованиям:
    * web-сервер на порту 8000 (можно использовать любой способ)
    * Отдающий содержимое директории /app внутри контейнера (например, если в директории /app лежит файл homework.html, то при запуске контейнера данный файл должен быть доступен по [URL](http://localhost:8000/homework.html)
    * Работающий с UID 1001

4. Описан под в web-pod.yaml, запущен, простестирован вывод странички после порт-форварда на порту 8000

5. Склонирован репозиторий Hipster Shop, собран `Docker image` приложения `frontend` и запушен в  `DockerHub`.

6. Задание со *:

    в логах приложения видно, что нехватает переменных среды:


    ```kubectl logs frontend
    panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set
В предложенном Документацией манифесте видно, что необходимо добавить несколько ENV переменных.
Копируем frontend-pod.yaml в frontend-pod-healthy.yaml, дополняем манифест переменными среды, удаляем старый pod и запускаем новый из манифеста frontend-pod-healthy.yaml.
После этого состояние под-а frontend   1/1     Running
