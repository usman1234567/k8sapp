# k8sapp
Before writing any code, we first need to create a new project folder. Start by cloning the project from the git repo here. If you don't have git installed, install it first. 
```
git clone https://github.com/usman1234567/k8sapp.git
```
**Create Secrets**

Some of our app components here would require authentication information - PostgreSQL would require a database user and password, and Django would require the admin login information. We can provide such authentication information directly as environment variables. However since this is considered a bad practice, we would instead use a Kubernetes configuration object called Secrets.

Inside the k8s folder there is a file called 'app-secrets.yaml'. The file should look like below:
```
apiVersion: v1
kind: Secret
metadata:
    name: app-secrets
type: Opaque
data:
    POSTGRES_PASSWORD: cG9zdGdyZXNfcGFzc3dvcmQ=
    DJANGO_ADMIN_PASSWORD: YWRtaW5fcGFzc3dvcmQ=
    DB_PASSWORD: cG9zdGdyZXNfcGFzc3dvcmQ=
```
This is a very basic way of creating secrets.Now execute the below command so that the secrets are available within the Kubernetes cluster during run time.
```
kubectl apply -f app_secrets.yaml
```
You would see the below message.
```
secret/app-secrets created
```
**Create a ConfigMap**

For all of the remaining environment objects, we can create a config map. Essentially these are the variables whose values don't need to be encoded. Again, we could use the environment variables directly instead of creating a config map to store them. However, the advantage of using a config map is that in the event of changing the values in the future we only need to make changes in one place.
Inside the k8s folder there is a file called 'app-variables.yaml'. The file should look like below:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-variables
data:
  #env variables for the postgres component
  POSTGRES_USER: postgres_user
  POSTGRES_DB: predictiondb

  #env variables for the backend component
  DJANGO_ENV: development
  DEBUG: "1"
  SECRET_KEY: secretsecretsecretsecretsecret
  DJANGO_ALLOWED_HOSTS: "*"
  DJANGO_ADMIN_USER: admin
  DJANGO_ADMIN_EMAIL: "admin@example.com"

  DATABASE: postgres
  DB_ENGINE: "django.db.backends.postgresql"
  DB_DATABASE: predictiondb
  DB_USER: postgres_user
  DB_HOST: postgres-cluster-ip-service
  DB_PORT: "5432"
```
Now apply app-variables.yaml file by running the following command:
```
kubectl apply -f app-variables.yaml
```
You would see the below message.
```
configmap/app-variables created
```
**PersistentVolume**
Now for 'postgre'our database to work we need to create a PersistentVolume for our database. For that we need to deploy PersistentVolume manifest file 'postgre-pv.yaml' present in k8s directory.
It should look like this:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgre-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
    
```
As for hostPath goes you can change for your liking. This manifest file basically tells our cluster to set aside said ammount of storage, to be claimed later.

Now to create Persistent volume you need to run following command:
```
kubectl apply -f postgre-pv.yaml
```

**Create the PostgreSQL deployment**

Inside the k8s folder create a new file called 'component_postgres.yaml'. The file should look like below. 
```
###########################
# Persistent Volume Claim
###########################
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-persistent-volume-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100M
---
###########################
# Deployment
###########################
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: postgres
  template:
    metadata:
      labels:
        component: postgres
    spec:
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-persistent-volume-claim
      containers:
        - name: postgres-container
          image: postgres
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
              subPath: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: POSTGRES_USER
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: POSTGRES_DB
---
###########################
# Cluster IP Service
###########################
apiVersion: v1
kind: Service
metadata:
  name: postgres-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: postgres
  ports:
    - port: 5432
      targetPort: 5432
      
 ```
 The manifest file 'component_postgres.yaml' consists of three parts:
 
 **Persistent Volume Claim**
 
 First part of this deployment file is a persistent volume claim for postgre operations. As we created a Persistent Volume earlier now this part of deployment claims a said amount of storage from that volume.
 
 **Deployment**
 
 This is the most important part as this creates our Postgres engine. A Deployment is a Kubernetes controller object which creates the pods and their replicas. We have given our deployment a name - 'postgres-deployment'.
 
 **Service**
 
 This component exposes our pod to comunicate with other pods.
 
 Now deploy this file by running following command:
 ```
 kubectl apply -f component_postgres.yaml
 ```
 You would see the below message appear if your file executed correctly.
 ```
 persistentvolumeclaim/postgres-persistent-volume-claim created
deployment.apps/postgres-deployment created
service/postgres-cluster-ip-service created
```
**Create the Django Backend deployment**

First, we need to create the Docker image of our Django app and push the image either to Docker Hub or the container registry of a chosen cloud service provider. 
The Dockerfile for Django backend is in the 'backend' folder of repo:

Now we are ready to build our docker image. 
The tag I have used is 'stoffles/django_dev_image:latest'. You would need to run the below command with your own tag
```
docker build -t stoffles/django_dev_image:latest .
```
When image has done building you would see the following output:
```
Successfully built d41aee9e1213
Successfully tagged stoffles/django_dev_image:latest
```
After this step, you are ready to push your image to Docker Hub. Perform the below command (with your own Docker Hub ID) from your terminal.
```
docker push stoffles/django_dev_image:latest
```
Now we need to setup job obeject for our Django backend.The purpose of creating this job object is to perform a one-time operation - collect the static assets (in case of production) and also migrate the app database schema to our database.
Inside the k8s folder create a file called 'job_django.yaml'. The file should look like below.
```
###########################
# Job
###########################
apiVersion: batch/v1
kind: Job
metadata:
  name: django-job
spec:
  template:
    spec:
      containers:
      - name: django-job-container
        image: stoffles/django_dev_image:latest
        command: ["bash", "-c", "/usr/src/app/entrypoint.sh"]
        env:
          - name: DJANGO_ENV
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DJANGO_ENV
          - name: SECRET_KEY
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: SECRET_KEY
          - name: DEBUG
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DEBUG
          - name: DJANGO_ALLOWED_HOSTS
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DJANGO_ALLOWED_HOSTS
          - name: DB_ENGINE
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DB_ENGINE
          - name: DB_DATABASE
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DB_DATABASE
          - name: DB_USER
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DB_USER
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: app-secrets
                key: DB_PASSWORD
          - name: DB_HOST
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DB_HOST 
          - name: DB_PORT
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DB_PORT
          - name: DJANGO_ADMIN_USER
            valueFrom:
              configMapKeyRef:
                name: app-variables
                key: DJANGO_ADMIN_USER
          - name: DJANGO_ADMIN_PASSWORD
            valueFrom:
              secretKeyRef:
                name: app-secrets
                key: DJANGO_ADMIN_PASSWORD                
      restartPolicy: Never
  backoffLimit: 4
```

Execute the below from your within the project folder in your terminal.
```
kubectl apply -f job-django.yaml
```
Wait for some seconds and then execute "kubectl get pods" from the terminal. You should see a message like below.

```
NAME                                   READY   STATUS      RESTARTS   AGE
django-job-n2wbq                       0/1     Completed   0          3m21s
postgres-deployment-54656d7fb9-bfhvv   1/1     Running     0          18m
```
 Now we can safely spin up containers for our Django deployment.
 
  Create a file called 'component_django.yaml' within the k8s folder. The file should look like below.
  
```
###########################
# Deployment
###########################
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      component: django
  template:
    metadata:
      labels:
        component: django
    spec:
      containers:
        - name: django-container
          image: stoffles/django_dev_image:latest
          ports:
            - containerPort: 8000
          command: ["bash", "-c", "python manage.py runserver 0.0.0.0:8000"]
          env:
            - name: DJANGO_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DJANGO_ENV
            - name: SECRET_KEY
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: SECRET_KEY
            - name: DEBUG
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DEBUG
            - name: DJANGO_ALLOWED_HOSTS
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DJANGO_ALLOWED_HOSTS
            - name: DB_ENGINE
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DB_ENGINE
            - name: DB_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DB_DATABASE
            - name: DB_USER
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_PASSWORD
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DB_HOST 
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: app-variables
                  key: DB_PORT                    

---
###########################
# Cluster IP Service
###########################
apiVersion: v1
kind: Service
metadata:
  name: django-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: django
  ports:
    - port: 8000
      targetPort: 8000
 ```
 
 Now run the following command to spin up the containers:
 ```
 kubectl apply -f component_django.yaml
```

**Create the React FrontEnd deployment**

We first need to build our React dev image and push that to Docker Hub. In our Dockerfile located inside the frontend folder, we need to pass the value of the environment variable API_SERVER as an argument with the docker build command.
We need to build the image using the below command.
```
docker build -t stoffles/react_dev_image:latest . --build-arg API_SERVER=''
```
After builiding this image push it to Docker Hub:
```
docker push stoffles/react_dev_image:latest
```
Inside the k8s folder create a file called 'component_react.yaml'. The content of the file is shown below. 
```
###########################
# Deployment
###########################
apiVersion: apps/v1
kind: Deployment
metadata:
  name: react-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      component: react
  template:
    metadata:
      labels:
        component: react
    spec:
      containers:
        - name: react-container
          image: stoffles/react_dev_image:latest
          ports:
            - containerPort: 3000
          command: ["sh", "-c", "serve -s build -l 3000 --no-clipboard"]       

---
###########################
# Cluster IP Service
###########################
apiVersion: v1
kind: Service
metadata:
  name: react-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: react
  ports:
    - port: 3000
      targetPort: 3000
      
 ```
  Create the deployment with the below command.
  ```
  kubectl apply -f component_react.yaml
  ```
  You would see the following output:
  ```
  deployment.apps/react-deployment created
service/react-cluster-ip-service created
```
Now run:
```
kubectl get pods
```
Verify that each pod is running 

**Create Ingress**

**MAKE SURE YOU HAVE AN INGRESS CONTROLLER BEFORE ADDING INGRESS BECAUSE AN INGRESS MANIFEST FILE WON'T WORK WITHOUT INGRESS CONTROLLER. THERE ARE MUTIPLE IGRESS
CONTROLLERS BUT MOST WIDELY USED IS "NGINX-INGRESS"**
If you are using Minikube locally on your system you just need to enable addon for ingress controller:
```
minikube addons enable ingress
```
Create a file called 'ingress_service.yaml' within the k8s folder. The file contents are shown below.
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-service
  annotations:
    kubernetes.io/ingress.class: 'nginx'
spec:
  rules:
    - http:
        paths:
          ################
          # URL ROUTING #
          ################
          - path: /admin
            backend:
              serviceName: django-cluster-ip-service
              servicePort: 8000
          - path: /api
            backend:
              serviceName: django-cluster-ip-service
              servicePort: 8000
          ##########################
          # STATIC FOLDER ROUTING #
          ##########################
          - path: /static/admin/
            backend:
              serviceName: django-cluster-ip-service
              servicePort: 8000
          - path: /static/rest_framework/
            backend:
              serviceName: django-cluster-ip-service
              servicePort: 8000
          - path: /static/
            backend:
              serviceName: react-cluster-ip-service
              servicePort: 3000
          - path: /media/
            backend:
              serviceName: react-cluster-ip-service
              servicePort: 3000
          ################
          # URL ROUTING #
          ################
          - path: /
            backend:
              serviceName: react-cluster-ip-service
              servicePort: 3000
  ```
  Now apply ingress manifest file:
  
  ```
  kubectl apply -f ingress-service.yaml
  ```
 
  
  During ingress deployment you can encounter multiple hurdles, each problem is different based on where your Kuberenetes cluster is deployed, If your cluster self managed or deployed on premise you might need to add ingress controller yourself and other components. Defining Type of path an also be a problem. **SO CONTACT CUSTOMER SERVICE OR YOUR SENIOR FOR GUIDANCE IF YOU ARE UNABLE TO RESOLVE ISSUE**
  
  To see that our ingress is successfully running, execute the below command.
  ```
  kubectl get ingress
```
