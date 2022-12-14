# Database Connection

!!! reminder "Environment Variables"

    We are going to use some environment variables in this tutorial. Please make sure you have set them correctly.
    ```bash
    # check if the environment variables are set if not set them
    export NAMESPACE=<namespace>
    echo $NAMESPACE
    export URL=${NAMESPACE}.k8s.golog.ch
    echo $URL
    ```

You've now created a Deployment, exposed it as a Service, and scaled it up. But how do you know if it's working? How do you know if your application is actually connecting to the database? In this module, you'll learn how to connect to your application and run some simple commands to verify that it's working.

For this tutorial you'll have to take a look at the [stepping stone postgresql](https://wiki.golog.ch/wiki/gog-pro-010#Operations_-_PostgreSQL) documentation.

## :octicons-tasklist-16: **Task 1**: Connect Test Webserver to Database
We've already created a Deployment. 
Now we will create another Deployment with a simple webserver that connects to the database. 
Because the application needs credentials to connect to the database, we will use a Secret to store the credentials.

!!! warning "Database Credentials"

    If there is no `access.yaml` file at your home dir on the db server, create a new user and new database.  
    Note the username and password of the new user.

Create a file called `db-secret.yaml` with the following content (make sure to replace the placeholders with your own values):

```yaml
# get secrets from wiki
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
stringData:
  DB_USER: "<username>"
  DB_PASSWORD: "<password>"
  DB_NAME: "<db name>"
  DB_HOST: "<db host>"
  DB_PORT: "5432"
  DB_SSLMODE: "<disable/enable>"
```

!!! note

    You can generate the yaml file using the following command:

    ```bash
    kubectl create secret generic db-secret \
        --from-literal=DB_USER="<username>" \
        --from-literal=DB_PASSWORD="<password>" \
        --from-literal=DB_NAME="<db name>" \
        --from-literal=DB_HOST="<db host>" \
        --from-literal=DB_PORT="5432" \
        --from-literal=DB_SSLMODE="<disable/enable>" \
        --dry-run=client -o yaml > db-secret.yaml
    ```

    You will then find the values base64 encoded in the yaml file. You can decode them using the following command:

    ```bash
    echo -n "<base64 encoded value>" | base64 -d
    ```

Create a file called `db-deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test-postgresql-webserver
  name: test-postgresql-webserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-postgresql-webserver
  template:
    metadata:
      labels:
        app: test-postgresql-webserver
    spec:
      containers:
      - image: ghcr.io/natrongmbh/kubernetes-workshop-golog-test-postgresql-webserver:latest
        name: test-postgresql-webserver
        resources:
          requests:
            cpu: 10m
            memory: 16Mi
          limits:
            cpu: 20m
            memory: 32Mi
        envFrom:
        - secretRef:
            name: db-secret
```

Also create a file called `db-service.yaml` with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-postgresql-webserver
spec:
  selector:
    app: test-postgresql-webserver
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  type: NodePort
```

Create a file called `db-ingress.yaml` with the following content:

```bash
kubectl create --dry-run=client --namespace $NAMESPACE -o yaml -f - <<EOF >> db-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-postgresql-webserver
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/add-base-url: "true"
spec:
  tls:
  - hosts:
    - $URL
    secretName: ${NAMESPACE}-tls
  rules:
  - host: $URL
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-postgresql-webserver
            port:
              number: 8080
EOF
```

Finally, create the Deployment and Service:

```bash
kubectl apply -f db-secret.yaml --namespace $NAMESPACE
kubectl apply -f db-deployment.yaml --namespace $NAMESPACE
kubectl apply -f db-service.yaml --namespace $NAMESPACE
```

Before you create the Ingress, you need to delete the old Ingress:

```bash
kubectl delete ingress <ingress name> --namespace $NAMESPACE
```

Create the Ingress:

```bash
kubectl apply -f db-ingress.yaml --namespace $NAMESPACE
```

Check if the pod is running and also check the logs:

```bash
kubectl get pods --namespace $NAMESPACE
kubectl logs <pod name> --namespace $NAMESPACE
```
