# Домашнее задание к занятию «Helm»

### Цель задания

В тестовой среде Kubernetes необходимо установить и обновить приложения с помощью Helm.
 
------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение, например, MicroK8S.
2. Установленный локальный kubectl.
3. Установленный локальный Helm.
4. Редактор YAML-файлов с подключенным репозиторием GitHub.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция](https://helm.sh/docs/intro/install/) по установке Helm. [Helm completion](https://helm.sh/docs/helm/helm_completion/).

------

### Задание 1. Подготовить Helm-чарт для приложения

1. Необходимо упаковать приложение в чарт для деплоя в разные окружения. 
2. Каждый компонент приложения деплоится отдельным deployment’ом или statefulset’ом.
3. В переменных чарта измените образ приложения для изменения версии.

Helm установлен:

```
user@k8s:/opt/hw_k8s_10$ microk8s helm version
version.BuildInfo{Version:"v3.14.4+unreleased", GitCommit:"1b3a3b6738c9321dc29902be5bb5fbb53c87d22a", GitTreeState:"clean", GoVersion:"go1.21.13"}
```

Создаем chart:

```
user@k8s:/opt/hw_k8s_10$ sudo microk8s helm create netology-chart
Creating netology-chart
```

Вносим изменения в yaml:

```
user@k8s:/opt/hw_k8s_10$ cat netology-chart/Chart.yaml
apiVersion: v2
name: netology-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

Создаем deployment для nginx:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "netology-chart.fullname" . }}
  labels:
    app:  {{ include "netology-chart.fullname" . }}
spec:
  selector:
    matchLabels:
      app: {{ include "netology-chart.fullname" . }}
  replicas: {{ .Values.repl }}
  template:
    metadata:
      labels:
        app:  {{ include "netology-chart.fullname" . }}
    spec:
      containers:
      - name:  {{ include "netology-chart.fullname" . }}
        image:  {{ .Values.image }}:{{ .Values.tag }}
```

Создаем service:

```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "netology-chart.fullname" . }}
  namespace: {{ .Values.namespase }}
spec:
  selector:
    app: {{ include "netology-chart.fullname" . }}
  ports:
  - name: {{ include "netology-chart.fullname" . }}
    protocol: TCP
    port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
```

Прописываем переменные для chart:

```
user@k8s:/opt/hw_k8s_10$ cat netology-chart/values.yaml
image: nginx
tag: 1.21.0
repl: 1
service:
  port: 80
  targetPort: 80
user@k8s:/opt/hw_k8s_10$ cat netology-chart/values2.yaml
image: nginx
tag: 1.26.2
repl: 1
service:
  port: 80
  targetPort: 80
user@k8s:/opt/hw_k8s_10$ cat netology-chart/values3.yaml
image: nginx
tag: latest
repl: 1
service:
  port: 80
  targetPort: 80
```

------
### Задание 2. Запустить две версии в разных неймспейсах

1. Подготовив чарт, необходимо его проверить. Запуститe несколько копий приложения.

```
user@k8s:/opt/hw_k8s_10$ sudo microk8s helm template netology-chart
---
# Source: netology-chart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: release-name-netology-chart
  namespace:
spec:
  selector:
    app: release-name-netology-chart
  ports:
  - name: release-name-netology-chart
    protocol: TCP
    port: 80
    targetPort: 80
---
# Source: netology-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: release-name-netology-chart
  labels:
    app:  release-name-netology-chart
spec:
  selector:
    matchLabels:
      app: release-name-netology-chart
  replicas: 1
  template:
    metadata:
      labels:
        app:  release-name-netology-chart
    spec:
      containers:
      - name:  release-name-netology-chart
        image:  nginx:1.21.0
---
# Source: netology-chart/templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "release-name-netology-chart-test-connection"
  labels:
    helm.sh/chart: netology-chart-0.1.0
    app.kubernetes.io/name: netology-chart
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['release-name-netology-chart:80']
  restartPolicy: Never
```

2. Одну версию в namespace=app1, вторую версию в том же неймспейсе, третью версию в namespace=app2.

```
user@k8s:/opt/hw_k8s_10$ sudo microk8s helm install netology-chart netology-chart/ --namespace app1
NAME: netology-chart
LAST DEPLOYED: Tue Dec 24 09:05:01 2024
NAMESPACE: app1
STATUS: deployed
REVISION: 1
user@k8s:/opt/hw_k8s_10$ sudo microk8s helm install netology-chart2 ./netology-chart/ -f netology-chart/values2.yaml  --namespace app1
NAME: netology-chart2
LAST DEPLOYED: Tue Dec 24 09:09:21 2024
NAMESPACE: app1
STATUS: deployed
REVISION: 1
user@k8s:/opt/hw_k8s_10$ sudo microk8s helm install netology-chart3 ./netology-chart/ -f netology-chart/values3.yaml  --namespace app2
NAME: netology-chart3
LAST DEPLOYED: Tue Dec 24 09:09:36 2024
NAMESPACE: app2
STATUS: deployed
REVISION: 1
```

3. Продемонстрируйте результат.

```
user@k8s:/opt/hw_k8s_10$ microk8s helm list --namespace app1
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
netology-chart  app1            1               2024-12-24 09:05:01.890488071 +0000 UTC deployed        netology-chart-0.1.0    1.16.0
netology-chart2 app1            1               2024-12-24 09:09:21.586996573 +0000 UTC deployed        netology-chart-0.1.0    1.16.0
user@k8s:/opt/hw_k8s_10$ microk8s helm list --namespace app2
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
netology-chart3 app2            1               2024-12-24 09:09:36.727813768 +0000 UTC deployed        netology-chart-0.1.0    1.16.0
```

```
user@k8s:/opt/hw_k8s_10$ microk8s kubectl get deployments --namespace app1
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
netology-chart    1/1     1            1           6m21s
netology-chart2   1/1     1            1           2m3s
user@k8s:/opt/hw_k8s_10$ microk8s kubectl get deployments --namespace app2
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
netology-chart3   1/1     1            1           109s
user@k8s:/opt/hw_k8s_10$ microk8s kubectl get pods --namespace app1
NAME                               READY   STATUS    RESTARTS   AGE
netology-chart-85fbbb45c7-plv8w    1/1     Running   0          7m10s
netology-chart2-5c8776ff74-m79vw   1/1     Running   0          2m53s
user@k8s:/opt/hw_k8s_10$ microk8s kubectl get pods --namespace app2
NAME                               READY   STATUS    RESTARTS   AGE
netology-chart3-7bd6cf6d44-sxlh2   1/1     Running   0          2m39s
```


### Правила приёма работы

1. Домашняя работа оформляется в своём Git репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, `helm`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
