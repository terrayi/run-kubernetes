# [Kubernetes](https://kubernetes.io/)

## [K3S](https://k3s.io/)

### [Install](https://docs.k3s.io/quick-start)

```shell
curl -sfL https://get.k3s.io | sh -
```

### Check for ready node

```shell
sudo k3s kubectl get node
```

## [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

** If K3S is installed, installation of Kubectl is not necessary.

### Download kubctl binary

- To download latest release:

```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

- or for specific version:

```shell
curl -LO https://dl.k8s.io/release/v1.27.0/bin/linux/amd64/kubectl
```

### Download checksum and validate kubectl binary

```shell
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

### Install kubectl

```shell
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Simple usages

#### Run a docker container for one time only

```shell
kubectl run -it --rm --image=docker-image-name:version --restart=Never container-name -- [command-to-run]
```

#### Run a single-instance stateful application

For full details, please refers to [Kubernetes documentation](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/).

##### Configurations

###### mysql-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

##### mysql-deployment.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

##### Deploy MySQL and PersistentVolumnClaim

```shell
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-pv.yaml
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-deployment.yaml
```

##### Retrieve information on deployment

```shell
kubectl describe deployment mysql
kubectl get pods -l app=mysql
kubectl describe pvc mysql-pv-claim
```

##### Access MySQL instance

```shell
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -ppassword
```

##### Delete deployment

```shell
kubectl delete deployment,svc mysql
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv-volume
```

### Port forwarding

#### Check the service is created

```shell
kubectl get service service-name
```

#### Verify pod is running and listening on port

```shell
kubectl get pod pod-name --template='{{(index (index .spec.containers 0).ports 0).containerPort}}{{"\n"}}'
```

#### Forward a local port to a port on the Pod

```shell
kubectl port-forward pod-name external-port:internal-port
```

For example with MySQL instance created in previous section,

```shell
kubectl port-forward mysql 3306:3306
```

## [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)

[Documentation](https://github.com/kubernetes/dashboard/tree/master/docs)

### [Installation](https://github.com/kubernetes/dashboard/blob/master/docs/user/installation.md)

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### Create a service account

#### dashboard-adminuser.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

#### Create service account

```shell
kubectl apply -f dashboard-adminuser.yaml
```

### Create a ClusterRoleBinding

#### dashboard-clusterrolebinding.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

#### Create ClusterRoleBinding

```shell
kubectl apply -f dashboard-clusterrolebinding.yaml
```

### Run proxy to access dashboard

```shell
kubectl proxy --port=8080
```

can acccess at ___http://localhost:8080/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/___

*** The reason the port is set specifically to 8080 is to run k9s.

### Create a bearer token

To log in to the Kubernetes dashboard, "Token" option should be selected and enter the token key generated from the following command.

```shell
kubectl -n kubernetes-dashboard create token admin-user
```

## [k9s](https://k9scli.io/)


### [Installation](https://k9scli.io/topics/install/)

For ArchLinux
```shell
pacman -S k9s
```

or download from [releases page](https://github.com/derailed/k9s/releases) and extract the binary

### Running

```shell
k9s [-n namespace]
```

### Commands

Refers ___https://k9scli.io/topics/commands/___

## [Helm](https://helm.sh/)

### [Installation](https://helm.sh/docs/intro/install/)

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### [Usage](https://helm.sh/docs/intro/using_helm/)

#### Add repo

```shell
helm repo add repo-name https://repo-url
```

#### Search package

```shell
helm search repo repo-name [package-name] 
```

#### Install package

```shell
helm install name package-name
```

## [KubeVirt](https://kubevirt.io/)

### Installation

#### Deploy KubeVirt operator

```shell
export VERSION=$(curl -s https://api.github.com/repos/kubevirt/kubevirt/releases | grep tag_name | grep -v -- '-rc' | sort -r | head -1 | awk -F': ' '{print $2}' | sed 's/,//' | xargs)
echo $VERSION
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
```

#### If nested virtualization cannot be enabled do enable KubeVirt emulation

```shell
kubectl -n kubevirt patch kubevirt kubevirt --type=merge --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

#### Deploy KubeVirt custom resource definition

```shell
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml
```

#### Verify components

- deployement

```shell
kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.phase}"
```

- compoents

```shell
kubectl get all -n kubevirt
```

### Virtctl

```shell
VERSION=$(kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.observedKubeVirtVersion}")
ARCH=$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/') || windows-amd64.exe
echo ${ARCH}
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-${ARCH}
chmod +x virtctl
sudo install virtctl /usr/local/bin
```

Refers ___https://kubevirt.io/labs/kubernetes/lab1___ for how to use Virtctl. And ___https://kubevirt.io/user-guide/virtual_machines/virtual_machine_instances/___ on more in-depth into creation of virtual machines.
