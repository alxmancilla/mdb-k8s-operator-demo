# mdb-k8s-operator-demo
A simple demo to deploy a MongoDB replica set in K8s (using minikube)



Based on publicly available information, here’s a mini tutorial on how to use MongoDB with Kubernetes. This tutorial combines general recommendations and step-by-step guidance from the provided notes:

---

### Prerequisites:
Before starting, ensure you have:
1. **Docker** installed.
2. A Kubernetes cluster set up (you can use Minikube for local development).
3. **kubectl** (Kubernetes command-line tool) installed.
4. **helm** (Kubernetes package manager) installed.

---

### Part 1: Setting Up the Environment

#### Set up your Kubernetes Cluster

1. **Create a Kubernetes Cluster** (use Minikube for simplicity):
   ```bash
   minikube start --driver=docker
   ```

2. Verify Minikube installation:
   ```bash
   minikube version
   ```

---

### Part 2: Installing MongoDB Enterprise Kubernetes Operator

#### Install the operator:
You can install the operator either via Helm or YAML:

**Option 1: Install via Helm**
1. Clone the MongoDB Kubernetes Operator repo:
   ```bash
   git clone https://github.com/mongodb/mongodb-enterprise-kubernetes.git
   ```
2. Navigate into the cloned directory and install using Helm:
   ```bash
   helm install helm_chart/ --name mongodb-enterprise
   ```

**Option 2: Install via YAML**
1. Apply the operator configuration file:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-enterprise-kubernetes/master/mongodb-enterprise.yaml
   ```

---

### Part 3: Setting Up Namespaces and CRDs

1. Create a namespace for MongoDB:
   ```bash
   kubectl create namespace mongodb
   ```

2. Set the Kubernetes context to use the namespace:
   ```bash
   kubectl config set-context $(kubectl config current-context) --namespace=mongodb
   ```

3. Apply the custom resource definitions for MongoDB:
   ```bash
   kubectl apply -f mongodb-enterprise-kubernetes/crds.yaml --namespace mongodb
   ```

---

### Part 4: Installing Dependencies (Optional Cert-Manager)

If your setup requires TLS, install **cert-manager**:
1. Add the cert-manager Helm repository:
   ```bash
   helm repo add jetstack https://charts.jetstack.io
   helm repo update
   ```

2. Install cert-manager via Helm:
   ```bash
   helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.3.1 --set installCRDs=true
   ```

---

### Part 5: Deploy MongoDB Replica Set

1. Create a MongoDB replica set configuration file (replica-set.yaml):
   ```bash
   cat << EOF > replica-set.yaml
   apiVersion: mongodb.com/v1
   kind: MongoDB
   metadata:
     name: my-demo-rs
     namespace: mongodb
   spec:
     version: 8.0.0-ent
     type: ReplicaSet
     members: 3
     security:
       authentication:
         enabled: true
         modes: ["SCRAM"]
     opsManager:
       configMapRef:
         name: my-project
     credentials: organization-secret
     persistent: true
     podSpec:
       podTemplate:
         spec:
            containers:
             - name: mongodb-enterprise-database
               resources:
                 limits:
                   cpu: "1"
                   memory: 1G
                 requests:
                   cpu: "0.5"
                   memory: 500M
     externalAccess:
       externalService:
         annotations:
         spec:
           type: NodePort
     security:
       certsSecretPrefix: mdb
       tls:
         ca: custom-ca
         enabled: true
   EOF
   ```

2. Apply the replica set configuration:
   ```bash
   kubectl apply -f replica-set.yaml -n mongodb
   ```

---

### Part 6: Add a MongoDB User

Create a MongoDB user with SCRAM authentication:
1. Define user credentials in a secret:
   ```bash
   cat << EOF > rs-scram-secret.yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: my-scram-secret
   type: Opaque
   stringData:
     password: P@ssw0rd123!
   EOF
   ```

   Apply the secret:
   ```bash
   kubectl apply -f rs-scram-secret.yaml -n mongodb
   ```

2. Create a MongoDB user resource:
   ```bash
   cat << EOF > rs-scram-user.yaml
   apiVersion: mongodb.com/v1
   kind: MongoDBUser
   metadata:
     name: my-scram-user
   spec:
     passwordSecretKeyRef:
       name: my-scram-secret
       key: password
     username: user1
     db: admin
     mongodbResourceRef:
       name: my-demo-rs
     roles:
       - db: admin
         name: clusterAdmin
       - db: admin
         name: readWriteAnyDatabase
   EOF
   ```

   Apply the user configuration:
   ```bash
   kubectl apply -f rs-scram-user.yaml -n mongodb
   ```

---

### Part 7: Monitor Deployment Status

1. Check for MongoDB pods:
   ```bash
   kubectl get pod -o wide
   ```

2. Monitor MongoDB resource status:
   ```bash
   kubectl get mdb -n mongodb -w
   ```

3. Access MongoDB shell via a pod:
   ```bash
   kubectl exec --namespace mongodb -it my-demo-rs-0 -- bash
   mongosh --host <POD-IP-HERE> --port 27017 -u user1
   ```

---

### Notes on MongoDB Ops Manager

MongoDB Ops Manager can integrate with Kubernetes for managing and monitoring clusters. If using Ops Manager:
- Generate an API key and add it as a secret in Kubernetes.
- Set up a `ConfigMap` to reference Ops Manager settings for your MongoDB cluster.

---

This tutorial provides a high-level walkthrough of using MongoDB with Kubernetes using the Enterprise Operator. If you’d like more advanced details or troubleshooting tips, exploring MongoDB’s documentation might be helpful.

**References**

[Introducing the MongoDB Enterprise Operator for Kubernetes and OpenShift](https://www.mongodb.com/developer/products/ops-manager/enterprise-operator-kubernetes-openshift)
[Ensuring High Availability for MongoDB on Kubernetes](https://www.mongodb.com/developer/products/mongodb/mongodb-with-kubernetes)
[How to Deploy an Application in Kubernetes With the MongoDB Atlas Operator](https://www.mongodb.com/developer/products/atlas/kubernetes-operator-application-deployment)
[Deploying the MongoDB Enterprise Kubernetes Operator on Google Cloud](https://www.mongodb.com/developer/products/connectors/deploying-kubernetes-operator)
[Deploying MongoDB Across Multiple Kubernetes Clusters With MongoDBMulti](https://www.mongodb.com/developer/products/connectors/deploying-across-multiple-kubernetes-clusters)
[Atlas Kubernetes Operator - Learning Byte - Learn](https://learn.mongodb.com/learn/course/atlas-kubernetes-operator/learning-byte/learn)
