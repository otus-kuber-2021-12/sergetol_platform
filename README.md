# sergetol_platform [![Run tests for OTUS homework](https://github.com/otus-kuber-2021-12/sergetol_platform/actions/workflows/run-tests.yml/badge.svg)](https://github.com/otus-kuber-2021-12/sergetol_platform/actions/workflows/run-tests.yml)
## [OTUS - Инфраструктурная платформа на основе Kubernetes](https://otus.ru/lessons/infrastrukturnaya-platforma-na-osnove-kubernetes/)
sergetol Platform repository


# HW02 (kubernetes-controllers)

- поднят k8s кластер с помощью `kind` согласно конфигурации kind-config.yaml
- задеплоен `frontend-replicaset.yaml` для приложения `frontend`, проведен эксперимент с изменением количества реплик
- собрано еще раз приложение `frontend` с новым тегом `1.0.1` (был `1.0.0`); после применения манифеста ReplicaSet с новым тегом образа поды остались работать на старом образе, потому как ReplicaSet следит лишь за их количеством, а оно совпадало с требуемым, но если удалить поды, то они уже поднимутся на новом образе, либо тут надо использовать Deployment
- собрано приложение `paymentservice` с тегами `v0.0.1` и `v0.0.2`; написан `paymentservice-replicaset.yaml`, проведен тот же эксперимент с изменением количества реплик и новым тегом образа
- написан `paymentservice-deployment.yaml`, проведены эсперименты с RollingUpdate
- (*) написаны `paymentservice-deployment-bg.yaml` и `paymentservice-deployment-reverse.yaml` для blue-green и reverse RollingUpdate деплоя соответственно
- написан `frontend-deployment.yaml` с описанием readinessProbe для приложения `frontend`
- (*) написан `node-exporter-daemonset.yaml`
- (**) чтобы поды `node-exporter` смогли запускаться и на master нодах, в `node-exporter-daemonset.yaml` добавлено соответствующее toleration условие

```
cd kubernetes-controllers

kind create cluster --config kind-config.yaml
# kubectl get nodes

kubectl apply -f paymentservice-deployment.yaml
kubectl apply -f frontend-deployment.yaml

# kubectl get deployments
# kubectl get rs
# kubectl get pods

# kubectl delete -f paymentservice-deployment.yaml
# kubectl delete -f frontend-deployment.yaml

kubectl apply -f node-exporter-daemonset.yaml

# kubectl port-forward <daemonset_any_pod_name> 9100:9100
# http://localhost:9100/metrics

kubectl delete -f node-exporter-daemonset.yaml

kind delete cluster
```


# HW01 (kubernetes-intro)

- использовалась рабочая машина на Windows 10, установлен Docker Desktop, который использует движок на WSL2; под WSL2 работает Ubuntu 20.04
- на этой Ubuntu установлен `kubectl`, настроено автодополнение для `zsh`
- на этой Ubuntu с помощью `minukube` запущен (container-based) k8s
- проверено, что после удаления подов из `kube-system` их работа восстанавливается:
  - например, `coredns` поднимается согласно своему манифесту deployment/replicaset
  - поды компонентов control-plane, например, `kube-apiserver`, поднимаются сервисом `kubelet` согласно их манифестам, находящимся в файловой системе master-ноды k8s
- написан `Dockerfile`, запускающий web-сервер на порту 8000 под пользователем 1001 (на основе nginx:1.20.2-alpine); собран образ и загружен на Docker Hub
- написан `web-pod.yaml`, запускающий контейнер из собранного образа; в манифест добавлены init контейнер и volume; через `kubectl port-forward` проброшен порт 8000 и проверена работоспособность web-сервера по адресу `http://localhost:8000`
- склонирован репозиторий [microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo), собран образ микросервиса `frontend`, образ загружен на Docker Hub
- приложение `frontend` запущено через `kubectl run`
- (*) для корректного запуска приложению `frontend` не хватало переменных окружения; создан манифест `frontend-pod-healthy.yaml` с добавленными необходимыми переменными окружения - приложение запустилось

```
minikube start

# minikube dashboard --port=33333 --url=false
# http://127.0.0.1:33333/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/

# kubectl get pods -n kube-system

cd kubernetes-intro
kubectl apply -f web-pod.yaml
# kubectl get pods -w
# kubectl get pod web -o yaml
# kubectl describe pod web

kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
# http://localhost:8000

# kubectl delete -f web-pod.yaml

kubectl apply -f frontend-pod-healthy.yaml
kubectl logs -f frontend

# kubectl delete -f frontend-pod-healthy.yaml

minikube delete
```
