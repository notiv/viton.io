+++
author = "Viton Vitanis"
date = "2016-02-01"
linktitle = "Apache Spark Cluster with Kubernetes and Docker - Part 2"
title = "Apache Spark Cluster with Kubernetes and Docker - Part 2"
description = "Spark Cluster on GCE with persistent Zeppelin Notebooks"
tags = [
  "apache",
  "spark",
  "docker",
  "kubernetes",
  "zeppelin"
]
categories = [
  "spark"
]
weight = 10
+++
In the [first part of this tutorial]({{< ref "post/2015-12-09-apache-spark-cluster-with-kubernetes-and-docker-part1.md" >}}), we deployed a Spark Cluster on Google Compute Engine (GCE) and exposed a Zeppelin notebook service in order to provide access to the cluster and launch jobs via a notebook frontend.

The problem with this configuration is that the notebooks are stored locally in the machine we create (specifically under "${ZEPPELIN_HOME}/notebook") and are lost the moment we shut the service down. In this second part, we create a GCE persistent disk to persistently store the Zeppelin notebooks. This requires the creation of a new docker image for Zeppelin and a new yaml file for the Zeppelin controller.

### Before we begin
Let's assume that all steps described in [Part 1]({{< ref "post/2015-12-09-apache-spark-cluster-with-kubernetes-and-docker-part1.md" >}}) have been completed. To deploy the new Zeppelin service, we first have to delete the existing one:
{{< highlight bash >}}
kubectl delete services zeppelin-controller
kubectl delete rc zeppelin-controller
{{< /highlight >}}

### Create a GCE persistent disk
Creating a persistent disk is a one-liner:
{{< highlight bash >}}
gcloud compute disks create spark-disk --size 20GB
{{< /highlight >}}
To get information about this disk run:
{{< highlight bash >}}
gcloud compute disks describe spark-disk
{{< /highlight >}}

### Create a new docker image for Zeppelin
From the kubernetes repository download the following files:
{{< highlight bash >}}
cd KubernetesSpark/examples/spark 
mkdir -p images/zeppelin
cd images/zeppelin
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/images/zeppelin/Dockerfile
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/images/zeppelin/docker-zeppelin.sh
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/images/zeppelin/zeppelin-env.sh
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/v1.2.0-alpha.5/examples/spark/images/zeppelin/zeppelin-log4j.properties
{{< /highlight >}}

In the Dockerfile change the spark-base version from _latest_ to _1.5.1_v2_.
{{< highlight bash >}}
#FROM gcr.io/google_containers/spark-base:latest  # before
FROM gcr.io/google_containers/spark-base:1.5.1_v2 # after
{{< /highlight >}}
Then in the _zeppelin-env.sh_ file modify the environment variable that defines where the Zeppelin notebooks are stored, e.g. export ZEPPELIN_NOTEBOOK_DIR="/pd/notebook".

![Zeppelin Env Var](/img/screenshots/KubernetesSpark/Screenshot_ZeppelinEnv.png)

Build the new image and tag it appropriately (replace _notiv_ by another username):
{{< highlight bash >}}
docker build -t notiv/zeppelin:1.5.1_v2 .
docker tag notiv/zeppelin:1.5.1_v2 eu.gcr.io/project-spark-k8s-vv2/zeppelin:1.5.1_v2
{{< /highlight >}}
Note that the Zeppelin image version (1.5.1_v2) should match the versions of the Spark images. 

Then upload the image to your registry (this can take a while, as the image is pretty big): 
{{< highlight bash >}}
gcloud docker push eu.gcr.io/project-spark-k8s-vv2/zeppelin:1.5.1_v2
{{< /highlight >}}

### Update the yaml file
Now we modify the yaml file for the Zeppelin controller, so that a) it corresponds to the new docker image and b) the persistent disk we created is provisioned when we start the controller.

![Zeppelin Yaml](/img/screenshots/KubernetesSpark/Screenshot_ZeppelinControllerYaml.png)

In [this gist](https://gist.github.com/notiv/23da1575acbf23780a21) you can find the code[^gist].

### Start the Zeppelin UI
Now we can start the Zeppelin service, as we did in [Part 1]({% post_url 2015-12-09-apache-spark-cluster-with-kubernetes-and-docker-part1 %}). Run:
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

### Final note
After setting up the cluster, the next thing I did, was to upload a dataset to play with. Here's the process.

First, make sure you have _gsutil_ installed[^gsutilSDK]. If this is not the case, follow the instructions [here](https://cloud.google.com/storage/docs/gsutil_install). Then create a bucket with a unique name:
{{< highlight bash >}}
gsutil mb -c nearline -l europe-west1 -p project-spark-k8s-vv2 gs://notiv-spark-bucket
{{< /highlight >}}
To upload a file (e.g. ~/linkage.csv) run the following command:
{{< highlight bash >}}
gsutil cp ~/linkage.csv gs://notiv-spark-bucket/data/linkage/linkage.csv
{{< /highlight >}}
and to access the file in the Zeppelin notebook:
{{< highlight bash >}}
val linkage = sc.textFile("gs://notiv-spark-bucket/data/linkage/linkage.csv")
{{< /highlight >}}


[^security]: **Important**: As noted in the previous post, everyone with this IP address has access to this notebook!
[^gsutilSDK]: If you installed the [Google Cloud SDK](https://cloud.google.com/sdk), then gsutil is already installed.
[^gist]: Don't forget to change the ProjectID when you define the image.
