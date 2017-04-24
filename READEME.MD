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
  $ gcloud config set project gdg-demo-165104
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

Once that operation completes download the credentials for your cluster using the [gcloud CLI](https://cloud.google.com/sdk/):
```shell
$ gcloud container clusters get-credentials gdg-demo-165104
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

docker tag kube-sample-backend asia.gcr.io/gdg-demo-165104/kube-sample-backend

2. Then, use the gcloud command-line tool to push the image to the Google Container Registry:

gcloud docker -- push asia.gcr.io/gdg-demo-165104/kube-sample-backend

3. Create Deployments and Services:

Change Deployment config

$ kubectl --namespace=backend apply -f k8s
$ kubectl --namespace=backend get service sample-backend-svc

$ kubectl proxy
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

docker tag kube-sample-app asia.gcr.io/gdg-demo-165104/kube-sample-app

2. Then, use the gcloud command-line tool to push the image to the Google Container Registry:

gcloud docker -- push asia.gcr.io/gdg-demo-165104/kube-sample-app

3. Create Deployments and Services:

Change Deployment config

$ kubectl --namespace=frontend apply -f k8s
$ kubectl --namespace=frontend get service sample-app-svc

$ kubectl proxy
curl http://localhost:8001/api/v1/proxy/namespaces/frontend/services/sample-app-svc:80/


