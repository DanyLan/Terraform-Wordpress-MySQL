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

terraform init

sudo terraform validate ( do upgrade below if you run into issues )

terraform 0.12upgrade

terraform apply
