# Using Terraform to create a Wordpress-MySQL deployment in GKE

Go to [cloudshell](https://cloud.google.com/shell/docs/quickstart) and create a folder name terraform

    mkdir terraform

# Create the `variable.tf` file

    nano variable.tf

<pre>variable "app_label" {
  default = "wordpress"
}

variable "mysql_tier" {
  default = "mysql"
}

variable "wordpress_tier" {
  default = "frontend"
}

variable "wordpress_version" {
  default = "4.7.3"
}

variable "mysql_password" {
  default = "P4sSw0rd0!"
}
</pre>

# Create the `main.tf` file

    nano main.tf
    
<pre>resource "kubernetes_service" "mysql_service" {
  metadata {
    name = "wordpress-mysql"
    labels = {
      app = "${var.app_label}"
    }
  }
  spec {
    selector {
      app  = "${var.app_label}"
      tier = "${var.mysql_tier}"
    }
    port {
      port = "3306"
    }

    type = "NodePort"
  }
}

resource "kubernetes_deployment" "mysql_deployment" {
  metadata {
    name   = "wordpress-mysql"
    labels = {
      app = "${var.app_label}"
    }
  }

  spec {
    replicas = "1"

    selector {
      match_labels {
        app  = "${var.app_label}"
        tier = "${var.mysql_tier}"
      }
    }

    template {
      metadata {
        labels {
          app  = "${var.app_label}"
          tier = "${var.mysql_tier}"
        }
      }

      spec {
        container {
          name  = "mysql"
          image = "mysql:5.7"

          env {
            name  = "MYSQL_ROOT_PASSWORD"
            value = "${var.mysql_password}"
          }

          port {
            container_port = "3306"
            name           = "mysql"
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "wordpress_service" {
  metadata {
    name   = "wordpress"
    labels = {
      app = "${var.app_label}"
    }
  }
  spec {
    selector {
      app  = "${var.app_label}"
      tier = "${var.wordpress_tier}"
    }

    port {
      port        = "80"
      target_port = "80"
    }

    type = "NodePort"
  }
}

resource "kubernetes_deployment" "wordpress_deployment" {
  metadata {
    name = "wordpress"
  }

  spec {
    replicas = "1"

    selector {
      match_labels {
        app  = "${var.app_label}"
        tier = "${var.wordpress_tier}"
      }
    }

    template {
      metadata {
        labels {
          app  = "${var.app_label}"
          tier = "${var.wordpress_tier}"
        }
      }

      spec {
        container {
          name  = "wordpress"
          image = "wordpress:${var.wordpress_version}-apache"

          env {
            name = "WORDPRESS_DB_HOST"
            value = "wordpress-mysql"
          }

          env {
            name  = "WORDPRESS_DB_PASSWORD"
            value = "${var.mysql_password}"
          }

          port {
            container_port = "80"
            name           = "wordpress"
          }
        }
      }
    }
  }
}
</pre>

Now connect to your cluster and run the following commands

`terraform init`

`sudo terraform validate` (Upgrade Terraform using `terraform 0.12upgrade` if needed)

`terraform apply`

# Wordpress access

In order to check whether your worpress site is working, you will need to allow the TCP traffic on the node port that was assigned.

Get nodeport this way

    kubectl get svc
    NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
    wordpress         NodePort    10.0.15.207   <none>        80:32015/TCP     23m
    
Get the node where your wordpress pod is sitting this way

    kubectl get pods -o wide
    NAME                               READY   STATUS    RESTARTS   AGE   IP          NODE                                       
    wordpress-mysql-77487d7bb6-4wsp7   1/1     Running   0          26m   10.60.1.8   gke-terraform-default-pool-0490f0a7-0k5k   <none>
   
Get the external IP address of that particular node

    kubectl get nodes -o wide
    NAME                                       STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP      OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
    gke-terraform-default-pool-0490f0a7-0k5k   Ready    <none>   7d    v1.12.8-gke.10   10.128.0.25   34.68.41.63      Container-Optimized OS from Google   4.14.127+        docker://17.3.2
    gke-terraform-default-pool-0490f0a7-bvx5   Ready    <none>   7d    v1.12.8-gke.10   10.128.0.21   35.193.227.243   Container-Optimized OS from Google   4.14.127+        docker://17.3.2
    gke-terraform-default-pool-0490f0a7-wktw   Ready    <none>   7d    v1.12.8-gke.10   10.128.0.24   35.225.13.119    Container-Optimized OS from Google   4.14.127+        docker://17.3.2

Create a firewall rule to allow TCP traffic on the nodeport:

    gcloud compute firewall-rules create test-node-port --allow tcp:32015
    
On browser test this way

<NODE_IP_ADDRESS>:[Node_Port]

34.68.41.63:32015

![](https://github.com/DanyLan/Terraform-Wordpress-MySQL/blob/master/wordpress.png)

# MySQL access

From 

    kubectl get svc
    NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
    wordpress-mysql   NodePort    10.0.6.233    <none>        3306:32567/TCP   23m
    
At this present point, the mysqp pod can be accessed two ways, either through cluster IP or nodeport. A third way will also be showedcase below

# Through cluster IP [pod to pod]

Get name of wordpress pod

    kubectl get pods
    NAME                               READY   STATUS    RESTARTS   AGE
    wordpress-76c79667dc-24zz4         1/1     Running   0          93m
    wordpress-mysql-77487d7bb6-4wsp7   1/1     Running   0          93m

Exec into the wordpress pod

    kubectl exec -it wordpress-76c79667dc-24zz4 -- sh

Install the MySQL client from the package manager:

    apt-get update
    apt-get install mysql-client

From variable.tf file, password is `P4sSw0rd0!`

    mysql --host=[cluster_ip] --user=root --password
    mysql --host=10.0.6.233 --user=root --password

If successful you will see

    mysql>
    
