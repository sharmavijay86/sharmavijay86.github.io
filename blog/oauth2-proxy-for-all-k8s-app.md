# Oauth2-proxy github auth integration for all k8s application.
Kubernetes is a very powerfull and moduler orchestration system. It gives option to modify it completely as per our use cases. One of the use case is setting up authorization mechanisms.
We had a requirement in which we wanted to secure most of application which does not have any native authentication system like kibana and kubernetes dashboard. It include some other application also.
Mainly flow is like this   
- User opens dashboard.devk8s.mylab.local ( application )
- User redirected to login.devk8s.mylab.local for github auth ( my oauth2-proxy url )
- User logged in with github authentication and presented (redirected) to application now. dashboard.devk8s.mylab.local

## Requirements:
To get this done in my k8s cluster i am going to use oauth2-proxy helm chart, nginx ingress controller and github app.

### Create github app
Note: to implement RBAC we need github orgnization and teams. i am taking example that i have orgnization name as **MyOrg** and team name is **adminusers**
1. Login to your github.com account. Nevigate to orgnization page. Go to settings --> developer settings --> new oauth app
2. Putt bello entries-
   Application Name:  oauth2proxy
   Home page url: https://login.devk8s.mylab.local
   Authorization callback URL: https://login.devk8s.mylab.local/oauth2/callback
3. Take a note of **client ID** and **Client secret**.

### Deploy oauth2-proxy by using helm chart

1. Bellow is the helm chart values file <script src="https://gist.github.com/sharmavijay86/1fa19b9775462e1794c48b9b3feb8c49.js"></script>
2. Make neccessory modifications
3. add helm repo 

```
helm repo add k8s-at-home https://k8s-at-home.com/charts/
helm repo update
```
Now you have already downloaded and created values.yaml file from above gist. Lets use it and deploy the helm chart.
```
helm install oauth2-proxy k8s-at-home/oauth2-proxy --namespace=auth-system --create-namespaces=true --values=values.yaml
```
Done!
### deploy hello world app
Bellow is the example manifest of hello k8s application. modify ingress host urls. Take a note here i am adding bellow annotations to work with oauth2.
```
nginx.ingress.kubernetes.io/auth-response-headers: X-Auth-Request-User,X-Auth-Request-Email
nginx.ingress.kubernetes.io/auth-signin: https://login.devk8s.mylab.local/oauth2/start
nginx.ingress.kubernetes.io/auth-url: https://login.devk8s.mylab.local/oauth2/auth
```
Full application manifest-
```
# hello-kubernetes.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-kubernetes
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: hello-kubernetes
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: hello-kubernetes
        image: sharmavijay86/hello-k8s:v1
        ports:
        - containerPort: 8080
        env:
          - name: MESSAGE
            value: Hello from git 
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/issuer: mylab
      nginx.ingress.kubernetes.io/auth-response-headers: X-Auth-Request-User,X-Auth-Request-Email
      nginx.ingress.kubernetes.io/auth-signin: https://login.devk8s.mylab.local/oauth2/start
      nginx.ingress.kubernetes.io/auth-url: https://login.devk8s.mylab.local/oauth2/auth
    name: nginx-test
spec:
    rules:
      - host: hello.devk8s.mylab.local
        http:
          paths:
            - backend:
                serviceName: hello-kubernetes
                servicePort: 80
              path: /
    tls:
    - hosts:
       - hello.devk8s.mylab.local
      secretName: hello-tls
  ```
  Now when i am trying to open hello.devk8s.mylab.local url it presents me github auth  and after login only i am getting access of hello k8s application.
  For same results with other applications like kibana or kubernetes dashboard, just add the three annotations as described above. done!
