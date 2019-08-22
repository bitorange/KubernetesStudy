# Deployment, Service and Ingress
## 1. create a deployment for nginx
deployment的文件内容 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
```
执行命令

```
kubectl apply -f deployment.yaml
```
执行成功之后，可以使用下面的命令查看结果

```
kubectl get deployment
kubectl get pods
```

## 2. create Service
service的文件内容
```
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376

``` 

## 3. enable nginx Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
再apply 下面的文件
```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

## 4. create Ingress
ingress的文件内容
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: 172.16.18.145.nip.io
    http:
      paths:
      - path:
        backend:
          serviceName: hostnames
          servicePort: 80
``` 
172.16.18.145 是宿主机的一个ip，即集群中一个node的ip


这样使用curl 172.16.18.145.nip.io访问时，就能够获取到相应的pod的hostname

## 5.Service原理
