+++
author = "Viton Vitanis"
date = "2015-12-09"
linktitle = "Apache Spark Cluster with Kubernetes and Docker - Part 1"
title = "Apache Spark Cluster with Kubernetes and Docker - Part 1"
description = "Spark Cluster on GCE"
tags = [
  "apache",
  "spark",
  "docker",
  "kubernetes"
]
categories = [
  "spark"
]
weight = 10
+++

If you're into Data Science and Analytics, you've probably heard of [Apache Spark](http://spark.apache.org). I've been playing with this framework on and off for a while and completed two very interesting [MOOCs](http://www.edx.org/course/introduction-big-data-apache-spark-uc-berkeleyx-cs100-1x) on Spark; recently I decided 

### First things first
Click [this link](https://cloud.google.com/container-engine/docs/before-you-begin) and follow the instructions in the section "Before you begin": Create a google account, enable billing[^billing], install the gcloud command line interface (CLI) and install docker locally. _kubectl_ should also be installed:
{{< highlight bash >}}
gcloud components update kubectl
{{< /highlight >}}

As a second step, create a new project in the [Google Developers Console](http://console.developers.google.com/), give it an appropriate name (e.g. project-spark-k8s),

![Create New Project -1](/img/screenshots/KubernetesSpark/Screenshot_CreateNewProject_1.png)

if necessary, assign a unique ProjectID[^uniquePID]

![Create New Project - 2](/img/screenshots/KubernetesSpark/Screenshot_CreateNewProject_2.png)

and set this project as default using the CLI:
{{< highlight bash >}}
gcloud config set project "project-spark-k8s-vv2"
{{< /highlight >}}

Then go again to the [Before you begin](https://cloud.google.com/container-engine/docs/before-you-begin) link and enable the Google Container and Google Compute Engine APIs:

![Enable Container API](/img/screenshots/KubernetesSpark/Screenshot_EnableContainerAPI.png)

Finally create a local folder where you'll work:
{{< highlight bash >}}
mkdir -p KubernetesSpark/examples/spark
{{< /highlight >}}

and download the following files in it:
{{< highlight bash >}}
cd KubernetesSpark/examples/spark
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/spark-master-controller.yaml
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/spark-master-controller.yaml
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/spark-master-service.yaml
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/spark-webui.yaml
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/spark-worker-controller.yaml
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/zeppelin-controller.yaml
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/zeppelin-service.yaml
{{< /highlight >}}

### Create a cluster
To create a cluster run the following command:
{{< highlight bash >}}
gcloud container --project "project-spark-k8s" clusters create spark-cluster --num-nodes 4 --zone europe-west1-c --machine-type n1-standard-1 --scope "https://www.googleapis.com/auth/compute"
{{< /highlight >}}

As you can see, the cluster created, named "spark-cluster", consists of four _n1-standard-1_ nodes. To get more information about it, type:
To check the names of the four instances and get basic information about them (e.g. internal IP, external IP, etc), type:
{{< highlight bash >}}
gcloud compute instances list
{{< /highlight >}}

### Start Master Service
NOTE: This documentation is heavily based on [https://github.com/kubernetes/kubernetes/tree/master/examples/spark](https://github.com/kubernetes/kubernetes/tree/master/examples/spark).

First create a replication controller running the Spark Master service:
{{< highlight bash >}}
kubectl create -f examples/spark/spark-master-controller.yaml
{{< /highlight >}}

To find out details about the replication controller, first find its exact name:
{{< highlight bash >}}
kubectl get pods
{{< /highlight >}}
This command returns something like this:
{{< highlight bash >}}
NAME                            READY     STATUS    RESTARTS   AGE
spark-master-controller-xyz12   1/1       Running   0          1h
{{< /highlight >}}
Then run:
{{< highlight bash >}}
kubectl describe pods spark-master-controller-xyz12
{{< /highlight >}}
which returns information about the image, node, label etc. To check the log of the service, run:
{{< highlight bash >}}
kubectl logs spark-master-controller-xyz12
{{< /highlight >}}

Here a sample of the log output:

![Master Controller Logs](/img/screenshots/KubernetesSpark/Screenshot_MasterControllerLogs1.png)

As a second step after the creation of the replication controller, create a **logical** service endpoint that Spark workers can use to access the Master pod:
{{< highlight bash >}}
kubectl create -f examples/spark/spark-master-service.yaml
{{< /highlight >}}
and then create a service for the Spark Master WebUI:
{{< highlight bash >}}
kubectl create -f examples/spark/spark-webui.yaml
{{< /highlight >}}
To get basic information about the running services, run:
{{< highlight bash >}}
kubectl get services
{{< /highlight >}}

![Get Services](/img/screenshots/KubernetesSpark/Screenshot_GetServices.png)

For more detailed information about each service, run the corresponding _describe_ command, e.g.
{{< highlight bash >}}
kubectl describe service spark-master
{{< /highlight >}}
To connect to the Spark WebUI, use the cluster proxy:
{{< highlight bash >}}
kubectl proxy --port=8001
{{< /highlight >}}
at which point the UI is available at: [http://localhost:8001/api/v1/proxy/namespaces/default/services/spark-webui/](http://localhost:8001/api/v1/proxy/namespaces/default/services/spark-webui/).

![WebUI Init](/img/screenshots/KubernetesSpark/Screenshot_WebUIInit.png)

To make sure the command line is accessible press _Ctrl-Z_ and then type _bg_. This way the process can run in the background, while other commands can be run in the foreground.

### Start the Spark Workers
NOTE: The Spark workers need the Master service to be running.

First create a replication controller that manages the worker pods:
{{< highlight bash >}}
kubectl create -f examples/spark/spark-worker-controller.yaml
{{< /highlight >}}
To see that the workers are running, type again:
{{< highlight bash >}}
kubectl get pods
{{< /highlight >}}
The output now contains not only the spark-master-controller pod, but also the spark-worker-controllers. To check the logs run again:
{{< highlight bash >}}
kubectl logs spark-master-controller-xyz12
{{< /highlight >}}
in which case the output looks like that:

![Master Controller Logs](/img/screenshots/KubernetesSpark/Screenshot_MasterControllerLogs2.png)

The WebUI now looks like that:

![Image of WebUI](/img/screenshots/KubernetesSpark/Screenshot_WebUI.png)

### Start the Zeppelin UI
The Zeppelin UI pod will be used to launch jobs into the Spark cluster. To start it, run:
{{< highlight bash >}}
kubectl create -f examples/spark/zeppelin-controller.yaml
{{< /highlight >}}
To check whether Zeppelin is running:
{{< highlight bash >}}
kubectl get pods -lcomponent=zeppelin
{{< /highlight >}}
The output of this last command contains the name of the Zeppelin pod, e.g. zeppelin-controller-xyz12. To port-forward the Zeppelin port run the following command:
{{< highlight bash >}}
kubectl port-forward zeppelin-controller-xyz12 8080:8080
{{< /highlight >}}
Zeppelin can now be found at [http://localhost:8080](http://localhost:8080):

![Zeppelin Initial Screen](/img/screenshots/KubernetesSpark/Screenshot_ZeppelinInit.png)

As noted in [https://github.com/kubernetes/kubernetes/tree/master/examples/spark](https://github.com/kubernetes/kubernetes/tree/master/examples/spark) "On GKE, kubectl port-forward may not be stable over long periods of time. If you see Zeppelin go into Disconnected state (there will be a red dot on the top right as well), the port-forward probably failed and needs to be restarted."

To access the notebook externally, the service should be exposed:
{{< highlight bash >}}
kubectl expose rc zeppelin-controller --type="LoadBalancer"
{{< /highlight >}}

Run the following command to find the external IP:
{{< highlight bash >}}
kubectl get services zeppelin-controller
{{< /highlight >}}

![ExternalIP](/img/screenshots/KubernetesSpark/Screenshot_ExternalIP.png)

That means that Zeppelin can also be found at [http://104.155.67.27:8080](http://104.155.67.27:8080)[^security]

In the [next part of this tutorial]({{< ref "post/2016-02-01-apache-spark-cluster-with-kubernetes-and-docker-part2.md" >}}), we'll create a GCE persistent disk to persistently store the Zeppelin notebooks we create in this cluster. This requires the creation of a new docker image for Zeppelin and a new yaml file for the Zeppelin controller.

For now, enjoy!

[^billing]: The first $300 are on Google.
[^uniquePID]: Note that the ProjectID is a globally unique identifier.
[^security]: **Important**: Note that everyone with this IP address has access to this notebook!

