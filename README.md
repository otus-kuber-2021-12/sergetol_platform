# sergetol_platform
sergetol Platform repository

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
