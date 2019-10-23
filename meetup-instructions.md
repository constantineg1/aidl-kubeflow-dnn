# Pre-requisites
* Google Cloud Platform Free Tier account - https://cloud.google.com/free/

# Installation
## Activate Google Cloud Shell
gcloud is the command-line tool for Google Cloud Platform. It comes pre-installed on Cloud Shell and supports tab-completion.
```
gcloud auth list
```

```
gcloud config list project
```

## Download the project files to Cloud Shell
```
git clone https://github.com/kubeflow/examples kubeflow-examples
cd kubeflow-examples
git checkout ec9020e85
```

## Setting Environment Variables
```
gcloud projects list
PROJECT_ID=<YOUR_CHOSEN_PROJECT_ID>
gcloud config set project $PROJECT_ID
ZONE=us-central1-a
cd ./mnist
WORKING_DIR=$PWD
```

## Installing Kustomize
```
mkdir $WORKING_DIR/bin
wget https://github.com/kubernetes-sigs/kustomize/releases/download/v2.0.3/kustomize_2.0.3_linux_amd64 -O $WORKING_DIR/bin/kustomize
chmod +x $WORKING_DIR/bin/kustomize
PATH=$PATH:${WORKING_DIR}/bin
```
## Install kfctl
```
export KUBEFLOW_TAG=0.6.2
wget -P /tmp https://github.com/kubeflow/kubeflow/releases/download/v${KUBEFLOW_TAG}/kfctl_v${KUBEFLOW_TAG}_linux.tar.gz
tar -xvf /tmp/kfctl_v${KUBEFLOW_TAG}_linux.tar.gz -C ${WORKING_DIR}/bin
```

## Enabling the API
Ensure the Google Kubernetes Engine (GKE) API is enabled for your project
```
gcloud services enable container.googleapis.com
```

# Cluster Setup
## Overview
To create a managed Kubernetes cluster on Kubernetes Engine using kfctl, we will walk through the following steps:

* Create an application directory with local configs
* Generate creation files
* Apply creation files

## Create App & Configs
To create an application directory with local config files and enable APIs for your project, run these commands:
```
export KUBEFLOW_USERNAME=aihub_user
export KUBEFLOW_PASSWORD=password
export KFAPP=kubeflow-aihub
kfctl init ${KFAPP} --project ${PROJECT_ID} --use_basic_auth \
  --config=https://raw.githubusercontent.com/kubeflow/kubeflow/c54401e/bootstrap/config/kfctl_gcp_basic_auth.0.6.2.yaml
```
This creates the file `kubeflow-aihub/app.yaml`, which defines a full, default Kubeflow installation.

## Generate Creation Files
To generate the files used to create the deployment, including a cluster and service accounts, run these commands:
```
cd ${WORKING_DIR}/${KFAPP}
kfctl generate all --zone ${ZONE}
sed -i 's/n1-standard-8/n1-standard-4/g' gcp_config/cluster-kubeflow.yaml
```

## Apply
This generates several new directories, each with customized files. To use the generated files to create all the objects in your project, run this command:
```
kfctl apply all
```
Cluster creation will take a few minutes to complete. While it is running, you can view instantiation of the following objects in the GCP Console:

In Deployment Manager, two deployment objects will appear:
* kubeflow-aihub-storage
* kubeflow-aihub
In Kubernetes Engine, a cluster will appear: kubeflow-aihub
* In the Workloads section, a number of Kubeflow components
* In the Services section, a number of Kubeflow services
After the cluster is set up, use gcloud to fetch its credentials so you can communicate with it using kubectl
```
gcloud container clusters get-credentials ${KFAPP} --zone ${ZONE} --project ${PROJECT_ID}
```
Switch to the kubeflow namespace
```
kubectl config set-context $(kubectl config current-context) --namespace=kubeflow
```
Check view all the Kubeflow resources deployed on the cluster
```
kubectl get all
```

## Setup storage 
Create a storage bucket on Google Cloud Storage to hold our trained model. Note that the name you choose for your bucket must be unique across all of GCS.
```
BUCKET_NAME=${KFAPP}-${PROJECT_ID}
gsutil mb gs://${BUCKET_NAME}/
```

# Training
## Building the Container

To deploy the code to Kubernetes, you have to first build it into a container image:
```
IMAGE_PATH=us.gcr.io/$PROJECT_ID/kubeflow-train
docker build $WORKING_DIR -t $IMAGE_PATH -f $WORKING_DIR/Dockerfile.model
```

Now, test the new container image locally to make sure everything is working as expected
```
docker run -it $IMAGE_PATH
```
You should see training logs start appearing in your console:

If you're seeing logs, that means training is working and you can terminate the container with Ctrl+c. 

## Push Container into GCR (Your Docker Repository)
Now that you know that the container can run locally, you can safely upload it to Google Container Registry (GCR) so you can run it on your cluster.
```
//allow docker to access our GCR registry
gcloud auth configure-docker --quiet

docker push $IMAGE_PATH
```
You should now see your new container listed on the GCR console
https://console.cloud.google.com/gcr

## Training on the Cluster
Finally, you can run the training job on the cluster. First, move into the training directory
```
cd $WORKING_DIR/training/GCS
```
If you look around this directory, you will see a number of YAML files. You can now use kustomize to configure the manifests. First, set a unique name for the training run
```
kustomize edit add configmap mnist-map-training \
    --from-literal=name=my-train-1
```
Next, set some default training parameters (number of training steps, batch size and learning rate)

```
kustomize edit add configmap mnist-map-training \
    --from-literal=trainSteps=200
kustomize edit add configmap mnist-map-training \
    --from-literal=batchSize=100
kustomize edit add configmap mnist-map-training \
    --from-literal=learningRate=0.01
```
Now, configure the manifest to use your custom bucket and training image

```
kustomize edit set image training-image=${IMAGE_PATH}:latest
kustomize edit add configmap mnist-map-training \
    --from-literal=modelDir=gs://${BUCKET_NAME}/my-model
kustomize edit add configmap mnist-map-training \
    --from-literal=exportDir=gs://${BUCKET_NAME}/my-model/export
```
One thing to keep in mind is that the python training code has to have permissions to read/write to the storage bucket you set up. Kubeflow solves this by creating a service account within your project as a part of the deployment. You can verify this by listing your service accounts:

```
gcloud --project=$PROJECT_ID iam service-accounts list | grep $KFAPP
```
This service account should be automatically granted the right permissions to read and write to our storage bucket. Kubeflow also added a Kubernetes secret called "user-gcp-sa" to our cluster, containing the credentials needed to authenticate as this service account within our cluster:

```
kubectl describe secret user-gcp-sa
```
To access your storage bucket from inside our training container, you need to set the GOOGLE_APPLICATION_CREDENTIALS environment variable to point to the json file contained in the secret. You can do this by setting a few more Kustomize parameters:
https://cloud.google.com/docs/authentication/getting-started
```
kustomize edit add configmap mnist-map-training \
    --from-literal=secretName=user-gcp-sa
kustomize edit add configmap mnist-map-training \
    --from-literal=secretMountPath=/var/secrets
kustomize edit add configmap mnist-map-training \
    --from-literal=GOOGLE_APPLICATION_CREDENTIALS=/var/secrets/user-gcp-sa.json
```
Now that all the parameters are set, use Kustomize to build the new customized YAML file:

```
kustomize build .
```
You can pipe this YAML manifest to kubectl to deploy the training job to the cluster:


```
kustomize build . | kubectl apply -f -
```
After applying the component, there should be a new tf-job on the cluster called my-train-1-chief-0 . You can use kubectl to query some information about the job, including its current state.

```
kubectl describe tfjob
```
For even more information, you can retrieve the python logs from the pod that's running the container itself (after the container has finished initializing):

```
kubectl logs -f my-train-1-chief-0
```
When training is complete, you can query your bucket's data using gsutil. You should see the model data added to your bucket:

```
gsutil ls -r gs://${BUCKET_NAME}/my-model/export
```
Alternatively, you can check the contents of your bucket through the GCP Cloud Console.
https://console.cloud.google.com/storage

# Serving
Now that you have a trained model, it's time to put it in a server so it can be used to handle requests. To do this, move into the "serving/GCS" directory
```
cd $WORKING_DIR/serving/GCS
```
The Kubeflow manifests in this directory contain a TensorFlow Serving implementation. You simply need to point the component to your GCS bucket where the model data is stored, and it will spin up a server to handle requests. Unlike the tf-job , no custom container is required for the server process. Instead, all the information the server needs is stored in the model file.
https://www.tensorflow.org/versions/r1.1/deploy/tfserve

Like before, start by setting some parameters to customize the deployment for our use. First, set the name for the service:

```
kustomize edit add configmap mnist-map-serving \
    --from-literal=name=mnist-service
```

Next, point the server at the trained model in your GCP bucket:
```
kustomize edit add configmap mnist-map-serving \
    --from-literal=modelBasePath=gs://${BUCKET_NAME}/my-model/export
```
Now, deploy the server to the cluster:

```
kustomize build . | kubectl apply -f -
```
If you describe the new service, you'll see it's listening for connections within the cluster on port 9000:

```
kubectl describe service mnist-service

```
# Deploying the UI / API
Change to the directory containing the web front end manifests:
```
cd $WORKING_DIR/front
```

Unlike the other steps, this manifest requires no customization. It can be applied directly:
```
kustomize build . | kubectl apply -f -
```
The service added is of type ClusterIP, meaning it can't be accessed from outside the cluster. In order to load the web UI in your web browser, you have to set up a direct connection to the cluster:

```
kubectl port-forward svc/web-ui 8080:80
```

Now in your Cloud Shell interface, press the web preview button and select "Preview on port 8080" to open the web interface in your browser:
You should now see the MNIST Web UI

Keep in mind that the web interface doesn't do much on its own, it's simply a basic HTML/JS wrapper around theTensorflow Serving component, which performs the actual predictions. To emphasize this, the web interface allows you to manually connect with the serving instance located in the cluster. It has three fields:

Model Name: mnist

* The model name associated with the Tensorflow Serving component
* You configured this when you set the modelName parameter for the mnist-deploy-gcp component
Server Address: mnist-service

* The IP address or DNS name of the server
* Because Kubernetes has an internal DNS service, you can enter the service name (https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
Port: 9000

* The port the server is listening on
* Kubeflow sets this to 9000 by default
These three fields uniquely define your model server. If you deploy multiple serving components, you should be able to switch between them using the web interface. Feel free to experiment!

# Monitoring
## Monitoring with Tensorfboard
Instructions - https://github.com/kubeflow/examples/tree/master/mnist
```
kubectl port-forward service/tensorboard-tb 8090:80
```

References: 
* https://www.qwiklabs.com (Introduction to Kubeflow on Google Kubernetes Engine) - great source to learn latest cloud architecture in virtual labs


