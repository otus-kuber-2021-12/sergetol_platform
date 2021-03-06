# sergetol_platform [![Run tests for OTUS homework](https://github.com/otus-kuber-2021-12/sergetol_platform/actions/workflows/run-tests.yml/badge.svg)](https://github.com/otus-kuber-2021-12/sergetol_platform/actions/workflows/run-tests.yml)
## [OTUS - Инфраструктурная платформа на основе Kubernetes](https://otus.ru/lessons/infrastrukturnaya-platforma-na-osnove-kubernetes/)
sergetol Platform repository


# HW05 (kubernetes-volumes)

- использовался k8s кластер, поднятый с помощью `minukube`
- задеплоен StatefulSet с `MinIO` (для него создался PVC и PV на этом PVC)
- добавлен headless-сервис для `MinIO`
- (*) модифицирован исходный StatefulSet: данные рутового пользователя берутся из соответствующего Secret

```
kind create cluster

cd kubernetes-volumes

kubectl apply -f minio-statefulset.yaml
kubectl apply -f minio-headless-service.yaml

kubectl wait --for=condition=Ready pod minio-0 --timeout=120s
# kubectl get all --selector=app=minio
# kubectl get pvc
# kubectl get pv
# kubectl describe secret minio-root
kubectl logs minio-0

# kubectl exec minio-0 -- env

# kubectl run -ti --rm test --image=alpine --restart=Never -- wget -SO- http://minio:9000
kubectl run -ti --rm test --image=alpine --restart=Never -- wget -SO- http://minio:9000/minio/health/live
# kubectl run -ti --rm test --image=alpine --restart=Never -- wget -SO- http://minio:9000/minio/health/ready
kubectl run -ti --rm test --image=alpine --restart=Never -- wget -SO- http://minio:9001

# kubectl delete -f minio-headless-service.yaml
# kubectl delete -f minio-statefulset.yaml
# kubectl delete pvc data-minio-0

kind delete cluster
```


# HW04 (kubernetes-networks)

- по условию задания использовался k8s кластер, поднятый с помощью `minukube`
- добавлены `livenessProbe` и `readinessProbe` в ранее созданный деплой `kubernetes-intro/web-pod.yml` пода приложения
- написан Deployment для этого приложения
- создан для приложения простой Service с типом `ClusterIP`, проверены различные варианты доступа по кластерному IP
- установлен и настроен в `layer2` режиме балансировщик `MetalLB` для обслуживания Service с типом `LoadBalancer`
- создан для нашего приложения Service с типом `LoadBalancer`, проверен доступ снаружи кластера
- (*) создан Service с типом `LoadBalancer` для доступа снаружи к `CoreDNS`
- установлен Ingress-контроллер из манифестов коробочного `ingress-nginx` и для него создан Service с типом `LoadBalancer`
- создан для нашего приложения headless-сервис (`clusterIP: None`) и добавлен для него Ingress через префикс `/web`
- (*) установлен `kubernetes-dashboard` из официальных манифестов и настроен для него Ingress через префикс `/dashboard`, проверен вход по токену
- (*) реализовано канареечное развертывание с помощью `ingress-nginx`: в отдельном Namespace `staging` для нашего приложения добавлены такие же Deployment, headless-сервис и Ingress через тот же префикс `/web`, только в Ingress-манифесте добавлены соответствующие annotations `canary-by-header` для включения роутинга на staging-приложение по http-заголовку `x-env: staging`

```
minikube start

cd kubernetes-networks

kubectl apply -f web-deploy.yaml
# kubectl get all
# kubectl describe deployment web

kubectl apply -f web-svc-cip.yaml
# kubectl get svc web-svc-cip
# kubectl describe svc web-svc-cip

minikube ssh "curl http://$(kubectl get svc web-svc-cip -o=jsonpath='{.spec.clusterIP}')/health"
minikube ssh "curl http://$(kubectl get svc web-svc-cip -o=jsonpath='{.spec.clusterIP}')/index.html"

# https://metallb.universe.tf/installation/#preparation
# kubectl -n kube-system get configmap kube-proxy -o yaml | sed -e 's/strictARP: false/strictARP: true/' | sed -e 's/mode: ""/mode: "ipvs"/' | kubectl -n kube-system diff -f -
kubectl -n kube-system get configmap kube-proxy -o yaml | sed -e 's/strictARP: false/strictARP: true/' | sed -e 's/mode: ""/mode: "ipvs"/' | kubectl -n kube-system apply -f -
kubectl -n kube-system exec $(kubectl get pods -n kube-system --selector='k8s-app=kube-proxy' -o=name) -- kube-proxy --cleanup
kubectl -n kube-system delete pod --selector='k8s-app=kube-proxy'
# kubectl -n kube-system get all

# https://metallb.universe.tf/installation/
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
# kubectl -n metallb-system get all

# https://metallb.universe.tf/configuration/#layer-2-configuration
kubectl apply -f metallb-config.yaml

kubectl apply -f web-svc-lb.yaml
# kubectl get svc web-svc-lb
# kubectl describe svc web-svc-lb
# kubectl -n metallb-system logs $(kubectl -n metallb-system get pod --selector='component=controller' -o=name)

minikube ssh "curl http://$(kubectl get svc web-svc-lb -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/health"
minikube ssh "curl http://$(kubectl get svc web-svc-lb -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/index.html"

# https://minikube.sigs.k8s.io/docs/handbook/accessing/#loadbalancer-access
minikube tunnel --alsologtostderr
curl http://localhost/health
curl http://localhost/index.html

# https://metallb.universe.tf/usage/#ip-address-sharing
kubectl apply -f coredns/kube-dns-lb.yaml
# kubectl -n kube-system get svc --selector='k8s-app=kube-dns-lb'

minikube ssh "nslookup web-svc-cip.default.svc.cluster.local 172.17.255.10"
# minikube ssh "nslookup web-svc-lb.default.svc.cluster.local 172.17.255.10"

# https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/baremetal/deploy.yaml
# kubectl -n ingress-nginx get all

kubectl apply -f nginx-lb.yaml
# kubectl -n ingress-nginx get svc ingress-nginx
# kubectl -n ingress-nginx describe svc ingress-nginx

minikube ssh "curl http://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')"
minikube ssh "curl -k https://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')"

kubectl apply -f web-svc-headless.yaml
# kubectl get svc web-svc
# kubectl describe svc web-svc

# https://kubernetes.github.io/ingress-nginx/examples/rewrite/
kubectl apply -f web-ingress.yaml
# kubectl get ingress web
# kubectl describe ingress web

minikube ssh "curl http://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/health"
minikube ssh "curl http://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/index.html"
minikube ssh "curl -k https://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/health"
minikube ssh "curl -k https://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/index.html"

# https://github.com/kubernetes/dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
# kubectl -n kubernetes-dashboard get all
# kubectl -n kubernetes-dashboard describe svc kubernetes-dashboard

minikube ssh "curl -k https://$(kubectl -n kubernetes-dashboard get svc kubernetes-dashboard -o=jsonpath='{.spec.clusterIP}')"

# https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
kubectl apply -f dashboard/dashboard-admin.yaml

# getting a Bearer Token:
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa dashboard-admin -o=jsonpath='{.secrets[0].name}') -o=go-template='{{.data.token | base64decode}}'

# https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#backend-protocol
# https://kubernetes.github.io/ingress-nginx/examples/rewrite/
# https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#configuration-snippet
kubectl apply -f dashboard/dashboard-ingress.yaml
# kubectl -n kubernetes-dashboard get ingress kubernetes-dashboard
# kubectl -n kubernetes-dashboard describe ingress kubernetes-dashboard

minikube ssh "curl -k https://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/dashboard/"

# https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary
kubectl apply -f canary/web-staging.yaml
# kubectl -n staging get all
# kubectl -n staging get ingress

minikube ssh "curl -s http://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/index.html | grep POD_NAMESPACE"
minikube ssh "curl -sk https://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/index.html | grep POD_NAMESPACE"

minikube ssh "curl -s --header 'x-env: staging' http://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/index.html | grep POD_NAMESPACE"
minikube ssh "curl -sk --header 'x-env: staging' https://$(kubectl -n ingress-nginx get svc ingress-nginx -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')/web/index.html | grep POD_NAMESPACE"

minikube delete
```


# HW03 (kubernetes-security)

- использовался k8s кластер, поднятый с помощью `minukube`
- написаны манифесты `01-sa-bob.yaml` для sa `bob` и `02-clusterrolebinding-bob-admin.yaml` для выдачи ему роли `admin` в рамках всего кластера
- написан `03-sa-dave.yaml` для sa `dave`
- написаны `01-ns-prometheus.yaml` для ns `prometheus` и `02-sa-carol.yaml` для sa `carol` в этом ns
- написаны `03-clusterrole-view-pods.yaml` для clusterrole, позволяющей делать `get`, `list`, `watch` в отношении Pods всего кластера, а также `04-clusterrolebinding-prometheus-sa-view-pods.yaml` для привязки этой clusterrole ко всем sa из ns `prometheus`
- написаны `01-ns-dev.yaml` для ns `dev`, `02-sa-jane.yaml` и `04-sa-ken.yaml` для sa `jane` и `ken`
- написаны `03-rolebinding-jane-admin-dev.yaml` и `05-rolebinding-ken-view-dev.yaml` для выдачи sa `jane` роли `admin`, а sa `ken` роли `view` в рамках ns `dev`

```
minikube start

cd kubernetes-security
kubectl apply -f task01/
kubectl apply -f task02/
kubectl apply -f task03/

# kubectl auth can-i get deployments --as system:serviceaccount:default:bob
# kubectl auth can-i list pods --as system:serviceaccount:prometheus:carol -n prometheus
# kubectl auth can-i create deployments --as system:serviceaccount:dev:ken -n dev

minikube delete
```


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
