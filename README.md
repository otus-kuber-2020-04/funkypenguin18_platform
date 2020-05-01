# funkypenguin18_platform

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
В пдерложенном Документацией манифесте видно, что необходимо добавить несколько ENV переменных. 
Копируем frontend-pod.yaml в frontend-pod-healthy.yaml, дополняем манифест переменными среды, удаляем старый pod и запускаем новый из манифеста frontend-pod-healthy.yaml.
После этого состояние под-а frontend   1/1     Running

