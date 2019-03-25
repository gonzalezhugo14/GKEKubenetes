# GKEKubenetes
Kubernetes demo 
Organizing into Modules
We’ll start by setting up this directory structure, and files referenced will use this:

.
├── gke
│   ├── cluster.tf
│   ├── gcp.tf
│   └── variables.tf
├── k8s
│   ├── k8s.tf
│   ├── pods.tf
│   ├── services.tf
│   └── variables.tf
└─── main.tf
We can create this structure and empty files:

mkdir terraform-gke
cd terraform-gke
mkdir gke k8s
touch main.tf
for f in cluster gcp variables; do touch gke/$f.tf; done
for f in k8s pods services variables; do touch k8s/$f.tf; done
Our top level, main.tf will reference two modules that we’ll create later. One module will create the GKE cluster, and the other module will use information from GKE to deploy software into the Kubernetes cluster.


main.tf
Cluster Specification
This is the module that will create a Kubernetes cluster on Google Cloud using GKE resource.

This first start by specifying all the variables this module will use in gke/variables.tf file:


variables.tf
We’ll need to specify a provider, which is Google Cloud in gke/gcp.tf file:


gcp.tf
With the variables specified and provider specified, we can now create our Kubernetes infrastructure. In Google Cloud this is one resource, but this encapsulates many components (managed instance group and template, persistence store, GCE instances for worker nodes, GKE master). This in done in gke/cluster.tf file:


cluster.tf
This creates a 3 worker node cluster. The output variables will be used later when we deploy applications. They are marked sensitive to avoid printing out to standard output.

Guestbook Application Specification
Kubernetes code repository has an example application called guestbook that uses Redis cluster to store information. This module is divided into four parts:

Variables used in this module
Kubernetes provider to connect to Kubernetes API
Pods using Replication Controller
Services creates permanent end point and connecting them to internal IP addresses as pods are added or removed.
The variables we’ll use are defined in variables.tf file:


variables.tf
Our Kubernetes provider is in the k8s.tf file:


k8s.tf
And now we create our minimum unit of deployment, the Kubernetes pods using Kubernetes Replication Controller in pods.tf file. These will be 1 redis master pod, 2 redis slave pods, and 1 frontend pod. The images for these components are available from Google’s Container Registry.


pods.tf
This will create the pods that we can now use to deploy services into them. We’ll create the services.tf for the services we wish to deploy (redis master, redis slave, frontend). One note about the frontend, as it uses the type LoadBalancer, this will create a google load balancer outside of the cluster to send traffic one of three pods.


services.tf
Launch the Application
Before we start, we need to initialize some variables that the GCP provider requires, which is the target project and the desired region to create the cluster. We’ll use our default project configured with gcloud:

export TF_VAR_project="$(gcloud config list \
  --format 'value(core.project)'
)"
export TF_VAR_region="us-east1"
Now we’ll need to specify the administrative account and a random password for the cluster:

export TF_VAR_user="admin"
export TF_VAR_password="m8XBWrg2zt8R8JoH"
With these setup, we can initialize our environment, which includes both downloading plugins required google cloud provider and kubernetes provider, as well as references to our modules.

terraform init
Now we can see what we want to create and then create it:

terraform plan
terraform apply
After some time (10 to 20 minutes) we can test out our application. Run this to see the end points:

kubectl get service
Final Thoughts
This is an easy way to quickly test clusters, pods, services, and other components. Currently, Terraform only supports deploying Pods and ReplicationControllers as deployable units.
