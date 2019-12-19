---
title: "Notes on: Amazon EKS, Microservices Demo Shop App, and Gremlin"
---
![Kubernetes](/assets/k8s.jpg)

The following information comes directly from this [tutorial](https://www.gremlin.com/community/tutorials/how-to-install-and-use-gremlin-with-eks/). The purpose of this post is largely an effort to rewrite/clarify. If there are any issues, please report it on [Github](https://github.com/sbd/sbd.github.io/issues).

Once you have gone through this tutorial, you'll have set up the following:

- Create an AWS EKS (Elastic Kubernetes Service) cluster
- Deploy a Kubernetes Dashboard
- Deploy a [microservices demo application](https://github.com/GoogleCloudPlatform/microservices-demo) (maintained by Google)
- Install the Gremlin agent as a daemon set and launch an attack with Gremlin to ensure cluster reliability

## Pre-requisite Installations:

- **Homebrew**
```bash
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
- **AWS Auth**
```bash
$ brew install aws-iam-authenticator
```
- **AWS CLI**

    *For all the following commands: if using v2 of the AWS cli use `$ aws2`, otherwise `$ aws`*
```bash
$ curl "https://d1vvhvl2y92vvt.cloudfront.net/awscli-exe-macos.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
$ aws2 --version    
$ ls  ~/.aws
$ aws2 configure

    AWS Access Key ID [None]: AAAAA
    AWS Secret Access Key [None]: AAAAAAAAAA
    Default region name [None]: us-west-2
    Default output format [None]: json
```
- **Amazon EKS CLI, [ekscli](https://eksctl.io/)**
```bash
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp sudo mv /tmp/eksctl /usr/local/bin
$ brew tap weaveworks/tap
$ brew install weaveworks/tap/eksctl
$ eksctl version
```
- **Kubernetes CLI, [kubctl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)**
```bash
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl"
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
$ kubectl version
```
- **[Helm](https://helm.sh/docs/intro/quickstart/)** to connect the Gremlin client on your Kubernetes cluster
```bash
$ brew install kubernetes-helm
```

- [Create Gremlin account](https://app.gremlin.com/) for reliability testing (free)


## Tutorial Step by Step
```bash
$ eksctl create cluster
$ eksctl get clusters
```

    Should display:

    NAME	REGION
    ferocious-outfit-AAAA	us-west-2

```bash
$ sudo aws2 eks --region us-west-2  update-kubeconfig --name ferocious-outfit-AAAA
```
    Should display:

    Added new context arn:aws:eks:us-west-2:11111111:cluster/ferocious-outfit-AAAA to /Users/sbd/.kube/config

```bash
$ kubectl get nodes
```
+ **Note:** For AWS CLI v2 users, if you come across the [error](https://github.com/aws/aws-cli/issues/4675) below, you'll need to change `command: aws` to `command: aws2` in the config file (e.g. `/Users/sbd/.kube/config`).
```
Unable to connect to the server: getting credentials: exec: exec: "aws": executable file not found in $PATH
```

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

### Deploy Kubernetes Web UI/Dashboard
For more information, check out the [AWS docs](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html).

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

Create a `eks-admin-service-account.yaml` file with the following:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```

Apply the service account and role binding to your cluster:
```bash
$ kubectl apply -f eks-admin-service-account.yaml
```

Obtain an auth token for the service account with:
```bash
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

```bash
$ kubectl proxy
```
+ Should display: Starting to serve on 127.0.0.1:8001
+ **Note:** control + C to exit

Access the Kubernetes Dashboard endpoint with:
```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login
```
  - Select **Token** > enter in auth token returned from previous step > **Sign In**

### **Microservices Demo App Deployment**
```bash
$ git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
$ cd microservices-demo/
$ kubectl apply -f ./release/kubernetes-manifests.yaml
$ kubectl get pods
```     

*Once all pods have a `Running` status then continue with:*
```bash
$ kubectl get svc frontend-external -o wide
```

Under external-IP, you should see a URL (which you'll visit in your browser):
    `aaaaaa-aaaaaaaaaaa.us-west-2.elb.amazonaws.com`

### **Connect Gremlin to your Kubernetes cluster with Helm**

- To access your Gremlin certificates:
    - From the [Gremlin Dashboard]([https://app.gremlin.com/](https://app.gremlin.com/)): Under Company Settings > Teams tab > select your Team > Configuration tab > Certificates > Download
    - FYI:  Zip file will contain both private and public keys. Rename the certificate and key files to gremlin.cert and gremlin.key, respectively
```bash
$ kubectl create secret generic gremlin-team-cert \
> \--namespace=gremlin  \
> \--from-file=/Users/sbd/Documents/certificate/gremlin.cert \
> \--from-file=/Users/sbd/Documents/certificate/gremlin.key
```
- Set Gremlin environment variables:
    - To find your Gremlin Team ID: Under Company Settings > Teams tab > select your Team > Configuration tab > Team ID
    - FYI: the cluster ID can be set to anything
```bash
$ export GREMLIN_TEAM_ID="replace-text-aaaaaa-aaaaaa"
$ export GREMLIN_CLUSTER_ID="replace-text"
```

- Add Helm to your repo and upload the chart to Kubernetes:
```bash
  $ helm repo add gremlin https://helm.gremlin.com
  $ helm install gremlin/gremlin \
  > \--namespace gremlin \
  > \--generate-name \
  > \--set gremlin.teamID=$GREMLIN_TEAM_ID \
  > \--set gremlin.clusterID=$GREMLIN_CLUSTER_ID
```

### **Gremlin Dashboard**
+ The final steps of the process involves launching a shutdown attack to test the microservices demo app (Hipster Shop) and the EKS clusters' reliability.
+ Go to your [Gremlin account's dashboard](https://app.gremlin.com/attacks/infrastructure) > select **New Attack**:
    - Under `What do you want to attack` >  select **Kubernetes**
    - Under `Choose objects to target`:
        - **Choose a cluster**: select the custom cluster ID you made when declaring environment variables previously
        - **Choose a namespace**: default
        - Under the **Deployments** dropdown > select **cartservice**
    - Under `Choose a Gremlin`:
        - **Category**: State
        - **Attacks**: Shutdown
        - Select **Unleash Gremlin**
- Refresh `http://{CUSTOM_URL}.us-west-2.elb.amazonaws.com/` in your browser and you should see 500 Internal Server Error.

![500 Error](https://raw.githubusercontent.com/sbd/sbd.github.io/master/assets/gremlin-shutdown-attack.jpg)

### Kubernetes Logs
While the server is running, head back to:  
```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login
```
Navigate to **Cluster** > **Nodes** > select the node which contains the `cartservice` pod > from the `cartservice` row, note the **Restarts** figure.


## Teardown
Now that the tutorial is completed, we need to [delete the cluster](https://docs.aws.amazon.com/eks/latest/userguide/delete-cluster.html).

List all the services running in the cluster with:
```bash
$ kubectl get svc --all-namespaces
```

Firstly, delete any service(s) associated with an **EXTERNAL-IP** value:
```bash
$ kubectl delete svc frontend-external
```
**Note:** If services in your cluster are associated with a load balancer, delete the services before deleting the cluster so that the load balancers are deleted properly. Else, you can have orphaned resources in your VPC that prevent you from being able to delete the VPC.

```bash
$ eksctl delete cluster --name ferocious-outfit-AAAA
```

### Troubleshooting
If you encounter any issues, please report it on [Github](https://github.com/sbd/sbd.github.io/issues).
```bash
$ aws2 sts get-caller-identity
$ aws-iam-authenticator token -i environment_name.region.environment_type
$ aws2 eks list-clusters
$ kubectl get svc --all-namespaces
$ helm version
$ helm help
$ helm install   
```        
