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

