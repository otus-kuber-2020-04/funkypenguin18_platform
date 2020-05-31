# funkypenguin18_platform

# HW 6

### k8s - -templating

В процессе выполнения ДЗ:

#### Helm+Ingress
- установлен Helm
```bash
Helm version  
version.BuildInfo{Version:"v3.2.1", GitCommit:"fe51cd1e31e6a202cba7dead9552a6d418ded79a", GitTreeState:"clean", GoVersion:"go1.13.10"}
```

- Создан кластер k8s
```bash
gcloud beta container --project "sunny-density-278813" clusters create "kuber" --zone "europe-west2-a"
NAME   LOCATION        MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION    NUM_NODES  STATUS
kuber  europe-west2-a  1.14.10-gke.36  34.105.163.194  n1-standard-1  1.14.10-gke.36  3          RUNNING
```
и генерируем локальный кубконфиг
gcloud container clusters get-credentials kuber --zone=europe-west2-a

- Узнаем IP nginx-ingress
```bash
kubectl get svc -n nginx-ingress                              
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-controller        LoadBalancer   10.31.253.39    34.105.141.198   80:30170/TCP,443:32081/TCP   149m
nginx-ingress-default-backend   ClusterIP      10.31.251.190   <none>           80/TCP                       149m
```
#### ChartMuseum

и создаём наймспейс chartmuseum + установим chart:

```bash
kubectl create ns chartmuseum
namespace/chartmuseum created

helm upgrade --install chartmuseum stable/chartmuseum --wait \
 --namespace=chartmuseum \
 --version=2.3.2 \
 -f kubernetes-templating/chartmuseum/values.yaml
```

- Проверяем сертификат:

```bash
curl --insecure -v https://chartmuseum.34.105.141.198.nip.io/  2>&1 | awk 'BEGIN { cert=0 } /^\* Server certificate:/ { cert=1 } /^\*/ { if (cert) print }'

* Server certificate:/ { cert=1 } /^\*/ { if (cert) print }'
* Server certificate:
* 	subject: CN=chartmuseum.34.105.141.198.nip.io
* 	start date: May 31 17:23:32 2020 GMT
* 	expire date: Aug 29 17:23:32 2020 GMT
* 	common name: chartmuseum.34.105.141.198.nip.io
* 	issuer: CN=Let's Encrypt Authority X3,O=Let's Encrypt,C=US
* Connection #0 to host chartmuseum.34.105.141.198.nip.io left intact
```

#### Harbor
Добавление репозитория harbor

helm repo add harbor https://helm.goharbor.io

Создадим неимспейс harbor и применим chart (в этом чарте выключим сервис notary)

kubectl create ns harbor
helm upgrade --install harbor harbor/harbor --wait \
--namespace=harbor \
--version=1.1.2 \
-f kubernetes-templating/harbor/values.yaml

- Проверяем сертификат:
```bash
curl --insecure -v https://harbor.34.105.141.198.xip.io 2>&1 | awk 'BEGIN { cert=0 } /^\* Server certificate:/ { cert=1 } /^\*/ { if (cert) print }'
* Server certificate:
* 	subject: CN=harbor.34.105.141.198.xip.io
* 	start date: May 31 18:23:11 2020 GMT
* 	expire date: Aug 29 18:23:11 2020 GMT
* 	common name: harbor.34.105.141.198.xip.io
* 	issuer: CN=Let's Encrypt Authority X3,O=Let's Encrypt,C=US
* Connection #0 to host harbor.34.105.141.198.xip.io left intact
```
Попробуем создать свой helmchart

```bash
helm repo add chartmuseum https://chartmuseum.34.105.141.198.nip.io/
"chartmuseum" has been added to your repositories
```

### Helmfile | Задание со ⭐
- Добавлен хелмфайл, все файлы в helmfile/template

#### Создадим свой helm chart:

helm create kubernetes-templating/hipster-shop
kubectl create ns hipster-shop

##### GPG
К сожалению в Centos 7 нет возможности выполнить команду gpg --full-generate-key ,
т.к. нет такой версии гпг, что поддерживает данный ключ.
также нет никакого вывода хэша по команде gpg -k , однако есть фингерпринт при генерации ключа, с пробелами
и если их убрать то получается что-то похожее на этот загадочный "хэш".

В результате с его помощью получилось зашифровать secrets.yaml и затем расшифровать его
```bash
helm secrets view secrets.yaml
visibleKey: hiddenValue
```
Передадим  в  helm  файл secrets.yaml  какvalues  файл  -  плагин helm-secrets  поймет,  что  его  надо расшифровать, а значение ключа visibleKey  подставить в соответствующий шаблон секрета.

Запустим установку:
```bash
helm secrets upgrade --install frontend kubernetes-templating/frontend --namespace hipster-shop -f kubernetes-templating/frontend/values.yaml -f kubernetes-templating/frontend/secrets.yaml
Release "frontend" does not exist. Installing it now.
NAME: frontend
LAST DEPLOYED: Mon Jun  1 01:09:36 2020
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 1
TEST SUITE: None
removed ‘kubernetes-templating/frontend/secrets.yaml.dec’
```
и добавим правило фаервола:

```bash
gcloud compute firewall-rules create frontend-svc-nodeport-rule --allow=tcp:$(kubectl -n hipster-shop get services frontend -o jsonpath="{.spec.ports[*].nodePort}")
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
```

#### Kubecfg

kubecfg show kubernetes-templating/kubecfg/services.jsonnet
kubecfg update kubernetes-templating/kubecfg/services.jsonnet --namespace hipster-shop

Kustomize:

Установим kustomize, для кастомизации выбран сервис recommendationservice.

Уберём его описание из all-hipster-shop.yaml и перенесём его в директорию kubernetes-templating/kustomize/recommendation

Проверим что YAML с манифестами генерируется валидным:

kustomize build kubernetes-templating/kustomize/recommendation/

Создадим дирректории kubernetes-templating/kustomize/overriddes/hipster-shop[-prod] и добавим туда файлы с кастомизацией, которые переписывают переменные (такие как labels) для наших манифестов в зависимости от среды и применим их:

```bash
kubectl apply -k kubernetes-templating/kustomize/overrides/hipster-shop/
kubectl apply -k kubernetes-templating/kustomize/overrides/hipster-shop-prod/

```

# HW 5

### k8s - volumes
В процессе выполнения ДЗ:
- Познакомились с StatefulSet
- Попробовали minio на вкус
- Попробовали посоздавать k8s secrets (задание с  * )


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
