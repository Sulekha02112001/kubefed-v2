## Build a Federation of Multiple Kubernetes Clusters With Kubefed V2

### A step-by-step guide to building a Kubernetes federation for managing multiple regions’ clusters with KubeFed

#### What Is KubeFed?
KubeFed (Kubernetes Cluster Federation) allows you to use a single Kubernetes cluster to coordinate multiple Kubernetes clusters. It can deploy multiple-cluster applications in different regions and design for disaster recovery.
To learn more about KubeFed: https://github.com/kubernetes-sigs/kubefed

#### Prerequisites
Kubernetes clusters must be up and running: kubernetes v1.13+.
In this article, we’ll have three Kubernetes clusters. One is for installing Federation Control Plane as the host cluster (named zone). The others are for deploying applications named lab-a and lab-b.

#### KubeFed CLI Installation
```
wget https://github.com/kubernetes-sigs/kubefed/releases/download/v0.8.1/kubefedctl-0.8.1-linux-amd64.tgz
tar -zxvf kubefedctl-*.tgz
chmod u+x kubefedctl
sudo mv kubefedctl /usr/local/bin/ #make sure the location is in the PATH
```
#### You can check your kubefedctl version via:
```
kubefedctl version
```
### KubeFed Installation
#### KubeFed installation uses Helm chart for deployment. In the host cluster, you can use the following command to install Helm CLI Helm v3
```
wget -O helm.tar.gz https://get.helm.sh/helm-v3.5.4-linux-amd64.tar.gz
tar -zxvf helm.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm
helm repo add stable https://charts.helm.sh/stable
```
#### Install KubeFed v0.8.1 in kube-federation-system namespace (default) with the following command:
```

git clone https://github.com/kubernetes-sigs/kubefed.git
cd kubefed/charts/kubefed/
kubectl create namespace kube-federation-system
helm install  kubefed  . --namespace kube-federation-system

kubectl get pod  -n kube-federation-system
```
#### Cluster Registration

In the host cluster, set up Kubectl config for lab-a and lab-b, so we’ll be able to access those clusters via a context switch and use the context to join the federation:

Refer - https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/ and follow the steps 

(OR)

```
#Replace CLUSTERNAME, CLUSTERIP, USERNAME, TOKEN, CONTEXTNAME
kubectl config set-cluster CLUSTERNAME --server=CLUSTERIP
kubectl config set-credentials USERNAME --token="TOKEN"
kubectl config set-context CONTEXTNAME --cluster=CLUSTERNAME --user=USERNAME
```
#### Check the contexts for all clusters:
```
kubectl config get-contexts
```
#### Use kubefedctl join to register clusters into the host cluster:
```
#Replace JOINED_CLUSTER_NAME, HOST_CLUSTER_NAME, HOST_CLUSTER_CONTEXT, JOINED_CLUSTER_CONTEXT
kubefedctl join JOINED_CLUSTER_NAME --host-cluster-name=HOST_CLUSTER_NAME --host-cluster-context=HOST_CLUSTER_CONTEXT --cluster-context=JOINED_CLUSTER_CONTEXT

example:
kubefedctl join us-west-oregon --host-cluster-name=host-cluster --host-cluster-context=zone --cluster-context=lab-a
```
#### After you’ve joined clusters, you can check the status via the below command:
```
kubectl -n kube-federation-system get kubefedclusters
```

Awesome!! Your federation clusters are ready now.

#### Deploy a Service
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: test-namespace
EOF
cat << EOF | kubectl apply -f -
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: test-namespace
  namespace: test-namespace
spec:
  placement:
    clusters:
    - name: us-west-oregon
    - name: asia-pacific-india
EOF
cat << EOF | kubectl apply -f -
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: test-deployment
  namespace: test-namespace
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
  placement:
    clusters:
    - name: us-west-oregon
    - name: asia-pacific-india
EOF
```
