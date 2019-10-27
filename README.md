# CodingWithKubernetes

How to code a web app composed of two web service :
- a frontend
- a backend

# Prerequisite

This tutorial assume that you have already installed Kubernetes. If not, follow this other tutorial: https://github.com/charroux/kubernetes

# The backend app

The backend is coded in Java (Spring boot), but it doesn't matter. The Web service simply return "World !" when it receives an HTTP Get on "/hello": https://github.com/charroux/CodingWithKubernetes/blob/master/BackEnd/src/main/java/com/example/BackEnd/MyWebService.java

# The frontend app

The frontend (also coded in Java Spring boot) waits for an HTTP get on "/". Then it requests the backend service:

```
restTemplate restTemplate = new RestTemplate();
String s = restTemplate.getForObject(backEndURL, String.class);
```

and concatenates and returns the result "World !" with "Hello": 

```
return "hello (from the front end)" + " " + s + " (from the back end)";
```

https://github.com/charroux/CodingWithKubernetes/blob/master/FrontEnd/src/main/java/com/example/FrontEnd/MyWebService.java

# Kubernestes configuration file

A single yaml file is enough to configure the cluster: https://github.com/charroux/CodingWithKubernetes/blob/master/front-back-app.yml

Note first of all how configurations are separated from each other with ---

This yaml file contains :

## The frontend deployment


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-end-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front-end
  template:
    metadata:
      labels:
        app: front-end
    spec:
      containers:
      - name: front-end-container
        image: efrei/front-end:1
        imagePullPolicy: Always
```

Note the last line "imagePullPolicy: Always". Hence, each time the deployement is done, the image is pull from Docker hub. Which is usefull at development stage. The other choices are 
- without imagePullPolicy and :latest as the image tag
- wihtout imagePullPolicy and specify the imahe tag

## The frontend service in front of the deployment

```
apiVersion: v1
kind: Service
metadata:
  name: front-end-service
spec:
  ports:
    - name: http
      targetPort: 8080
      port: 80
  selector:
app: front-end
```
where targetPort is the port used by the frontend web service.

## The backend deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: back-end-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: back-end
  template:
    metadata:
      labels:
        app: back-end
    spec:
      containers:
        - name: back-end-container
          image: efrei/back-end:1   
          imagePullPolicy: Always
```          

## The backend service in front of the deployment

```
apiVersion: v1
kind: Service
metadata:
  name: back-end-service
spec:
  ports:
    - name: http
      targetPort: 8080
      port: 80
  type: ClusterIP
  selector:
app: back-end
```
Notice the type set to ClusterIP. A Kubernetes Service is an abstraction which defines a logical set of Pods running somewhere in the cluster, that all provide the same functionality. When created, each Service is assigned a unique IP address (also called clusterIP). This address is tied to the lifespan of the Service, and will not change while the Service is alive.

## How the frontend can reache the backend ?

Apps sometimes store such laddress as constants in the code. This is a violation of twelve-factor (https://12factor.net/). This address is stored in a property files of the frontend: https://github.com/charroux/CodingWithKubernetes/blob/master/FrontEnd/src/main/resources/application.yml

The pattern to build such Kubernetes address is: <service_name>.<name_space>.svc.<cluster_name>:<ClusterIP port>
  
  As you can see is the property file:
  - service_name: back-end-service
  - name_space: default
  - cluster_name: cluster.local
  - ClusterIP port: 80/hello
  
## The configuration of the ingress controller 

Go back to the previous tutorial to discover why an Ingress controller is useful: https://github.com/charroux/kubernetes

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: front-end-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: front-end.localhost
      http:
        paths:
          - path: /
            backend:
              serviceName: front-end-service
              servicePort: http
```
It uses Traefik as the Ingress controller (See https://github.com/charroux/kubernetes) 

## Launching the deployment

```
kubectl apply -f front-back-app.yml
```

Check if all resources are available: 
```
kubectl get all
```
NAME                                        READY   STATUS    RESTARTS   AGE
pod/back-end-deployment-5dbf4f845c-hpszd    1/1     Running   0          3h17m
pod/front-end-deployment-666cbf7cb9-lz44d   1/1     Running   0          3h17m

NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes          ClusterIP   10.43.0.1      <none>        443/TCP   3d2h
service/front-end-service   ClusterIP   10.43.18.233   <none>        80/TCP    3h17m
service/back-end-service    ClusterIP   10.43.22.135   <none>        80/TCP    3h17m

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/back-end-deployment    1/1     1            1           3h17m
deployment.apps/front-end-deployment   1/1     1            1           3h17m

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/back-end-deployment-5dbf4f845c    1         1         1       3h17m
replicaset.apps/front-end-deployment-666cbf7cb9   1         1         1       3h17m


Retreive the Ingress controller state: 
```
kubectl get ingress
```
NAME                HOSTS                 ADDRESS     PORTS   AGE
front-end-ingress   front-end.localhost   10.0.2.15   80      3h51m

Test the apps in your web browser at this URL: http://front-end.localhost/

Where front-end.localhost has been set in the above Ingress configuration.

## Delete resources

Delete the deployments and associated services: 
```
kubectl delete deployment.apps/front-end-deployment service/front-end-service deployment.apps/back-end-deployment service/back-end-service
```
Delete the Ingress controller: 
```
kubectl delete ingress front-end-ingress --namespace default
```
