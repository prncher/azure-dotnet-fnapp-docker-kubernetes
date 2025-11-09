# azure-dotnet-fnapp-docker-kubernetes
Azure function app in C# with Docker and Kubernetes

Dockerfile
=============
#### Stage 1: Build the application
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS installer-env

#### Copy the files and publish the app
COPY . /src/dotnet-function-app <br>
RUN cd /src/dotnet-function-app && \\ <br>
    mkdir -p /home/site/wwwroot && \\ <br>
    dotnet publish *.csproj --output /home/site/wwwroot

#### To enable ssh & remote debugging on app service change the base image to the one below
#### FROM mcr.microsoft.com/azure-functions/dotnet:4-dotnet8.0-appservice
FROM mcr.microsoft.com/azure-functions/dotnet:4-dotnet8.0 <br>
ENV AzureWebJobsScriptRoot=/home/site/wwwroot \\ <br>
&emsp;AzureFunctionsJobHost__Logging__Console__IsEnabled=true

#### Copy the published application from the build stage
COPY --from=installer-env ["/home/site/wwwroot", "/home/site/wwwroot"]

---
```bash
  docker build --tag lambdafnimage:v1.0.0 .  (Here lambdafnimage:v1.0.0 is the image name)
  docker run -p 8080:80 -it lambdafnimage:v1.0.0
  http://localhost:8080/api/MyHttpTrigger
```
## (Stop the Docker container and delete the image - Not needed for Kubernetes.)

Open a command prompt and follow the steps below.

### 1. minikube start --hyperv-use-external-switch --driver=docker --docker-env=local
### 2. minikube addons enable metrics-server
### 3. minikube -p minikube docker-env
### 4. @FOR /f "tokens=*" %i IN ('minikube -p minikube docker-env --shell cmd') DO @%i
### 5. docker build --tag lambdafnimage:v1.0.0 .
### 6. notepad deployment.yaml
```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: lambdafunctionapp-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lambdafunctionapp-app
  template:
    metadata:
      labels:
        app: lambdafunctionapp-app
    spec:
      containers:
      - name: lambdafunctionapp-container
        image: lambdafnimage:v1.0.0
        imagePullPolicy: Never # Crucial for local images
        ports:
        - containerPort: 80 # Replace with your application's port
```
### 7. kubectl apply -f deployment.yaml
### 8. minikube kubectl -- get deployments
| NAME                              | READY | UP-TO-DATE  |AVAILABLE    |AGE          |
|-----------------------------------|-------|-------------|-------------|-------------|
|lambdafunctionapp-app-deployment   |  1/1  |    1        |    1        |   4m32s     |

### 9. notepad service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: lambdafunctionapp-app-service
spec:
  selector:
    app: lambdafunctionapp-app
  ports:
  - name: http
    protocol: TCP
    port: 30081
    targetPort: 80  # Replace with your application's port
  type: NodePort # Or ClusterIP, LoadBalancer depending on your needs
```

### 10. minikube kubectl -- apply -f service.yaml
### 11. minikube kubectl -- get services
| NAME                           | TYPE        | CLUSTER-IP        | EXTERNAL-IP |  PORT(S)           |     AGE    |
|--------------------------------|------------ |-------------------|-------------|--------------------|------------|
|kubernetes                      |  ClusterIP  |   10.96.0.1       |    <none>   |    443/TCP         |   3h12m    |
|lambdafunctionapp-app-service   |  NodePort   |   10.108.254.128  |    <none>   |    30081:30737/TCP |   26s      |

### 12. minikube service lambdafunctionapp-app-service --url
Open another command prompt.
### 13 kubectl get service
### 14. minikube service lambdafunctionapp-app-service
### 15. Open browser  => http://127.0.0.1:65363/api/MyHttpTrigger?name=Prince


## Some useful commands


netstat -ano | findStr "7071" => Find the app using port 7071 <br>
taskkill /F /PID 1234 => Kill an app with process Id 1234 <br>

docker images => list images <br>
docker rmi webapiapp:latest => delete image webapiapp:latest <br>




