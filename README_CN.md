### 容器化改造



### K8S资源化改造

1. 创建系统所需的secret资源

```
# piggymetrics/kubernetes/0-secrets.yaml

---
apiVersion: v1
kind: Secret
metadata:
  name: piggymetrics
type: Opaque
data:  #Replace default values. This is only meant to create the structure.
  config_service_password: Q09ORklHX1NFUlZJQ0VfUEFTU1dPUkQ=
  auth_mongodb_password: QVVUSF9NT05HT0RCX1BBU1NXT1JE
  notification_service_password: Tk9USUZJQ0FUSU9OX1NFUlZJQ0VfUEFTU1dPUkQ=
  notification_mongodb_password: Tk9USUZJQ0FUSU9OX01PTkdPREJfUEFTU1dPUkQ=
  statistics_service_password: U1RBVElTVElDU19TRVJWSUNFX1BBU1NXT1JE
  statistics_mongodb_password: U1RBVElTVElDU19NT05HT0RCX1BBU1NXT1JE
  account_service_password: QUNDT1VOVF9TRVJWSUNFX1BBU1NXT1JE
  account_mongodb_password: QUNDT1VOVF9NT05HT0RCX1BBU1NXT1JE
  notification_email_host: c210cC5nbWFpbC5jb20=
  notification_email_port: NDY1
  notification_email_user: ZGV2LXVzZXI=
  notification_email_pass: ZGV2LXBhc3N3b3Jk

```

> 默认相关敏感数据已经Base64加密。

2. 创建RabbitMQ资源文件

```
# piggymetrics/kubernetes/1-rabbitmq-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: rabbitmq
  name: rabbitmq
spec:
  ports:
  - name: http
    port: 15672
    targetPort: 15672
  - name: amqp
    port: 5672
    targetPort: 5672
  selector:
    project: piggymetrics
    tier: infrastructure
    app: rabbitmq
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: rabbitmq
  name: rabbitmq
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: infrastructure
      app: rabbitmq
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: infrastructure
        app: rabbitmq
    spec:
      containers:
      - image: 192.168.3.48:5000/sqshq/rabbitmq:3-management
        name: rabbitmq
        ports:
        - containerPort: 15672
        - containerPort: 5672
        resources: {}
      restartPolicy: Always
status: {}

```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像，如有需要，也可以为该容器配置资源配额，如1C/4G。另外，如有需要可以使用https://www.rabbitmq.com/kubernetes/operator/operator-overview.html

3. 配置中心

```
# piggymetrics/kubernetes/2-config-service.yaml

---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: config
  name: config
spec:
  ports:
  - name: http
    port: 8888
    targetPort: 8888
  selector:
    project: piggymetrics
    tier: infrastructure
    app: config
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: config
  name: config
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: infrastructure
      app: config
  template:
    metadata:
      labels:    
        project: piggymetrics
        tier: infrastructure
        app: config
    spec:
      containers:
      - env:
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        image: 192.168.3.48:5000/sqshq/piggymetrics-config:latest
        name: config
        ports:
        - containerPort: 8888
        resources: {}
      restartPolicy: Always
status: {}
```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像


4. 服务发现

```
# piggymetrics/kubernetes/3-registry-service.yaml

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: registry
spec:
  ingressClassName: nginx
  rules:
  - host: registry.ex.test
    http:
      paths:
      - backend:
          service:
            name: registry
            port:
              number: 8761
        path: /
        pathType: Prefix
---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: registry
  name: registry
spec:
  ports:
  - name: http
    port: 8761
    targetPort: 8761
  selector:
    project: piggymetrics
    tier: infrastructure
    app: registry
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: registry
  name: registry
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: infrastructure
      app: registry
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: infrastructure
        app: registry
    spec:
      containers:
      - env:
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        image: 192.168.3.48:5000/sqshq/piggymetrics-registry:latest
        name: registry
        ports:
        - containerPort: 8761
        resources: {}
      restartPolicy: Always
status: {}
```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像

5. 网关

```
# piggymetrics/kubernetes/4-gateway-service.yaml

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gateway
spec:
  ingressClassName: nginx
  rules:
  - host: gateway.ex.test
    http:
      paths:
      - backend:
          service:
            name: gateway
            port:
              number: 80
        path: /
        pathType: Prefix
---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: frontend
    app: gateway
  name: gateway
spec:
  # comment or delete the following line if you want to use a LoadBalancer
  type: ClusterIP 
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 4000
    # comment or delete the following line if you want to use a LoadBalancer
    # nodePort: 30080
  selector:
    project: piggymetrics
    tier: frontend
    app: gateway
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  labels:
    project: piggymetrics
    tier: frontend
    app: gateway
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: frontend
      app: gateway
  template:
    metadata:
      creationTimestamp: null
      labels:
        project: piggymetrics
        tier: frontend
        app: gateway
    spec:
      containers:
      - name: gateway
        env:
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        image: 192.168.3.48:5000/sqshq/piggymetrics-gateway:latest
        
        ports:
        - containerPort: 4000
      restartPolicy: Always

```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像
> 另外，ingress的配置也需要根据自身环境的情况进行调整。

6. Auth服务

```
# piggymetrics/kubernetes/5-auth-mongodb-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: auth-mongodb
  labels:
    project: piggymetrics
    tier: database
    app: auth-mongodb
spec:
  ports:
  - name: mongo
    port: 27017
    targetPort: 27017
  selector:
    project: piggymetrics
    tier: database
    app: auth-mongodb
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-mongodb
  labels:
    project: piggymetrics
    tier: database
    app: auth-mongodb
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      project: piggymetrics
      tier: database
      app: auth-mongodb
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: database
        app: auth-mongodb
    spec:
      containers:
      - image: 192.168.3.48:5000/sqshq/piggymetrics-mongodb:latest
        name: auth-mongodb
        env:
          - name: MONGODB_PASSWORD
            valueFrom: 
              secretKeyRef:
                name: piggymetrics
                key: auth_mongodb_password
        ports:
          - containerPort: 27017
        resources: {}
      restartPolicy: Always
status: {}

```

```
# piggymetrics/kubernetes/6-auth-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: auth-service
  name: auth-service
spec:
  ports:
  - name: http
    port: 5000
    targetPort: 5000
  selector:
    project: piggymetrics
    tier: infrastructure
    app: auth-service
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: infrastructure
    app: auth-service
  name: auth-service
spec:
  replicas: 1
  strategy: 
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: infrastructure
      app: auth-service
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: infrastructure
        app: auth-service
    spec:
      containers:
      - env:
        - name: ACCOUNT_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: account_service_password
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        - name: MONGODB_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: auth_mongodb_password
        - name: NOTIFICATION_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_service_password
        - name: STATISTICS_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: statistics_service_password
        image: 192.168.3.48:5000/sqshq/piggymetrics-auth-service:latest
        name: auth-service
        ports:
          - containerPort: 5000
        resources: {}
      restartPolicy: Always
status: {}

```

> 注意： 需要替换yaml文件中的镜像为当前使用的镜像

7. Account服务

```
# piggymetrics/kubernetes/7-account-mongodb-service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: account-mongodb
  labels:
    project: piggymetrics
    tier: database
    app: account-mongodb
spec:
  ports:
  - name: mongo
    port: 27017
    targetPort: 27017
  selector:
    project: piggymetrics
    tier: database
    app: account-mongodb
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: account-mongodb
  labels:
    project: piggymetrics
    tier: database
    app: account-mongodb
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      project: piggymetrics
      tier: database
      app: account-mongodb
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: database
        app: account-mongodb
    spec:
      containers:
      - name: account-mongodb
        env:
        - name: INIT_DUMP
          value: account-service-dump.js
        - name: MONGODB_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: account_mongodb_password
        ports:
          - containerPort: 27017
        image: 192.168.3.48:5000/sqshq/piggymetrics-mongodb:latest
      restartPolicy: Always
```

```
# piggymetrics/kubernetes/8-account-service.yaml

---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: backend
    app: account-service
  name: account-service
spec:
  ports:
  - name: http
    port: 6000
    targetPort: 6000
  selector:
    project: piggymetrics
    tier: backend
    app: account-service
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: backend
    app: account-service 
  name: account-service
spec:
  replicas: 1
  strategy: 
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: backend
      app: account-service
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: backend
        app: account-service
    spec:
      containers:
      - env:
        - name: ACCOUNT_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: account_service_password
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        - name: MONGODB_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: account_mongodb_password
        ports:
          - containerPort: 6000
        image: 192.168.3.48:5000/sqshq/piggymetrics-account-service:latest
        name: account-service
      restartPolicy: Always

```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像

8. Statistics服务

```
# piggymetrics/kubernetes/9-statistics-mongodb-service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: statistics-mongodb
  labels:
    project: piggymetrics
    tier: database
    app: statistics-mongodb
spec:
  ports:
  - name: mongo
    port: 27017
    targetPort: 27017
  selector:
    project: piggymetrics
    tier: database
    app: statistics-mongodb
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: database
    app: statistics-mongodb
  name: statistics-mongodb
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      project: piggymetrics
      tier: database
      app: statistics-mongodb
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: database
        app: statistics-mongodb
    spec:
      containers:
      - env:
        - name: MONGODB_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: statistics_mongodb_password
        ports:
          - containerPort: 27017
        image: 192.168.3.48:5000/sqshq/piggymetrics-mongodb:latest
        name: statistics-mongodb
        resources: {}
      restartPolicy: Always
status: {}
```

```
# piggymetrics/kubernetes/A-statistics-service.yaml

---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: backend
    app: statistics-service
  name: statistics-service
spec:
  ports:
  - name: http
    port: 7000
    targetPort: 7000
  selector:
    project: piggymetrics
    tier: backend
    app: statistics-service
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: backend
    app: statistics-service
  name: statistics-service
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: backend
      app: statistics-service
  template:
    metadata:
      labels:
        project: piggymetrics
        tier: backend
        app: statistics-service
    spec:
      containers:
      - env:
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        - name: MONGODB_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: statistics_mongodb_password
        - name: STATISTICS_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: statistics_service_password
        ports:
          - containerPort: 7000
        image: 192.168.3.48:5000/sqshq/piggymetrics-statistics-service:latest
        name: statistics-service
        resources: {}
      restartPolicy: Always
status: {}
```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像

9. notification服务

```
# piggymetrics/kubernetes/B-notification-mongodb-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: notification-mongodb
  labels:
    project: piggymetrics
    tier: database
    app: notification-mongodb
spec:
  ports:
  - name: mongo
    port: 27017
    targetPort: 27017
  selector:
    project: piggymetrics
    tier: database
    app: notification-mongodb
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: database
    app: notification-mongodb
  name: notification-mongodb
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      project: piggymetrics
      tier: database
      app: notification-mongodb
  template:
    metadata:
      labels:    
        project: piggymetrics
        tier: database
        app: notification-mongodb
    spec:
      containers:
      - env:
        - name: MONGODB_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_mongodb_password
        ports:
          - containerPort: 27017
        image: 192.168.3.48:5000/sqshq/piggymetrics-mongodb:latest
        name: notification-mongodb
        resources: {}
      restartPolicy: Always
status: {}

```


```
# piggymetrics/kubernetes/C-notification-service.yaml

---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    tier: backend
    app: notification-service
  name: notification-service
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 8000
  selector:
    project: piggymetrics
    tier: backend
    app: notification-service
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    tier: backend
    app: notification-service
  name: notification-service
spec:
  replicas: 1
  strategy: 
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      tier: backend
      app: notification-service
  template:
    metadata:
      labels:       
        project: piggymetrics
        tier: backend
        app: notification-service
    spec:
      containers:
      - env:
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        - name: MONGODB_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_mongodb_password
        - name: NOTIFICATION_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_service_password
        - name: NOTIFICATION_EMAIL_HOST
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_email_host
        - name: NOTIFICATION_EMAIL_PORT
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_email_port
        - name: NOTIFICATION_EMAIL_USER
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_email_user
        - name: NOTIFICATION_EMAIL_PASS
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: notification_email_pass
        ports:
          - containerPort: 8000
        image: 192.168.3.48:5000/sqshq/piggymetrics-notification-service:latest
        name: notification-service
        resources: {}
      restartPolicy: Always
status: {}

```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像


10. turbine stream服务

```
# piggymetrics/kubernetes/E-turbine-stream-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    project: piggymetrics
    trier: infrastructure
    app: turbine-stream-service
  name: turbine-stream-service
spec:
  ports:
  - name: exposed
    port: 8989
    targetPort: 8989
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    project: piggymetrics
    trier: infrastructure
    app: turbine-stream-service
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: piggymetrics
    trier: infrastructure
    app: turbine-stream-service
  name: turbine-stream-service
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      project: piggymetrics
      trier: infrastructure
      app: turbine-stream-service
  template:
    metadata:
      labels:
        project: piggymetrics
        trier: infrastructure
        app: turbine-stream-service
    spec:
      containers:
      - env:
        - name: CONFIG_SERVICE_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: piggymetrics
              key: config_service_password
        image: 192.168.3.48:5000/sqshq/piggymetrics-turbine-stream-service:latest
        name: turbine-stream-service
        ports:
        - containerPort: 8989
        - containerPort: 8080
        resources: {}
      restartPolicy: Always
status: {}
```
> 注意： 需要替换yaml文件中的镜像为当前使用的镜像


### 服务发现

访问Eureka服务的地址： http://registry.ex.test/ ,并查看相关微服务是否已经注册成功。

> 提示：请将服务地址替换为您当前的服务地址

![image](https://user-images.githubusercontent.com/4653664/175772378-6629d7e4-ced8-4abd-90f6-c706a6dc6675.png)

### 访问应用

登录Piggymetrics应用地址： http://gateway.ex.test

> 提示：请将服务地址替换为您当前的服务地址

![image](https://user-images.githubusercontent.com/4653664/175774202-cf16bd99-9b48-4049-a7fc-171d724cdc59.png)

注册账户，如demotest/123456，跳过通知订阅。

![image](https://user-images.githubusercontent.com/4653664/175774274-112bf056-43ff-484d-b090-c1aa8d880be1.png)

点击"Get IN",然后依次添加`INCOME`\`EXPENSE`\`SAVINGS`信息

![image](https://user-images.githubusercontent.com/4653664/175774388-5a3f9618-89e4-4de1-89b9-fc272beefcc9.png)

然后点击“SAVECHANGES”

![image](https://user-images.githubusercontent.com/4653664/175774408-be065ffe-51d9-447e-bdcd-f0cf86627d0e.png)


### 断路器

###### 操作一、创建用户

```
curl --location --request POST --X POST 'http://gateway.ex.test/accounts/' \
--header 'User-Agent: Apipost client Runtime/+https://www.apipost.cn/' \
--header 'Content-Type: application/json' \
--data '{"username": "nginx1234","password": "123456"}'
```
> 提示：请将服务地址替换为您当前的服务地址
###### 操作二、获取token

```
curl --location --request POST --X POST 'http://gateway.ex.test/uaa/oauth/token' \
--header 'User-Agent: Apipost client Runtime/+https://www.apipost.cn/' \
--header 'Authorization: Basic YnJvd3Nlcjo=' \
--form 'scope=ui' \
--form 'username=nginx123' \
--form 'password=123456' \
--form 'grant_type=password'
```
> 提示：请将服务地址替换为您当前的服务地址

###### 操作三、保存账户数据

```
curl --location --request PUT --X PUT 'http://gateway.ex.test/accounts/current' \
--header 'User-Agent: Apipost client Runtime/+https://www.apipost.cn/' \
--header 'Authorization: Bearer  980e1ce3-b0d9-441c-ab50-fcd186966b58' \
--header 'Content-Type: application/json' \
--data '{
	"incomes": null,
	"expenses": null,
	"saving": {
		"amount": 2,
		"currency": "USD",
		"interest": 0,
		"deposit": false,
		"capitalization": false
	},
	"note": null
}'
```
> 提示：请将服务地址替换为您当前的服务地址

登录您的监控服务：http://monitoring.ex.test/hystrix

> 提示：请将服务地址替换为您当前的服务地址

并按照图示输入以下信息 `http://turbine-stream-service:8080/turbine/turbine.stream`，并点击"Monitor Stream"

![image](https://user-images.githubusercontent.com/4653664/175772271-77cdd263-bf1c-4892-885b-f93d8dcbe118.png)

依次执行操作1-2，然后重复执行操作3，登录断路器监控界面，查看断路器的状态是否已经`Open`。

![image](https://user-images.githubusercontent.com/4653664/175772202-c717450a-c47f-4575-97c9-19599cc69e2d.png)


### Spring Cloud组件高可用

###### 扩容Spring Cloud Config组件

```
[root@workstation ~]# kubectl -n demo scale deployment config --replicas=2
deployment.apps/config scaled

[root@workstation ~]# kubectl -n demo get pods | grep config
config-7b9cbf4c8b-8vhs2                  1/1     Running   0               36s
config-7b9cbf4c8b-cgnd6                  1/1     Running   0               4h52m
```
###### 扩容Spring Cloud Gateway组件

```
[root@workstation ~]# kubectl -n demo scale deployment gateway --replicas=2
deployment.apps/gateway scaled
[root@workstation ~]# kubectl -n demo get pods | grep gateway
gateway-58c776d459-rqjvb                 1/1     Running   0               2m10s
gateway-58c776d459-xv9pj                 1/1     Running   0               4h54m
```
###### 扩容Spring Cloud Eureka组件

更新`registry/src/main/resources/bootstrap.yml`文件内容如下:

```
spring:
  application:
    name: registry
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
      password: ${CONFIG_SERVICE_PASSWORD}
      username: user

eureka:
  instance:
    prefer-ip-address: true
  client:
    registerWithEureka: true
    fetchRegistry: true
    server:
      waitTimeInMsWhenSyncEmpty: 0
```

构建新的镜像

```
[root@workstation registry]# docker build . -t=192.168.3.48:5000/sqshq/piggymetrics-registry:v2

[root@workstation registry]# docker push 192.168.3.48:5000/sqshq/piggymetrics-registry:v2
```

更新镜像

```
[root@workstation ~]# kubectl -n demo set image deployment/registry registry=192.168.3.48:5000/sqshq/piggymetrics-registry:v2
deployment.apps/registry image updated

```

扩容

```
[root@workstation ~]# kubectl -n demo scale deployment registry --replicas=2
deployment.apps/registry scaled
[root@workstation ~]# kubectl -n demo get pods | registry
-bash: registry: command not found
[root@workstation ~]# kubectl -n demo get pods | grep registry
registry-7f877bc696-r6w2h                1/1     Running   0              24s
registry-7f877bc696-vvtjz                1/1     Running   0              60s
```

访问Eureka服务地址验证是否可以实现Eureka相互注册高可用

![image](https://user-images.githubusercontent.com/4653664/175775830-9b001221-ecb6-426c-ae1c-7ab2de837ba0.png)



