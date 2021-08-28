## Build a Federation of Multiple Kubernetes Clusters With Kubefed V2

### A step-by-step guide to building a Kubernetes federation for managing multiple regions’ clusters with KubeFed
![image](https://user-images.githubusercontent.com/16441282/131210088-76e17973-a237-420c-aed6-5ff7996470c8.png)

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
![image](https://user-images.githubusercontent.com/16441282/131210160-6a9d78da-ef15-4044-8a56-25d778e7d817.png)

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
![image](https://user-images.githubusercontent.com/16441282/131210238-16dca25c-9513-432d-a95e-d8785b8622cf.png)

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
kubectl config use-context <context-name>
```
![image](https://user-images.githubusercontent.com/16441282/131210264-d5aa6376-c5b3-4de6-a37c-0eb6ff6c3ae7.png)


#### Use kubefedctl join to register clusters into the host cluster:
```
#Replace JOINED_CLUSTER_NAME, HOST_CLUSTER_NAME, HOST_CLUSTER_CONTEXT, JOINED_CLUSTER_CONTEXT
kubefedctl join JOINED_CLUSTER_NAME --host-cluster-name=HOST_CLUSTER_NAME --host-cluster-context=HOST_CLUSTER_CONTEXT --cluster-context=JOINED_CLUSTER_CONTEXT

example:
kubefedctl join us-west-oregon --host-cluster-name=host-cluster --host-cluster-context=zone --cluster-context=lab-a
kubefedctl join asia-pacific-tokyo --host-cluster-name=host-cluster --host-cluster-context=zone --cluster-context=lab-b
```
#### After you’ve joined clusters, you can check the status via the below command:
```
kubectl -n kube-federation-system get kubefedclusters
```
![image](https://user-images.githubusercontent.com/16441282/131210286-42909792-ecd0-4462-af5d-62ac9004bbc0.png)

Awesome!! Your federation clusters are ready now.

#### Deploy Nginx Service
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
    - name: asia-pacific-tokyo
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
            imagePullPolicy: IfNotPresent
            name: nginx
  placement:
    clusters:
    - name: us-west-oregon
    - name: asia-pacific-tokyo
EOF
```

#### After deployment, you will be able to see the nginx deployments are up and running in both clusters.
```
#Check for lab-a
kubectl get deployment -n test-namespace -owide --context lab-a  

#Check for lab-b
kubectl get deployment -n test-namespace -owide --context lab-b
```
![image](https://user-images.githubusercontent.com/16441282/131209999-bfb67085-351e-4557-8e6b-2bc76e61041e.png)

#### You can also override application deployment version, etc., for specific clusters only via defining overrides in the YAML file :

```
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
    - name: asia-pacific-tokyo
  overrides:
  - clusterName: asia-pacific-tokyo
    clusterOverrides:
    - path: "/spec/replicas"
      value: 5
    - path: "/spec/template/spec/containers/0/image"
      value: "nginx:1.17.0-alpine"
    - path: "/metadata/annotations"
      op: "add"
      value:
        foo: bar
    - path: "/metadata/annotations/foo"
      op: "remove"
EOF
```

After deployment, you’ll be able to see that the nginx deployment’s replicas, image version, etc., in lab-a are now modified.
```
#Check for lab-a
kubectl get deployment -n test-namespace -owide --context lab-a  

#Check for lab-b
kubectl get deployment -n test-namespace -owide --context lab-b
```

That’s all for the application deployment testing.
Now you’ll be able to use the federation to manage your clusters and application!
