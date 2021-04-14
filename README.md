# aerospike-kubernetes-operator-workshop
This repo provides step by step guide to deploy aerospike-kubernetes-operator and do the Aerospike cluster lifecycle management

- [**Workshop**](#--workshop--)
  * [**Create a Kubernetes cluster**](#--create-a-kubernetes-cluster--)
  * [**Get the operator and prerequisite files from git**](#--get-the-operator-and-prerequisite-files-from-git--)
  * [**Deploy the operator**](#--deploy-the-operator--)
    + [**Prerequisites**](#--prerequisites--)
    + [**Deploy the Operator**](#--deploy-the-operator--)
  * [**Deploy Aerospike cluster**](#--deploy-aerospike-cluster--)
    + [**Prerequisites**](#--prerequisites---1)
    + [**Deploy Aerospike cluster**](#--deploy-aerospike-cluster---1)
    + [**Connect to the cluster**](#--connect-to-the-cluster--)
  * [**Add monitoring**](#--add-monitoring--)
    + [**Pre-Requisites**](#--pre-requisites--)
    + [**Add sidecar in aerospike_cluster_cr.yaml**](#--add-sidecar-in-aerospike-cluster-cryaml--)
    + [**Deploy Monitoring Stack**](#--deploy-monitoring-stack--)
    + [**View Dashboards**](#--view-dashboards--)
  * [**Scale up/down**](#--scale-up-down--)
  * [**Upgrade**](#--upgrade--)
  * [**Update config**](#--update-config--)
  * [**Rack management**](#--rack-management--)
    + [**Add/Remove racks**](#--add-remove-racks--)
    + [**Cluster node distribution in racks**](#--cluster-node-distribution-in-racks--)
    + [**Setting rack lavel storage and aerospikeConfig**](#--setting-rack-lavel-storage-and-aerospikeconfig--)
  * [**Access control management**](#--access-control-management--)
    + [**Creating a role**](#--creating-a-role--)
    + [**Creating a user with roles**](#--creating-a-user-with-roles--)
    + [**Changing a user's password**](#--changing-a-user-s-password--)
  * [**Multicluster setup**](#--multicluster-setup--)
    + [**Deploy destination cluter**](#--deploy-destination-cluter--)
    + [**Deploy source cluster**](#--deploy-source-cluster--)

# **Workshop**

## **Create a Kubernetes cluster**

To use the Aerospike Kubernetes Operator, you will need a working Kubernetes cluster with version 1.16, 1.17 or 1.18.

If you need to set up a new cluster: [See here for official guides](https://kubernetes.io/docs/setup/production-environment/)

There are specific guides for:

* [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)
* [Google GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster)
* [Microsoft AKS](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster)

## **Get the operator and prerequisite files from git**

Download the Aerospike Operator package [here](https://github.com/aerospike/aerospike-kubernetes-operator/tree/1.0.0/deploy/), and unpack it on the same computer where you normally run kubectl. The Operator package contains contains the CRDs and other resource files necessary to deploy the operator along with sample Aerospike cluster deployment resource files.

```sh
# To clone the Aerospike Github Operator repository
git clone https://github.com/aerospike/aerospike-kubernetes-operator.git
cd aerospike-kubernetes-operator
git checkout 1.0.0

# Create work dir to create this workshop related files
mkdir workshop
```

The `deploy` folder has the prerequisite files.

## **Deploy the operator**

### **Prerequisites**

  - Create a new `Kubernetes namespace`. This will help in putting all Aerospike related resource in a single logical space
  - Register `Aerospike CRD` [aerospike.com_aerospikeclusters_crd.yaml](https://github.com/aerospike/aerospike-kubernetes-operator/tree/1.0.0/deploy/crds/aerospike.com_aerospikeclusters_crd.yaml)
  - Setup [Role based access control (`RBAC`)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). RBAC helps in regulating access to the Kubernetes cluster and its resources based on the roles of individual users within your organization.

```sh
# Create a new Kubernetes namespace.
kubectl create namespace aerospike

# Register Aerospike CRD
kubectl apply -f deploy/crds/aerospike.com_aerospikeclusters_crd.yaml

# Setup RBAC
kubectl apply -f deploy/rbac.yaml
```

### **Deploy the Operator**

Aerospike Kubernetes Operator can be deployed by applying [deploy/operator.yaml](https://github.com/aerospike/aerospike-kubernetes-operator/tree/1.0.0/deploy/operator.yaml) file. 

```sh
# Deploy the Operator
kubectl apply -f deploy/operator.yaml

# Verify Operator is running
kubectl get pod -n aerospike -w

NAME                                             READY   STATUS    RESTARTS   AGE
aerospike-kubernetes-operator-79775b6cbf-c9m7k   1/1     Running   0          37s
```

This step could take some time initially as the operator image needs to be downloaded the first time.

**Check Operator logs**

Use the pod name obtained above to check the Operator logs.
```sh
kubectl -n aerospike logs -f aerospike-kubernetes-operator-5587bc7758-psn5t
```
```
t=2020-03-26T06:23:42+0000 lvl=info msg="Operator Version: 1.0.0" module=cmd caller=main.go:79
t=2020-03-26T06:23:42+0000 lvl=info msg="Go Version: go1.13.4" module=cmd caller=main.go:80
t=2020-03-26T06:23:42+0000 lvl=info msg="Go OS/Arch: linux/amd64" module=cmd caller=main.go:81
t=2020-03-26T06:23:42+0000 lvl=info msg="Version of operator-sdk: v0.12.0+git" module=cmd caller=main.go:82
t=2020-03-26T06:23:43+0000 lvl=info msg="Set sync period" module=cmd period=nil caller=main.go:183
t=2020-03-26T06:23:43+0000 lvl=info msg="Registering Components" module=cmd caller=main.go:199
....
```

## **Deploy Aerospike cluster**

To deploy an Aerospike cluster using the Operator, you will create an Aerospike custom resource file that describes what you want the cluster to look like (e.g. number of nodes, types of services, system resources, etc), and then push that configuration file into Kubernetes.

### **Prerequisites**

**Configure persistent storage**

The Aerospike Operator is designed to work with dynamically provisioned storage classes. A [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/) is used to dynamically provision the persistent storage. Aerospike Server pods may have different storage volumes associated with each service.

To apply a sample storage class based on your Kubernetes environment:

For GKE
```sh
kubectl apply -f deploy/samples/storage/gce_ssd_storage_class.yaml
```

For EKS
```sh
kubectl apply -f deploy/samples/storage/eks_ssd_storage_class.yaml
```

For MicroK8s
```sh
kubectl apply -f deploy/samples/storage/microk8s_filesystem_storage_class.yaml
```

See [Storage Provisioning](Storage-provisioning) for more details on configuring persistent storage.

**Create secrets**

Create secrets to setup Aerospike authentication, TLS, and features.conf. See [[Manage-TLS-Certificates]] for more details.

Aerospike secrets like `TLS certificates, security credentials, and features.conf` can be packaged in a single directory and converted to Kubernetes secrets

> Operator only supports Enterprise Aerospike Clusters. Hence secret should have the `features.conf` file. Copy your features.conf in `deploy/secrets` and create the secret using given command.

```sh
kubectl -n aerospike create secret generic aerospike-secret --from-file=deploy/secrets
```

Create a secret containing the `password` for Aerospike cluster `admin user` by passing the password from the command line.

> This is needed only if cluster is security enabled

```sh
kubectl -n aerospike create secret generic auth-secret --from-literal=password='admin123'
```

**Create Aerospike cluster Custom Resource (CR)**

Refer to the [[cluster configuration settings]] section for details on the Aerospike cluster custom resource (CR) file. Sample Aerospike cluster CR files for different configurations can be found [here](https://github.com/aerospike/aerospike-kubernetes-operator/tree/1.0.0/deploy/samples/).

This custom resource file can be edited later on to make any changes/manage the Aerospike cluster.

```sh
echo '
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike

spec:
  size: 2
  image: aerospike/aerospike-server-enterprise:5.4.0.5

  multiPodPerHost: true

  storage:
    filesystemVolumePolicy:
      cascadeDelete: true
      initMethod: deleteFiles
    volumes:
      - path: /opt/aerospike
        storageClass: ssd
        volumeMode: filesystem
        sizeInGB: 1

  aerospikeAccessControl:
    users:
      - name: admin
        secretName: auth-secret
        roles:
          - sys-admin
          - user-admin
          - read-write

  aerospikeConfigSecret:
    secretName: aerospike-secret
    mountPath:  /etc/aerospike/secret

  aerospikeConfig:
    logging:
      - name: /var/log/aerospike/aerospike.log
        any: info

    service:
      feature-key-file: /etc/aerospike/secret/features.conf

    security:
      enable-security: true

    namespaces:
      - name: test
        memory-size: 1000000000
        replication-factor: 2
        storage-engine:
          type: memory

  resources:
    requests:
      memory: 2Gi
      cpu: 200m

  podSpec:
    sidecars:
      - name: aerospike-prometheus-exporter
        image: aerospike/aerospike-prometheus-exporter:1.1.6
        ports:
        - containerPort: 9145
          name: exporter
        env:
        - name: AS_AUTH_USER
          value: admin
        - name: AS_AUTH_PASSWORD
          value: admin123
' > workshop/aerospike_cluster_cr.yaml
```


### **Deploy Aerospike cluster**

Use the CR yaml file that you created to deploy an Aerospike cluster.

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml

# Verify cluster status
# Check the pods to confirm the status. This step may take some time as the pod's provision resources, initialize, and are ready. Please wait for the pods to switch to the running state.
kubectl get pods -n aerospike -w

NAME                                             READY   STATUS    RESTARTS   AGE
aerocluster-0-0                                  0/2     Pending   0          1s
aerocluster-0-1                                  0/2     Pending   0          1s
aerospike-kubernetes-operator-79775b6cbf-tv7nq   1/1     Running   0          17m
aerocluster-0-0                                  0/2     Pending   0          3s
aerocluster-0-1                                  0/2     Pending   0          3s
aerocluster-0-0                                  0/2     Init:0/1   0          3s
aerocluster-0-1                                  0/2     Init:0/1   0          3s
aerocluster-0-0                                  0/2     Init:0/1   0          14s
aerocluster-0-1                                  0/2     Init:0/1   0          14s
aerocluster-0-1                                  0/2     PodInitializing   0          16s
aerocluster-0-0                                  0/2     PodInitializing   0          16s
aerocluster-0-0                                  2/2     Running           0          17s
aerocluster-0-1                                  2/2     Running           0          17s

```

### **Connect to the cluster**
 

**Port access**

When the Aerospike cluster is deployed in a `single pod per Kubernetes host mode`, ports `3000 (service port)` and `4333 (TLS port)` on all Kubernetes hosts should be accessible to all client and tools.

When the Aerospike cluster is configured to have `multiple pods per Kubernetes host mode`, port-range `(30000–32767)` on all Kubernetes hosts should be accessible to all client and tools.

Configure the `firewall rules` for the Kubernetes cluster accordingly.

Also see [[Cluster-configuration-settings]] file for the use of `multiPodPerHost` setting.

**Obtain the Aerospike node endpoints**

Run the kubectl describe command to get the IP addresses and port numbers:

```sh
 kubectl -n aerospike describe aerospikecluster aerocluster
```

The **Status > Pods*** section provides pod-wise access, alternate access, TLS access, and TLS alternate access endpoints as well as TLS name (if TLS is configured) to be used to access the cluster.

```sh
kubectl -n aerospike describe aerospikecluster aerocluster
Name:         aerocluster
Namespace:    aerospike
API Version:  aerospike.com/v1alpha1
Kind:         AerospikeCluster
.
.
Status:
.
.
  Pods:
    aerocluster-0-0:
      Aerospike:
        Access Endpoints:
          10.128.15.225:31312
        Alternate Access Endpoints:
          34.70.193.192:31312
        Cluster Name:  aerocluster
        Node ID:       0a0
        Tls Access Endpoints:
        Tls Alternate Access Endpoints:
        Tls Name:
      Host External IP:  34.70.193.192
      Host Internal IP:  10.128.15.225
      Image:             aerospike/aerospike-server-enterprise:5.4.0.5
      Initialized Volume Paths:
        /opt/aerospike
      Pod IP:        10.0.4.6
      Pod Port:      3000
      Service Port:  31312
.
.
```

**Connecting to the cluster**

When connecting from outside the Kubernetes cluster network, you need to use the host external IPs. By default, the Operator configures access endpoints to use Kubernetes host internal IPs and alternate access endpoints to use host external IPs.

Please refer to [network policy](Configuration#network-policy) configuration for details.

From the example status output, for pod aerocluster-0-0, the alternate access endpoint is 34.70.193.192:31312

With kubectl
```sh
# kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- asadm -h <cluster-name> -U <user> -P <password>
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- asadm -h aerocluster -U admin -P admin123
```

To use asadm from outside the Kubernetes network:
```sh
 asadm -h 34.70.193.192:31312 -U admin -P admin123 --services-alternate
```

To use asadm from within the Kubernetes network:
```sh
 asadm -h 10.128.15.225:31312 -U admin -P admin123
```

## **Add monitoring**

### **Pre-Requisites**

**Helm CLI**

For Ubuntu/Debian 

```sh
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

For more details - https://helm.sh/docs/intro/install/

### **Add sidecar in aerospike_cluster_cr.yaml**

```yaml
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike
spec:
  size: 2
  image: aerospike/aerospike-server-enterprise:5.2.0.7
  multiPodPerHost: true
.
.
  podSpec:
    sidecars:
      - name: aerospike-prometheus-exporter
        image: aerospike/aerospike-prometheus-exporter:1.1.6
        ports:
        - containerPort: 9145
          name: exporter
        env:
        - name: AS_AUTH_USER
          value: admin
        - name: AS_AUTH_PASSWORD
          value: admin123
.
.

```

**Apply the change**

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml

kubectl get pods -n aerospike -w

```

### **Deploy Monitoring Stack**

Now, let’s configure and deploy the monitoring stack.

Add Helm Repository For Aerospike Monitoring Stack Helm Chart

```sh
helm repo add summit-demo https://spkesan.github.io/aerospike-monitoring-stack/
```

Create a values YAML file to configure the Monitoring Stack

As a minimal configuration, let’s configure Prometheus to discover our Aerospike pods and the corresponding exporter targets.

See serviceDiscovery section in the below `aerospike_monitoring_stack.yaml` file.

This will let Prometheus discover exporter targets from aerospike namespace, pod labels with app: aerospike-cluster, container port name as exporter. Notice that the container port name exporter matches with what we specified in the podSpec.sidecars in aerospike_cluster_cr.yaml.

**aerospike_monitoring_stack.yaml**

```sh
echo '
prometheus:
  # Service discovery (to discover exporter targets)
  serviceDiscovery:
    # Namespace to look for targets
    namespaces:
      - aerospike
    # Prometheus relabel configs
    relabelConfigs:
      - source_labels: [__meta_kubernetes_namespace]
        separator: ;
        regex: aerospike
        replacement: $1
        action: keep
      - source_labels: [__meta_kubernetes_pod_label_app]
        separator: ;
        regex: aerospike-cluster
        replacement: $1
        action: keep
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        separator: ;
        regex: exporter
        replacement: $1
        action: keep
global:
  scrape_interval:     5s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 5s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
' > workshop/aerospike_monitoring_stack.yaml
```



```sh
# Deploy the helm chart
helm install aerospike-monitoring-stack summit-demo/aerospike-monitoring-stack \
  --namespace aerospike \
  -f workshop/aerospike_monitoring_stack.yaml

# Verify monitoring stack pods
kubectl get pods -n aerospike -w

NAME                                             READY   STATUS    RESTARTS   AGE
aerocluster-0-0                                  2/2     Running   0          13m
aerocluster-0-1                                  2/2     Running   0          13m
aerospike-kubernetes-operator-79775b6cbf-tv7nq   1/1     Running   0          30m
aerospike-monitoring-stack-alertmanager-0        1/1     Running   0          118s
aerospike-monitoring-stack-grafana-0             1/1     Running   0          117s
aerospike-monitoring-stack-prometheus-0          1/1     Running   0          116s
```

### **View Dashboards**

For Grafana dashboard,

```sh
kubectl port-forward -n aerospike service/aerospike-monitoring-stack-grafana 3000:80
```

Open http://localhost:3000 Log in to Grafana with default credentials : admin/admin

> To uninstall the aerospike-monitoring-stack, run: ```helm uninstall aerospike-monitoring-stack --namespace aerospike```

## **Scale up/down**

Change the `spec.size` field in the cr yaml file to scale up/down the cluster.

```yaml
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike
spec:
  size: 3
  image: aerospike/aerospike-server-enterprise:5.5.0.3
  .
  .
```

**Apply the change**
```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml

# Check the pods
kubectl get pods -n aerospike -w

NAME                                             READY   STATUS    RESTARTS   AGE
aerocluster-0-0                                  2/2     Running   0          4m48s
aerocluster-0-1                                  2/2     Running   0          5m27s
aerocluster-0-2                                  2/2     Running   0          4m30s
aerospike-kubernetes-operator-79775b6cbf-tv7nq   1/1     Running   0          39m
aerospike-monitoring-stack-alertmanager-0        1/1     Running   0          11m
aerospike-monitoring-stack-grafana-0             1/1     Running   0          11m
aerospike-monitoring-stack-prometheus-0          1/1     Running   0          11m

```

## **Upgrade**

The Operator performs a rolling upgrade of the Aerospike cluster one node at a time. 

To upgrade the Aerospike cluster change the `spec.image` field in the aerocluster CR to the desired Aerospike enterprise server docker image.

```yaml
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike
spec:
  size: 2
  image: aerospike/aerospike-server-enterprise:5.5.0.3
  .
  .
```

**Apply the change**

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml

# Check the pods
# The pods will undergo a rolling restart.
kubectl get pods -n aerospike -w

NAME                                             READY   STATUS    RESTARTS   AGE
aerocluster-0-0                                  2/2     Running   0          11m
aerocluster-0-1                                  2/2     Running   0          11m
aerocluster-0-2                                  2/2     Running   0          11m
aerocluster-0-2                                  2/2     Terminating   0          11m
aerocluster-0-2                                  0/2     Terminating   0          11m
aerocluster-0-2                                  0/2     Terminating   0          11m
```

After all the pods have restarted, use kubectl describe to get the status of the cluster. Check `image` for all Pods in status.

```sh
kubectl -n aerospike describe aerospikecluster aerocluster
Name:         aerocluster
Namespace:    aerospike
Kind:         AerospikeCluster
.
.
Status:
  .
  .
  Pods:
    aerocluster-0-0:
      Aerospike:
        Access Endpoints:
          10.128.15.225:31312
        Alternate Access Endpoints:
          34.70.193.192:31312
        Cluster Name:  aerocluster
        Node ID:       0a0
        Tls Access Endpoints:
        Tls Alternate Access Endpoints:
        Tls Name:
      Host External IP:  34.70.193.192
      Host Internal IP:  10.128.15.225
      Image:             aerospike/aerospike-server-enterprise:5.5.0.3
      Initialized Volume Paths:
        /opt/aerospike
      Pod IP:        10.0.4.6
      Pod Port:      3000
      Service Port:  31312
```

## **Update config**

Add an in-memory namespace `bar`. 

Currently we do not allow a persistent storage namespace to be added dynamically due to Kubernetes related limitations.  

```yaml
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike
spec:
  size: 2
  image: aerospike/aerospike-server-enterprise:5.5.0.3
  aerospikeConfig:
    namespaces:
      - name: bar
        memory-size: 1000000000
        replication-factor: 2
        storage-engine:
          type: memory
  .
  .
```

**Apply the change**

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml

# Check the pods
# Pods will undergo a rolling restart.
kubectl get pods -n aerospike -w

STATUS    RESTARTS   AGE
aerocluster-0-0                                  2/2     Running   0          15m
aerocluster-0-1                                  2/2     Running   0          15m
aerocluster-0-2                                  2/2     Running   0          15m
aerocluster-0-2                                  2/2     Terminating   0          15m
aerocluster-0-2                                  0/2     Terminating   0          15m
aerocluster-0-2                                  0/2     Terminating   0          15m
```


## **Rack management**

### **Add/Remove racks**

Add the Rack specific config for the Aerospike cluster in CR file.

```yaml
  rackConfig:
    namespaces:
      - test
    racks:
    #  Check this zone for your k8s cluster
    #  Zone can not be updated 
      - id: 1
        # zone: us-central1-b
      - id: 2
        # zone: us-central1-a
.
.
```

**Apply the change**

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml

# Check the pods
# Pods will be moved to new racks. Pods will be divided equally across all the racks as far as possible.
kubectl get pods -n aerospike -w

NAME                                             READY   STATUS    RESTARTS   AGE
aerocluster-1-0                                  2/2     Running   0          7m49s
aerocluster-1-1                                  2/2     Running   0          6m9s
aerocluster-2-0                                  2/2     Running   0          7m42s
```

### **Cluster node distribution in racks**

Cluster nodes are distributed across racks as evenly as possible. The cluster size is divided by the number of racks to get nodes per rack. If there are remainder nodes, they are distributed one by one across racks starting from first rack.

For e.g.

Nodes: 10, Racks: 4

Topology:
  - NodesForRack1: 3
  - NodesForRack2: 3
  - NodesForRack3: 2
  - NodesForRack4: 2

### **Setting rack lavel storage and aerospikeConfig**

Rack also provide for setting local storage and aerospikeConfig. If local storage is given for rack then rack will use this storage otherwise common global storage will be used. Here aerospikeConfig is config patch which will be merged with common global aerospikeConfig and will be used for rack.

```yaml
  rackConfig:
    namespaces:
      - test
    racks:
      - id: 1
        zone: us-central1-b
        aerospikeConfig:
          service:
            proto-fd-max: 18000
        storage:
          filesystemVolumePolicy:
            cascadeDelete: false
            initMethod: deleteFiles
          volumes:
            - path: /opt/aerospike
              storageClass: ssd
              volumeMode: filesystem
              sizeInGB: 3
```

## **Access control management**

### **Creating a role**

Add a role in `roles` list under `aerospikeAccessControl`.

Add `profiler` role.

```yaml
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike

spec:
  .
  .
  aerospikeAccessControl:
    roles: 
      - name: profiler
        privileges: 
          - read
    users:
      - name: admin
        secretName: auth-secret
        roles:
          - sys-admin
          - user-admin
          - read-write
```
**Apply the change**

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml
```

**Check in AQL**

```sh
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- aql -h aerocluster -U admin -P admin123

aql> 
aql> show roles
+------------------+------------------+-----------+
| role             | privileges       | whitelist |
+------------------+------------------+-----------+
| "data-admin"     | "data-admin"     | ""        |
| "profiler"       | "read"           | ""        |
| "read"           | "read"           | ""        |
| "read-write"     | "read-write"     | ""        |
| "read-write-udf" | "read-write-udf" | ""        |
| "sys-admin"      | "sys-admin"      | ""        |
| "user-admin"     | "user-admin"     | ""        |
| "write"          | "write"          | ""        |
+------------------+------------------+-----------+
8 rows in set (0.001 secs)

```

### **Creating a user with roles**

Create the secret for the user and add the user in `users` list under `aerospikeAccessControl`.

Create a secret `profile-user-secret` containing the password for the user `profiler` by passing the password from the command line:

```sh
kubectl -n aerospike create secret generic profile-user-secret --from-literal=password='userpass'
```
Add `profileUser` user having `profiler` role.

```yaml
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike

spec:
  .
  .
  aerospikeAccessControl:
    roles: 
      - name: profiler
        privileges: 
          - read
    users:
      - name: profileUser
        secretName: profile-user-secret
        roles:
          - profiler

      - name: admin
        secretName: auth-secret
        roles:
          - sys-admin
          - user-admin
          - read-write
```
**Apply the change**

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml
```

**Check in AQL**

```sh
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- aql -h aerocluster -U admin -P admin123

aql> show users
+---------------+-------------------------------------+
| user          | roles                               |
+---------------+-------------------------------------+
| "admin"       | "read-write, sys-admin, user-admin" |
| "profileUser" | "profiler"                          |
+---------------+-------------------------------------+
2 rows in set (0.000 secs)

```

### **Changing a user's password**

Create a new secret `new-profile-user-secret` containing the password for Aerospike cluster user `profileUser` by passing the password from the command line:

```sh
kubectl  -n aerospike create secret generic new-profile-user-secret --from-literal=password='newuserpass'
```
Update the `secretName` for `profileUser` to the new secret name `new-profile-user-secret`.

```yaml
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aerocluster
  namespace: aerospike

spec:
  .
  .
  aerospikeAccessControl:
    roles: 
      - name: profiler
        privileges: 
          - read
    users:
      - name: profileUser
        secretName: new-profile-user-secret
        roles:
          - profiler
          - user-admin

      - name: admin
        secretName: auth-secret
        roles:
          - sys-admin
          - user-admin
          - read-write
```

**Apply the change**

```sh
kubectl apply -f workshop/aerospike_cluster_cr.yaml
```

**Connect to AQL using new User and Password**

```sh
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- aql -h aerocluster -U profileUser -P newuserpass

aql> show user
+---------------+------------------------+
| user          | roles                  |
+---------------+------------------------+
| "profileUser" | "profiler, user-admin" |
+---------------+------------------------+
1 row in set (0.002 secs)

aql> 
```

## **Multicluster setup**

We can make XDR setup using multicluster support in operator. Operator can manage multiple aerospike clusters, deployed in a single namespace or in multiple namespaces.

Here we will deploy a source cluster and a destination cluster to create the XDR setup in the single kubernetes namespace.

### **Deploy destination cluter**

We will use the existing cluster as Destination cluster and deploy another cluster as the source cluster. 

### **Deploy source cluster**

**Configure persistent storage**

Using existing storage class created earlier

**Create secrets**

Using existing secrets created earlier. 

> If destination cluster is security enabled then secret created in this section should also have  a credentials file  for destination DC
```sh
$ cat password_DC1.txt
credentials
{
   username <dst_cluster_user>
   password <dst_cluster_pass>
}
```

If existing secret doesn't have `password_DC1.txt` file then update that or create a new secret with this file.

**XDR config to be used in source cluster cr file**

Two things needed for connecting to the destination DC are:
  - `node-address-ports` list of destination cluster pods
  - `auth-password-file` is the credentials file  for destination DC. If destination cluster is security enabled then a credentials file for destination DC should be provided as a secret in `spec.aerospikeConfigSecret` and it's path should be mentioned here

```yaml
    xdr:
      dcs:
        - name: dc1
          node-address-ports:
            - aerocluster-1-0 3000
            - aerocluster-2-0 3000

          auth-user: admin
          auth-password-file: /etc/aerospike/secret/password_DC1.txt
          namespaces:
            - name: test
```

**Deploy source cluster**

```sh

echo '
apiVersion: aerospike.com/v1alpha1
kind: AerospikeCluster
metadata:
  name: aeroclustersrc
  namespace: aerospike

spec:
  size: 2
  image: aerospike/aerospike-server-enterprise:5.5.0.3
  multiPodPerHost: true

  storage:
    filesystemVolumePolicy:
      cascadeDelete: true
      initMethod: deleteFiles
    volumes:
      - path: /opt/aerospike
        storageClass: ssd
        volumeMode: filesystem
        sizeInGB: 1

  aerospikeAccessControl:
    users:
      - name: admin
        secretName: auth-secret
        roles:
          - sys-admin
          - user-admin
          - read-write

  aerospikeConfigSecret:
    secretName: aerospike-secret
    mountPath:  /etc/aerospike/secret

  aerospikeConfig:
    logging:
      - name: /var/log/aerospike/aerospike.log
        any: info

    service:
      feature-key-file: /etc/aerospike/secret/features.conf

    security:
      enable-security: true

    xdr:
      dcs:
        - name: dc1
          node-address-ports:
            - aerocluster-1-0 3000
            - aerocluster-2-0 3000

          auth-user: admin
          auth-password-file: /etc/aerospike/secret/password_DC1.txt
          namespaces:
            - name: test

    namespaces:
      - name: test
        memory-size: 1000000000
        replication-factor: 2
        storage-engine:
          type: memory

  resources:
    requests:
      memory: 2Gi
      cpu: 200m

  podSpec:
    sidecars:
      - name: aerospike-prometheus-exporter
        image: aerospike/aerospike-prometheus-exporter:1.1.6
        ports:
        - containerPort: 9145
          name: exporter
        env:
        - name: AS_AUTH_USER
          value: admin
        - name: AS_AUTH_PASSWORD
          value: admin123
' > workshop/xdr_src_cluster_cr.yaml
```

```sh
kubectl apply -f workshop/xdr_src_cluster_cr.yaml

# Verify cluster status
kubectl get pods -n aerospike -w

NAME                                             READY   STATUS    RESTARTS   AGE
aerocluster-1-0                                  2/2     Running   0          22m
aerocluster-1-1                                  2/2     Running   0          20m
aerocluster-2-0                                  2/2     Running   0          22m
aeroclustersrc-0-0                               1/1     Running   0          48s
aeroclustersrc-0-1                               1/1     Running   0          48s

```

**Connecting to the cluster**

```sh
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- asadm -h aeroclustersrc -U admin -P admin123
```

**Add data in source cluster**
```sh
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- asbenchmark -h aeroclustersrc -U admin -P admin123 -k 1000 -o I -w I


kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- aql -h aeroclustersrc -U admin -P admin123

aql> 
aql> select * from test

```

**Verify data in destination cluster**

```sh
kubectl run -it --rm --restart=Never aerospike-tool -n aerospike --image=aerospike/aerospike-tools:latest -- aql -h aerocluster -U admin -P admin123

aql> 
aql> select * from test

```

> **Note**: Here Source and Destination clusters are deployed in a single namespace. If the user wants to deploy these clusters in different namespaces then the user has to follow [these](#multiple-aerospike-clusters-in-multiple-k8s-namespaces) steps.

