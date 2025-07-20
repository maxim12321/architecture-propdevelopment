# Управление трафиком внутри кластера

## Создаем сервисы

```bash
kubectl run front-end-app --image=nginx --labels role=front-end --expose --port 80
kubectl run back-end-api-app --image=nginx --labels role=back-end-api --expose --port 80
kubectl run admin-front-end-app --image=nginx --labels role=admin-front-end --expose --port 80
kubectl run admin-back-end-api-app --image=nginx --labels role=admin-back-end-api --expose --port 80
```

В результате создается 4 пода:
```
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
admin-back-end-api-app   1/1     Running   0          26s
admin-front-end-app      1/1     Running   0          33s
back-end-api-app         1/1     Running   0          42s
front-end-app            1/1     Running   0          53s
```

## Создаем сетевые политики

```bash
kubectl apply -f non-admin-api-allow.yaml
```

Результат:
```
$ kubectl apply -f non-admin-api-allow.yaml
networkpolicy.networking.k8s.io/allow-frontend-to-backend created
networkpolicy.networking.k8s.io/allow-backend-to-frontend created
networkpolicy.networking.k8s.io/allow-admin-frontend-to-admin-backend created
networkpolicy.networking.k8s.io/allow-admin-backend-to-admin-frontend created
```

Политики создаются с правильными селекторами:
```
$ kubectl get networkpolicy
NAME                                    POD-SELECTOR              AGE
allow-admin-backend-to-admin-frontend   role=admin-front-end      6m51s
allow-admin-frontend-to-admin-backend   role=admin-back-end-api   6m51s
allow-backend-to-frontend               role=front-end            6m51s
allow-frontend-to-backend               role=back-end-api         6m51s
```

## Проверка

Из нового пода все сервисы недоступны:
```
$ kubectl run test-$RANDOM --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://front-end-app
wget: download timed out
/ # wget -qO- --timeout=2 http://back-end-api-app
wget: download timed out
/ # wget -qO- --timeout=2 http://admin-front-end-app
wget: download timed out
/ # wget -qO- --timeout=2 http://admin-back-end-api-app
wget: download timed out
```

При этом если запустить с лейблом `front-end`, то `back-end-api-app` будет доступен, а остальные нет:
```
$ kubectl run test-$RANDOM --rm -i -t --image=alpine --labels role=front-end -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -qO- --timeout=2 http://front-end-app
wget: download timed out
/ # wget -qO- --timeout=2 http://back-end-api-app
<!DOCTYPE html>
...
/ # wget -qO- --timeout=2 http://admin-front-end-app
wget: download timed out
/ # wget -qO- --timeout=2 http://admin-back-end-api-app
wget: download timed out
```

А с лейблом `admin-back-end-api` доступен только `admin-front-end-app`:
```
$ kubectl run test-$RANDOM --rm -i -t --image=alpine --labels role=admin-back-end-api -- sh
If you don't see a command prompt, try pressing enter.
/ # wget -qO- --timeout=2 http://front-end-app
wget: download timed out
/ # wget -qO- --timeout=2 http://back-end-api-app
wget: download timed out
/ # wget -qO- --timeout=2 http://admin-front-end-app
<!DOCTYPE html>
...
/ # wget -qO- --timeout=2 http://admin-back-end-api-app
wget: download timed out
```
