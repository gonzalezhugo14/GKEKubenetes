# Project Title

Terraform + GKE + Kubenetes

## Getting Started

This article illustrates spinning up a Kubernetes cluster on Google Cloud using GKE, and then deploying the guestbook sample application (AngularJS, PHP, Redis), which google has made available in their container registry.

### Prerequisites

Organizing into Modules
We’ll start by setting up this directory structure, and files referenced will use this:

```
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
```
We can create this structure and empty files:

```
mkdir terraform-gke
cd terraform-gke
mkdir gke k8s
touch main.tf
for f in cluster gcp variables; do touch gke/$f.tf; done
for f in k8s pods services variables; do touch k8s/$f.tf; done
```



### Installing

Our top level, main.tf will reference two modules that we’ll create later. One module will create the GKE cluster, and the other module will use information from GKE to deploy software into the Kubernetes cluster.


```
#####################################################################
# Variables
#####################################################################
variable "username" {
  default = "admin"
}
variable "password" {}
variable "project" {}
variable "region" {}

#####################################################################
# Modules
#####################################################################
module "gke" {
  source   = "./gke"
  project  = "${var.project}"
  region   = "${var.region}"
  username = "${var.username}"
  password = "${var.password}"
}

module "k8s" {
  source   = "./k8s"
  host     = "${module.gke.host}"
  username = "${var.username}"
  password = "${var.password}"

  client_certificate     = "${module.gke.client_certificate}"
  client_key             = "${module.gke.client_key}"
  cluster_ca_certificate = "${module.gke.cluster_ca_certificate}"
}
```

Cluster Specification
This is the module that will create a Kubernetes cluster on Google Cloud using GKE resource.

This first start by specifying all the variables this module will use in gke/variables.tf file:

```
#####################################################################
# Variables
#####################################################################
variable "project" {}
variable "region" {}
variable "username" {
  default = "admin"
}
variable "password" {}
```

We’ll need to specify a provider, which is Google Cloud in gke/gcp.tf file:

```
#####################################################################
# Google Cloud Platform
#####################################################################
provider "google" {
  project = "${var.project}"
  region  = "${var.region}"
}
```

With the variables specified and provider specified, we can now create our Kubernetes infrastructure. In Google Cloud this is one resource, but this encapsulates many components (managed instance group and template, persistence store, GCE instances for worker nodes, GKE master). This in done in gke/cluster.tf file:


```
#####################################################################
# GKE Cluster
#####################################################################
resource "google_container_cluster" "guestbook" {
  name               = "guestbook"
  zone               = "us-east1-b"
  initial_node_count = 3

  addons_config {
    network_policy_config {
      disabled = true
    }
  }

  master_auth {
    username = "${var.username}"
    password = "${var.password}"
  }

  node_config {
    oauth_scopes = [
      "https://www.googleapis.com/auth/devstorage.read_only",
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
      "https://www.googleapis.com/auth/service.management.readonly",
      "https://www.googleapis.com/auth/servicecontrol",
      "https://www.googleapis.com/auth/trace.append",
      "https://www.googleapis.com/auth/compute",
    ]
  }
}

#####################################################################
# Output for K8S
#####################################################################
output "client_certificate" {
  value     = "${google_container_cluster.guestbook.master_auth.0.client_certificate}"
  sensitive = true
}

output "client_key" {
  value     = "${google_container_cluster.guestbook.master_auth.0.client_key}"
  sensitive = true
}

output "cluster_ca_certificate" {
  value     = "${google_container_cluster.guestbook.master_auth.0.cluster_ca_certificate}"
  sensitive = true
}

output "host" {
  value     = "${google_container_cluster.guestbook.endpoint}"
  sensitive = true
}
```

This creates a 3 worker node cluster. The output variables will be used later when we deploy applications. They are marked sensitive to avoid printing out to standard output



End with an example of getting some data out of the system or using it for a little demo

## Guestbook Application Specification

Kubernetes code repository has an example application called guestbook that uses Redis cluster to store information. This module is divided into four parts:

Variables used in this module
Kubernetes provider to connect to Kubernetes API
Pods using Replication Controller
Services creates permanent end point and connecting them to internal IP addresses as pods are added or removed.
The variables we’ll use are defined in variables.tf file:

```
#####################################################################
# Variables
#####################################################################
variable "username" {
  default = "admin"
}
variable "password" {}
variable "host" {}
variable client_certificate {}
variable client_key {}
variable cluster_ca_certificate {}
```
Our Kubernetes provider is in the k8s.tf file:
```
provider "kubernetes" {
  host     = "${var.host}"
  username = "${var.username}"
  password = "${var.password}"

  client_certificate     = "${base64decode(var.client_certificate)}"
  client_key             = "${base64decode(var.client_key)}"
  cluster_ca_certificate = "${base64decode(var.cluster_ca_certificate)}"
}
```
And now we create our minimum unit of deployment, the Kubernetes pods using Kubernetes Replication Controller in pods.tf file. These will be 1 redis master pod, 2 redis slave pods, and 1 frontend pod. The images for these components are available from Google’s Container Registry.

```
resource "kubernetes_replication_controller" "redis-master" {
  metadata {
    name = "redis-master"

    labels {
      app  = "redis"
      role = "master"
      tier = "backend"
    }
  }

  spec {
    replicas = 1

    selector = {
      app  = "redis"
      role = "master"
      tier = "backend"
    }

    template {
      container {
        image = "k8s.gcr.io/redis:e2e"
        name  = "master"

        port {
          container_port = 6379
        }

        resources {
          requests {
            cpu    = "100m"
            memory = "100Mi"
          }
        }
      }
    }
  }
}

resource "kubernetes_replication_controller" "redis-slave" {
  metadata {
    name = "redis-slave"

    labels {
      app  = "redis"
      role = "slave"
      tier = "backend"
    }
  }

  spec {
    replicas = 2

    selector = {
      app  = "redis"
      role = "slave"
      tier = "backend"
    }

    template {
      container {
        image = "gcr.io/google_samples/gb-redisslave:v1"
        name  = "slave"

        port {
          container_port = 6379
        }

        env {
          name  = "GET_HOSTS_FROM"
          value = "dns"
        }

        resources {
          requests {
            cpu    = "100m"
            memory = "100Mi"
          }
        }
      }
    }
  }
}

resource "kubernetes_replication_controller" "frontend" {
  metadata {
    name = "frontend"

    labels {
      app  = "guestbook"
      tier = "frontend"
    }
  }

  spec {
    replicas = 3

    selector = {
      app  = "guestbook"
      tier = "frontend"
    }

    template {
      container {
        image = "gcr.io/google-samples/gb-frontend:v4"
        name  = "php-redis"

        port {
          container_port = 80
        }

        env {
          name  = "GET_HOSTS_FROM"
          value = "dns"
        }

        resources {
          requests {
            cpu    = "100m"
            memory = "100Mi"
          }
        }
      }
    }
  }
}
```
This will create the pods that we can now use to deploy services into them. We’ll create the services.tf for the services we wish to deploy (redis master, redis slave, frontend). One note about the frontend, as it uses the type LoadBalancer, this will create a google load balancer outside of the cluster to send traffic one of three pods.


```
resource "kubernetes_service" "redis-master" {
  metadata {
    name = "redis-master"

    labels {
      app  = "redis"
      role = "master"
      tier = "backend"
    }
  }

  spec {
    selector {
      app  = "redis"
      role = "master"
      tier = "backend"
    }

    port {
      port        = 6379
      target_port = 6379
    }
  }
}

resource "kubernetes_service" "redis-slave" {
  metadata {
    name = "redis-slave"

    labels {
      app  = "redis"
      role = "slave"
      tier = "backend"
    }
  }

  spec {
    selector {
      app  = "redis"
      role = "slave"
      tier = "backend"
    }

    port {
      port        = 6379
      target_port = 6379
    }
  }
}

resource "kubernetes_service" "frontend" {
  metadata {
    name = "frontend"

    labels {
      app  = "guestbook"
      tier = "frontend"
    }
  }

  spec {
    selector {
      app  = "guestbook"
      tier = "frontend"
    }

    type = "LoadBalancer"

    port {
      port = 80
    }
  }
}
```




### Launch the Application

Before we start, we need to initialize some variables that the GCP provider requires, which is the target project and the desired region to create the cluster. We’ll use our default project configured with gcloud:

```
export TF_VAR_project="$(gcloud config list \  --format 'value(core.project)')"
export TF_VAR_region="us-east1"
```
Now we’ll need to specify the administrative account and a random password for the cluster:

```
export TF_VAR_user="admin"
exportTF_VAR_password="Clust3rK8s4D3monstr4ti0n"
```


### Terraforming

With these setup, we can initialize our environment, which includes both downloading plugins required google cloud provider and kubernetes provider, as well as references to our modules.

```
terraform init
```
Now we can see what we want to create and then create it

```
terraform plan
terraform apply
```

## Testing the Application

After some time (3 to 5 minutes) we can test out our application. Run this to see the end points:
```
kubectl get service
```
Review the ooutput External IP Address and Open with your prefered Browser

## Finishing the demo

Destroy the evidence from your cloud provider

```
terraform destroy
```
Final Thoughts
This is an easy way to quickly test clusters, pods, services, and other components. Currently, Terraform only supports deploying Pods and ReplicationControllers as deployable units.

## Built With

* [Terraform]( https://www.terraform.io/downloads.html) 
* [Google Cloud SDK]( https://cloud.google.com/sdk/) 
* [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 







