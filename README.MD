## CLONE FROM [Lab: Build a Continuous Deployment Pipeline with Jenkins and Kubernetes](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes)

## Prerequisites
1. A Google Cloud Platform Account
2. [Enable the Google Compute Engine and Google Container Engine APIs](https://console.cloud.google.com/flows/enableapi?apiid=compute_component,container)
3. [Installing Cloud SDK](https://cloud.google.com/sdk/downloads)

## Do this first

In this section you will start your [Google Cloud Shell](https://cloud.google.com/cloud-shell/docs/) and clone the lab code repository to it.

1. Create a new Google Cloud Platform project: [https://console.developers.google.com/project](https://console.developers.google.com/project)

1. Click the Google Cloud Shell icon in the top-right and wait for your shell to open:

  ![](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes/blob/master/docs/img/cloud-shell.png)

  ![](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes/blob/master/docs/img/cloud-shell-prompt.png)

1. When the shell is open, set your default compute zone:

  ```shell
  $ gcloud config set compute/zone asia-east1-a
  $ gcloud config set project gdg-demo-165702
  ```

## Create a Kubernetes Cluster
You'll use Google Container Engine to create and manage your Kubernetes cluster. Provision the cluster with `gcloud`:

More config detail [here](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create)

```shell
gcloud container clusters create NAME [--additional-zones=ZONE,[ZONE,…]] [--async] [--cluster-ipv4-cidr=CLUSTER_IPV4_CIDR] [--disable-addons=[DISABLE_ADDONS,…]] [--disk-size=DISK_SIZE] [--no-enable-cloud-endpoints] [--no-enable-cloud-logging] [--no-enable-cloud-monitoring] [--image-type=IMAGE_TYPE] [--machine-type=MACHINE_TYPE, -m MACHINE_TYPE] [--max-nodes-per-pool=MAX_NODES_PER_POOL] [--network=NETWORK] [--node-labels=[NODE_LABELS,…]] [--num-nodes=NUM_NODES; default="3"] [--password=PASSWORD] [--scopes=SCOPE,[SCOPE,…]] [--subnetwork=SUBNETWORK] [--tags=TAG,[TAG,…]] [--username=USERNAME, -u USERNAME; default="admin"] [--wait] [--zone=ZONE, -z ZONE] [GLOBAL-FLAG …]
```

```shell
$ gcloud container clusters create gdg-demo \
  --num-nodes 2 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

View cluster info
```shell
$ kubectl cluster-info
```
Once that operation completes download the credentials for your cluster using the [gcloud CLI](https://cloud.google.com/sdk/):
```shell
$ gcloud container clusters get-credentials gdg-demo-165702
Fetching cluster endpoint and auth data.
kubeconfig entry generated for gdg-demo.
```

Confirm that the cluster is running and `kubectl` is working by listing pods:

```shell
$ kubectl get pods
```
You should see an empty response.

## Create namespace and Deploy backend app to Kubernetes

Create the `backend` namespace:
```shell
$ kubectl create ns backend
```

Check demo code for [backend](https://github.com/thachphongphong/kube-sample-backend)
```shell
$ git clone https://github.com/thachphongphong/kube-sample-backend.git
$ cd kube-sample-backend
$ docker build -t kube-sample-backend .
```

### [Pushing to the registry](https://cloud.google.com/container-registry/docs/pushing?hl=en_US&_ga=1.83862806.1224371239.1468984551)


1. Add the tag to your image:

```shell
docker tag kube-sample-backend asia.gcr.io/gdg-demo-165702/kube-sample-backend
```

2. Then, use the gcloud command-line tool to push the image to the Google Container Registry:

```shell
gcloud docker -- push asia.gcr.io/gdg-demo-165702/kube-sample-backend
```
3. Create Deployments and Services:

Change Deployment config

```shell
$ kubectl --namespace=backend apply -f k8s
$ kubectl --namespace=backend get service sample-backend-svc
```
Run via proxy
```shell
$ kubectl proxy
```
curl http://localhost:8001/api/v1/proxy/namespaces/backend/services/sample-backend-svc:8080/

## Create namespace and Deploy frontend app to Kubernetes

Create the `frontend` namespace:

```shell
$ kubectl create ns frontend
```

Check demo code for [frontend](https://github.com/thachphongphong/kube-sample-app.git)
```shell
$ git clone https://github.com/thachphongphong/kube-sample-app.git
$ cd kube-sample-app
$ docker build -t kube-sample-app .
```

### [Pushing to the registry](https://cloud.google.com/container-registry/docs/pushing?hl=en_US&_ga=1.83862806.1224371239.1468984551)


1. Add the tag to your image:
```shell
docker tag kube-sample-app asia.gcr.io/gdg-demo-165702/kube-sample-app
```

2. Then, use the gcloud command-line tool to push the image to the Google Container Registry:
```shell
gcloud docker -- push asia.gcr.io/gdg-demo-165702/kube-sample-app
```

3. Create Deployments and Services:

Change Deployment config
```shell
$ kubectl --namespace=frontend apply -f k8s
$ kubectl --namespace=frontend get service sample-app-svc
```

```shell
$ kubectl proxy
```
curl http://localhost:8001/api/v1/proxy/namespaces/frontend/services/sample-app-svc:80/

====================================================================
# [CI WITH JENKINS](https://cloud.google.com/solutions/jenkins-on-container-engine)

## Create namespace and Deploy jenkins to Kubernetes

Create the `jenkins` namespace:
```shell
$ kubectl create ns jenkins
```

### Create the Jenkins Home Volume
In order to pre-populate Jenkins with the necessary plugins and configuration for the rest of the tutorial, you will create
a volume from an existing tarball of that data.

```shell
gcloud compute images create jenkins-home-image --source-uri https://storage.googleapis.com/solutions-public-assets/jenkins-cd/jenkins-home-v3.tar.gz
gcloud compute disks create jenkins-home --image jenkins-home-image
```

### Create a Jenkins Deployment and Service
Here you'll create a Deployment running a Jenkins container with a persistent disk attached containing the Jenkins home directory.

First, set the password for the default Jenkins user. Edit the password in `jenkins/k8s/options` with the password of your choice by replacing _CHANGE_ME_. To Generate a random password and replace it in the file, you can run:

```shell
$ PASSWORD=`openssl rand -base64 15`; echo "Your password is $PASSWORD"; sed -i.bak s#CHANGE_ME#$PASSWORD# jenkins/k8s/options
Your password is 2UyiEo2ezG/CKnUcgPxt
```

Now create the secret using `kubectl`:
```shell
$ kubectl create secret generic jenkins --from-file=jenkins/k8s/options --namespace=jenkins
secret "jenkins" created
```

Additionally you will have a service that will route requests to the controller.

> **Note**: All of the files that define the Kubernetes resources you will be creating for Jenkins are in the `jenkins/k8s` folder. You are encouraged to take a look at them before running the create commands.

The Jenkins Deployment is defined in `kubernetes/jenkins.yaml`. Create the Deployment and confirm the pod was scheduled:

```shell
$ kubectl apply -f jenkins/k8s/
deployment "jenkins" created
service "jenkins-ui" created
service "jenkins-discovery" created
```

Check that your master pod is in the running state

```shell
$ kubectl get pods --namespace jenkins
NAME                   READY     STATUS    RESTARTS   AGE
jenkins-master-to8xg   1/1       Running   0          30s
```

Now, check that the Jenkins Service was created properly:

```shell
$ kubectl get svc --namespace jenkins
NAME                CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
jenkins-discovery   10.79.254.142   <none>        50000/TCP   10m
jenkins-ui          10.79.242.143   nodes         8080/TCP    10m
```

We are using the [Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) so that our builder nodes will be automatically launched as necessary when the Jenkins master requests them.
Upon completion of their work they will automatically be turned down and their resources added back to the clusters resource pool.

Notice that this service exposes ports `8080` and `50000` for any pods that match the `selector`. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster.
Additionally the `jenkins-ui` services is exposed using a NodePort so that our HTTP loadbalancer can reach it.

Kubernetes makes it simple to deploy an [Ingress resource](http://kubernetes.io/docs/user-guide/ingress/) to act as a public load balancer and SSL terminator.

The Ingress resource is defined in `jenkins/k8s/lb/ingress.yaml`. We used the Kubernetes `secrets` API to add our certs securely to our cluster and ready for the Ingress to use.

In order to create your own certs run:

```shell
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls.key -out /tmp/tls.crt -subj "/CN=jenkins/O=jenkins"
```

Now you can upload them to Kubernetes as secrets:
```shell
$ kubectl create secret generic tls --from-file=/tmp/tls.crt --from-file=/tmp/tls.key --namespace jenkins
```

Now that the secrets have been uploaded, create the ingress load balancer. Note that the secrets must be created before the ingress, otherwise the HTTPs endpoint will not be created.

```shell
$ kubectl apply -f jenkins/k8s/lb
```

<a name="connect-to-jenkins"></a>
### Connect to Jenkins

Now find the load balancer IP address of your Ingress service (in the `Address` field). **This field may take a few minutes to appear as the load balancer is being provisioned**:

```shell
$  kubectl get ingress --namespace jenkins
NAME      RULE      BACKEND            ADDRESS         AGE
jenkins      -         master:8080        130.X.X.X      4m
```

The loadbalancer will begin health checks against your Jenkins instance. Once the checks go to healthy you will be able to access your Jenkins instance:
```shell
$  kubectl describe ingress jenkins --namespace jenkins
Name:			jenkins
Namespace:		jenkins
Address:		130.211.14.253
Default backend:	jenkins-ui:8080 (10.76.2.3:8080)
TLS:
  tls terminates
Rules:
  Host	Path	Backends
  ----	----	--------
Annotations:
  https-forwarding-rule:	k8s-fws-jenkins-jenkins
  https-target-proxy:		k8s-tps-jenkins-jenkins
  static-ip:			k8s-fw-jenkins-jenkins
  target-proxy:			k8s-tp-jenkins-jenkins
  url-map:			k8s-um-jenkins-jenkins
  backends:			{"k8s-be-32371":"HEALTHY"}   <---- LOOK FOR THIS TO BE HEALTHY
  forwarding-rule:		k8s-fw-jenkins-jenkins
Events:
  FirstSeen	LastSeen	Count	From				SubobjectPath	Type		Reason	Message
  ---------	--------	-----	----				-------------	--------	------	-------
  2m		2m		1	{loadbalancer-controller }			Normal		ADD	jenkins/jenkins
  1m		1m		1	{loadbalancer-controller }			Normal		CREATE	ip: 130.123.123.123 <--- This is the load balancer's IP
```

Open the load balancer's IP address in your web browser, click "Log in" in the top right and sign in with the default Jenkins username `jenkins` and the password you configured when deploying Jenkins. You can find the password in the `jenkins/k8s/options` file.

> **Note**: To further secure your instance follow the steps found [here](https://wiki.jenkins-ci.org/display/JENKINS/Securing+Jenkins).


![](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes/blob/master/docs/img/jenkins-login.png)

### Your progress, and what's next
You've got a Kubernetes cluster managed by Google Container Engine. You've deployed:

* a Jenkins Deployment
* a (non-public) service that exposes Jenkins to its slave containers
* an Ingress resource that routes to the Jenkins service

You have the tools to build a continuous deployment pipeline. Now you need a sample app to deploy continuously.