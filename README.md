# **Deploying PostgreSQL Logical Replication on Kubernetes**

This guide explains how to deploy PostgreSQL logical replication using Kubernetes. The setup includes a primary PostgreSQL server and a replica server, with automated initialization scripts and external access via LoadBalancer.

---

## **Overview**

Logical replication enables fine-grained control of data replication between PostgreSQL instances. In this setup:  
- **Primary Server** creates a replication slot and publication.  
- **Replica Server** subscribes to the publication to replicate data.

---

## **Prerequisites**

Before you begin, ensure the following:  
- Kubernetes cluster (e.g., Minikube, GKE, or EKS).  
- `kubectl` CLI configured for your cluster.  
- PostgreSQL Docker image (official `postgres:15` is used here).  
- Basic understanding of Kubernetes and PostgreSQL replication.

---

## **Architecture**

1. **Primary Server**: Configures `wal_level` for logical replication, creates a replication slot, and publishes data for replication.
2. **Replica Server**: Subscribes to the primary server's publication, replicating data in near real-time.
3. **External Access**: Both instances are exposed via LoadBalancer services.

---

## **Setup Steps**

### **1. Create Kubernetes Namespace**
Organize resources in a dedicated namespace.

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres-replication
```

Apply the namespace:
```bash
kubectl apply -f namespace.yaml
```

---

### **2. Create ConfigMaps for Initialization Scripts**

#### **Primary Server Initialization Script**
This script enables logical replication, creates a replication slot, and publishes data:

```yaml
# primary-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pg-primary-config
  namespace: postgres-replication
data:
  init-replication.sh: |
    #!/bin/bash
    set -e
    until pg_isready -U postgres; do sleep 2; done
    echo "wal_level = logical" >> /var/lib/postgresql/data/postgresql.conf.sample
    echo "max_replication_slots = 4" >> /var/lib/postgresql/data/postgresql.conf.sample
    echo "max_wal_senders = 4" >> /var/lib/postgresql/data/postgresql.conf.sample
    echo "host replication postgres 0.0.0.0/0 scram-sha-256" >> /var/lib/postgresql/data/pg_hba.conf
    psql -U postgres -c "SELECT pg_create_logical_replication_slot('pg_slot', 'pgoutput');"
    psql -U postgres -c "CREATE PUBLICATION pg_publication FOR ALL TABLES;"
```

#### **Replica Server Initialization Script**
This script configures the replica to subscribe to the primary server's publication:

```yaml
# replica-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pg-replica-config
  namespace: postgres-replication
data:
  init-replica.sh: |
    #!/bin/bash
    set -e
    until pg_isready -U postgres; do sleep 2; done
    psql -U postgres -c "CREATE SUBSCRIPTION my_subscription \
      CONNECTION 'host=pg-primary.postgres-replication.svc.cluster.local port=5432 user=postgres password=mysecretpassword dbname=mydb' \
      PUBLICATION pg_publication;"
```

Apply the ConfigMaps:
```bash
kubectl apply -f primary-configmap.yaml
kubectl apply -f replica-configmap.yaml
```

---

### **3. Deploy PostgreSQL Instances**

#### **Primary Server Deployment**
Deploy the primary PostgreSQL instance, mounting the initialization script from the ConfigMap.

```yaml
# primary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pg-primary
  namespace: postgres-replication
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pg-primary
  template:
    metadata:
      labels:
        app: pg-primary
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: mysecretpassword
            - name: POSTGRES_DB
              value: mydb
          volumeMounts:
            - name: init-script
              mountPath: /docker-entrypoint-initdb.d/init-replication.sh
              subPath: init-replication.sh
      volumes:
        - name: init-script
          configMap:
            name: pg-primary-config
```

#### **Replica Server Deployment**
Deploy the replica PostgreSQL instance.

```yaml
# replica-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pg-replica
  namespace: postgres-replication
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pg-replica
  template:
    metadata:
      labels:
        app: pg-replica
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: mysecretpassword
            - name: POSTGRES_DB
              value: mydb
          volumeMounts:
            - name: init-script
              mountPath: /docker-entrypoint-initdb.d/init-replica.sh
              subPath: init-replica.sh
      volumes:
        - name: init-script
          configMap:
            name: pg-replica-config
```

Apply the deployments:
```bash
kubectl apply -f primary-deployment.yaml
kubectl apply -f replica-deployment.yaml
```

---

### **4. Expose Services**

#### **Primary Server Service**
Expose the primary instance using a LoadBalancer for external access.

```yaml
# primary-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: pg-primary
  namespace: postgres-replication
spec:
  type: LoadBalancer
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: pg-primary
```

#### **Replica Server Service**
Expose the replica instance similarly.

```yaml
# replica-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: pg-replica
  namespace: postgres-replication
spec:
  type: LoadBalancer
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: pg-replica
```

Apply the services:
```bash
kubectl apply -f primary-service.yaml
kubectl apply -f replica-service.yaml
```

---

### **5. Verify the Setup**

#### **Check External IPs**
Get the external IPs assigned to the LoadBalancer services:
```bash
kubectl get svc -n postgres-replication
```

#### **Validate Logical Replication**
- **On the Primary Server**:
  ```bash
  psql -h <PRIMARY_EXTERNAL_IP> -U postgres -d mydb -c "SELECT * FROM pg_replication_slots;"
  ```
- **On the Replica Server**:
  ```bash
  psql -h <REPLICA_EXTERNAL_IP> -U postgres -d mydb -c "SELECT * FROM pg_stat_subscription;"
  ```

---

## **Enhancements**

1. **Secure Connections**: Use SSL/TLS for secure communication between instances.
2. **Scaling**: Add more replica instances by creating additional deployments.
3. **Monitoring**: Integrate with monitoring tools like Prometheus and Grafana for replication insights.
4. **Backups**: Use tools like pgBackRest to back up critical data.

---

## **Conclusion**

This documentation provides a comprehensive guide to deploying PostgreSQL logical replication on Kubernetes with external access. It includes automation via ConfigMaps, scalable deployments, and easy service exposure. Let me know if you need further assistance!
