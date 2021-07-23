# TerraForm Task 4

## Task Requirement
1. Create 3 different workspace and create a full stack webserver on 3 different cloud.
2. Launch wordpress on GCP and RDS service using AWS

### Step by step Implementation
#### 1.  Create 3 different workspace
```
terraform workspace new workspace_name
```

- ##### Check all existing workspace
```
terraform workspace list
```
![image](https://user-images.githubusercontent.com/75135128/126815520-d497930d-644f-462d-beb8-4f81bc4cc9e9.png)

#### 2. Create 3 full stack webserver on 3 cloud providers

first make aws.tf
```
provider "aws" {
  region = var.region
  shared_credentials_file = var.creds
  profile = "default"
}
```
create EC2 instance
```
# create a resource for ec2 instance 
resource "aws_instance" "TFuser1_os1" {
ami = "ami-010aff33ed5991201"
instance_type ="t2.micro"
tags = {
  Name = "NEW OS"
 } 
}
```
Configure wbserver
```
# ssh on ec2
resource "null_resource" "test1" {
 connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = file("C:/Users/pardeep/Downloads/terraform_trial.pem")
    host     = aws_instance.webserver1.public_ip
  }
  
  #execute cmds on ec2
 provisioner "remote-exec" {
    inline = [
      "sudo yum install http -y",
      "sudo yum install php -y",
      "sudo systemctl start httpd",
      "sudo systemctl start php",
      "cd /var/www/html"
  }
}
```

Create gcp.tf
```

provider "google" {

credentials = file("/Users/testuser/Desktop/gpsvc.json")

project = "googleproject"
 region  = "us-central1"
 zone    = "us-central1-c"
}



resource "google_compute_instance" "apache_test" {
    name = "apacheserver"
    machine_type = "f1-micro"

    tags = ["http-server"]

    boot_disk {
        initialize_params {
            image = "debian-cloud/debian-9"
        }
    }

    metadata_startup_script =  file("/Users/testuser/Desktop/apache2.sh")

scheduling {
        preemptible = true
        automatic_restart = false
    }

    network_interface {
        network ="default"
        access_config {
        }
}
}
```

startup code for webserver
```
!/bin/bash
sudo apt-get update && sudo apt -y install apache2
echo '<!doctype html><html><body><h1>This text confirm the running server!</h1></body></html>' | sudo tee /var/www/html/index.html
```

write terraform code for azure

(a) provider.tf

```
provider "azurerm" {
	  version = "~> 1.4"
	  environment = "public"
}
```

(b) network.tf

```

resource "azurerm_resource_group" "network-rg" {
	  name     = "${lower(replace(var.app_name," ","-"))}-${var.environment}-rg"
	  location = var.location
	  tags = {
	    application = var.app_name
	    environment = var.environment
	  }
	}
	

	# Create the network VNET
	resource "azurerm_virtual_network" "network-vnet" {
	  name                = "${lower(replace(var.app_name," ","-"))}-${var.environment}-vnet"
	  address_space       = [var.network-vnet-cidr]
	  resource_group_name = azurerm_resource_group.network-rg.name
	  location            = azurerm_resource_group.network-rg.location
	  tags = {
	    application = var.app_name
	    environment = var.environment
	  }
	}
	

	# Create a subnet for Network
	resource "azurerm_subnet" "network-subnet" {
	  name                 = "${lower(replace(var.app_name," ","-"))}-${var.environment}-subnet"
	  address_prefix       = var.network-subnet-cidr
	  virtual_network_name = azurerm_virtual_network.network-vnet.name
	  resource_group_name  = azurerm_resource_group.network-rg.name
}
```
(c) variables.tf

```
variable "company" {
	  type        = string
	  description = "This variable defines thecompany name used to build resources"
	}
	

	# application name 
	variable "app_name" {
	  type        = string
	  description = "This variable defines the application name used to build resources"
	}
	

	# application or company environment
	variable "environment" {
	  type        = string
	  description = "This variable defines the environment to be built"
	}
	

	# azure region
	variable "location" {
	  type        = string
	  description = "Azure region where the resource group will be created"
	  default     = "north europe"
    }

variable "network-vnet-cidr" {
	  type        = string
	  description = "The CIDR of the network VNET"
	}
	

	variable "network-subnet-cidr" {
	  type        = string
	  description = "The CIDR for the network subnet"
	}
  ```
  
  (d) userdata.tf

```
  sudo apt-get update
	sudo apt-get install -y apache2
	sudo systemctl start apache2
	sudo systemctl enable apache2
	echo "<h1>Azure Linux VM with Web Server</h1>" | sudo tee /var/www/html/index.html
  ```
  
  (e) instance.tf
  
  ```
  resource "random_password" "web-vm-password" {
	  length           = 16
	  min_upper        = 2
	  min_lower        = 2
	  min_special      = 2
	  number           = true
	  special          = true
	  override_special = "!@#$%&"
	}
	

	# Generate a random vm name
	resource "random_string" "web-vm-name" {
	  length  = 8
	  upper   = false
	  number  = false
	  lower   = true
	  special = false
	}
	

	# Create Security Group to access web
	resource "azurerm_network_security_group" "web-vm-nsg" {
	  depends_on=[azurerm_resource_group.network-rg]
	

	  name                = "web-${lower(var.environment)}-${random_string.web-vm-name.result}-nsg"
	  location            = azurerm_resource_group.network-rg.location
	  resource_group_name = azurerm_resource_group.network-rg.name
	

	  security_rule {
	    name                       = "AllowWEB"
	    description                = "Allow web"
	    priority                   = 100
	    direction                  = "Inbound"
	    access                     = "Allow"
	    protocol                   = "Tcp"
	    source_port_range          = "*"
	    destination_port_range     = "80"
	    source_address_prefix      = "Internet"
	    destination_address_prefix = "*"
	  }
	

	  security_rule {
	    name                       = "AllowSSH"
	    description                = "Allow SSH"
	    priority                   = 150
	    direction                  = "Inbound"
	    access                     = "Allow"
	    protocol                   = "Tcp"
	    source_port_range          = "*"
	    destination_port_range     = "22"
	    source_address_prefix      = "Internet"
	    destination_address_prefix = "*"
	  }
	  tags = {
	    environment = var.environment
	  }
	}
	

	# Associate the web NSG with the subnet
	resource "azurerm_subnet_network_security_group_association" "web-vm-nsg-association" {
	  depends_on=[azurerm_resource_group.network-rg]
	

	  subnet_id                 = azurerm_subnet.network-subnet.id
	  network_security_group_id = azurerm_network_security_group.web-vm-nsg.id
	}
	

	# Get a Static Public IP
	resource "azurerm_public_ip" "web-vm-ip" {
	  depends_on=[azurerm_resource_group.network-rg]
	

	  name                = "web-${random_string.web-vm-name.result}-ip"
	  location            = azurerm_resource_group.network-rg.location
	  resource_group_name = azurerm_resource_group.network-rg.name
	  allocation_method   = "Static"
	  
	  tags = { 
	    environment = var.environment
	  }
	}
	

	# Create Network Card for web VM
	resource "azurerm_network_interface" "web-private-nic" {
	  depends_on=[azurerm_resource_group.network-rg]
	

	  name                = "web-${random_string.web-vm-name.result}-nic"
	  location            = azurerm_resource_group.network-rg.location
	  resource_group_name = azurerm_resource_group.network-rg.name
	  
	  ip_configuration {
	    name                          = "internal"
	    subnet_id                     = azurerm_subnet.network-subnet.id
	    private_ip_address_allocation = "Dynamic"
	    public_ip_address_id          = azurerm_public_ip.web-vm-ip.id
	  }
	

	  tags = { 
	    environment = var.environment
	  }
	}
	

	# Create Linux VM with web server
	resource "azurerm_virtual_machine" "web-vm" {
	  depends_on=[azurerm_network_interface.web-private-nic]
	

	  location              = azurerm_resource_group.network-rg.location
	  resource_group_name   = azurerm_resource_group.network-rg.name
	  name                  = "web-${random_string.web-vm-name.result}-vm"
	  network_interface_ids = [azurerm_network_interface.web-private-nic.id]
	  vm_size               = var.web_vm_size
	  license_type          = var.web_license_type
	

	  delete_os_disk_on_termination    = var.web_delete_os_disk_on_termination
	  delete_data_disks_on_termination = var.web_delete_data_disks_on_termination
	

	  storage_image_reference {
	    id        = lookup(var.web_vm_image, "id", null)
	    offer     = lookup(var.web_vm_image, "offer", null)
	    publisher = lookup(var.web_vm_image, "publisher", null)
	    sku       = lookup(var.web_vm_image, "sku", null)
	    version   = lookup(var.web_vm_image, "version", null)
	  }
	

	  storage_os_disk {
	    name              = "web-${random_string.web-vm-name.result}-disk"
	    caching           = "ReadWrite"
	    create_option     = "FromImage"
	    managed_disk_type = "Standard_LRS"
	  }
	

	  os_profile {
	    computer_name  = "web-${random_string.web-vm-name.result}-vm"
	    admin_username = var.web_admin_username
	    admin_password = random_password.web-vm-password.result
	    custom_data    = file("azure-user-data.sh")
	  }
	

	  os_profile_linux_config {
	    disable_password_authentication = false
	  }
	

	  tags = {
	    environment = var.environment
	  }
}

output "web_vm_name" {
	  description = "Virtual Machine name"
	  value       = azurerm_virtual_machine.web-vm.name
	}
	

	output "web_vm_ip_address" {
	  description = "Virtual Machine name IP Address"
	  value       = azurerm_public_ip.web-vm-ip.ip_address
	}
	

	output "web_vm_admin_username" {
	  description = "Username password for the Virtual Machine"
	  value       = azurerm_virtual_machine.web-vm.os_profile.*
	  #sensitive   = true
	}
	

	output "web_vm_admin_password" {
	  description = "Admin pass for the Virtual Machine"
	  value       = random_password.web-vm-password.result
	  #sensitive   = true
	}
  ```
  
  (f) instancevar.tf
  
  ```
  variable "web_vm_size" {
	  type        = string
	  description = "Size (SKU) of the virtual machine to create"
	}
	

	variable "web_license_type" {
	  type        = string
	  description = "Specifies the BYOL type for the virtual machine. Possible values are 'Windows_Client' and 'Windows_Server' if set"
	  default     = null
	}
	

	# Azure virtual machine storage settings #
	

	variable "web_delete_os_disk_on_termination" {
	  type        = string
	  description = "Should the OS Disk (either the Managed Disk / VHD Blob) be deleted when the Virtual Machine is destroyed?"
	  default     = "true"  # Update for your environment
	}
	

	variable "web_delete_data_disks_on_termination" {
	  description = "Should the Data Disks (either the Managed Disks / VHD Blobs) be deleted when the Virtual Machine is destroyed?"
	  type        = string
	  default     = "true" # Update for your environment
	}
	

	variable "web_vm_image" {
	  type        = map(string)
	  description = "Virtual machine source image information"
	  default     = {
	    publisher = "Canonical"
	    offer     = "UbuntuServer"
	    sku       = "18.04-LTS" 
	    version   = "latest"
	  }
	}
	

	# Azure virtual machine OS profile #
	

	variable "web_admin_username" {
	  description = "Username for Virtual Machine administrator account"
	  type        = string
	  default     = ""
	}
	

	variable "web_admin_password" {
	  description = "Password for Virtual Machine administrator account"
	  type        = string
	  default     = ""
    }
```
    
Initialize all three workspaces

```
terraform init
```
![image](https://user-images.githubusercontent.com/75135128/126817156-862842c4-7d7e-4030-9bc3-64459efee346.png)

Note: initialize in every folder you are going to use.

Apply the terraform code
```
terraform apply
```

Snipt of GCP webserver:
![image](https://user-images.githubusercontent.com/75135128/126817629-2afc09cb-08c8-41f1-ab35-cd01ca5ff3da.png)

#### 2. Launch wordpress on GCP and RDS service in AWS 

(a) Wordpress on GCP

provider.tf

```
provider "google" {
	 credentials = file("gcpCreds.json")
	 project     = var.project_id
	}
	

	//Configuring Kubernetes Provider
	provider "kubernetes" {}
	

	//Creating First VPC Network
	resource "google_compute_network" "vpc_network1" {
	  name        = "prod-wp-env"
	  description = "VPC Network for WordPress"
	  project     = var.project_id
	  auto_create_subnetworks = false
	}
	

	//Creating Subnetwork for First VPC
	resource "google_compute_subnetwork" "subnetwork1" {
	  name          = "wp-subnet"
	  ip_cidr_range = "10.2.0.0/16"
	  project       = var.project_id
	  region        = var.region1
	  network       = google_compute_network.vpc_network1.id
	

	  depends_on = [
	    google_compute_network.vpc_network1
	  ]
    }
```

vpc.tf

```
//Creating Firewall for First VPC Network
	resource "google_compute_firewall" "firewall1" {
	  name    = "wp-firewall"
	  network = google_compute_network.vpc_network1.name
	

	  allow {
	    protocol = "icmp"
	  }
	

	  allow {
	    protocol = "tcp"
	    ports    = ["80", "8080"]
	  }
	

	  source_tags = ["wp", "wordpress"]
	

	  depends_on = [
	    google_compute_network.vpc_network1
	  ]
	}
	

	//Creating Second VPC Network
	resource "google_compute_network" "vpc_network2" {
	  name        = "prod-db-env"
	  description = "VPC Network For dataBase"
	  project     = var.project_id
	  auto_create_subnetworks = false
	}
	

	//Creating Network For Second VPC
	resource "google_compute_subnetwork" "subnetwork2" {
	  name          = "db-subnet"
	  ip_cidr_range = "10.4.0.0/16"
	  project       = var.project_id
	  region        = var.region2
	  network       = google_compute_network.vpc_network2.id
	

	  depends_on = [
	    google_compute_network.vpc_network2
	  ]
	}

```
network.tf
```
//Creating Firewall for Second VPC Network
	resource "google_compute_firewall" "firewall2" {
	  name    = "db-firewall"
	  network = google_compute_network.vpc_network2.name
	

	  allow {
	    protocol = "tcp"
	    ports    = ["80", "8080", "3306"]
	  }
	

	  source_tags = ["db", "database"]
	

	  depends_on = [
	    google_compute_network.vpc_network2
	  ]
	}
	

	//VPC Network Peering1 
	resource "google_compute_network_peering" "peering1" {
	  name         = "wp-to-db"
	  network      = google_compute_network.vpc_network1.id
	  peer_network = google_compute_network.vpc_network2.id
	

	  depends_on = [
	    google_compute_network.vpc_network1,
	    google_compute_network.vpc_network2
	  ]
	}
	

	//VPC Network Peering2
	resource "google_compute_network_peering" "peering2" {
	  name         = "db-to-wp"
	  network      = google_compute_network.vpc_network2.id
	  peer_network = google_compute_network.vpc_network1.id
	

	  depends_on = [
	    google_compute_network.vpc_network1,
	    google_compute_network.vpc_network2
	  ]
    }
```

sql.tf
```
resource "google_sql_database" "sql_db" {
	  name     = var.database
	  instance = google_sql_database_instance.sqldb_Instance.name
	

	  depends_on = [
	    google_sql_database_instance.sqldb_Instance
	  ]  
	}
	

	//Creating SQL Database User
	resource "google_sql_user" "dbUser" {
	  name     = var.db_user
	  instance = google_sql_database_instance.sqldb_Instance.name
	  password = var.db_user_pass
	

	  depends_on = [
	    google_sql_database_instance.sqldb_Instance
	  ]
	}
	

	//Creating Container Cluster
	resource "google_container_cluster" "gke_cluster1" {
	  name     = "my-cluster"
	  description = "My GKE Cluster"
	  project = var.project_id
	  location = var.region1
	  network = google_compute_network.vpc_network1.name
	  subnetwork = google_compute_subnetwork.subnetwork1.name
	  remove_default_node_pool = true
	  initial_node_count       = 1
	

	  depends_on = [
	    google_compute_subnetwork.subnetwork1
	  ]
	}
	

	//Creating Node Pool For Container Cluster
	resource "google_container_node_pool" "nodepool1" {
	  name       = "my-node-pool"
	  project    = var.project_id
	  location   = var.region1
	  cluster    = google_container_cluster.gke_cluster1.name
	  node_count = 1
	

	  node_config {
	    preemptible  = true
	    machine_type = "e2-micro"
	  }
	

	  autoscaling {
	    min_node_count = 1
	    max_node_count = 3
	  }
	

	  depends_on = [
	    google_container_cluster.gke_cluster1
	  ]
	}
	

	//Set Current Project in gcloud SDK
	resource "null_resource" "set_gcloud_project" {
	  provisioner "local-exec" {
	    command = "gcloud config set project ${var.project_id}"
	  }  
	}
	

	//Configure Kubectl with Our GCP K8s Cluster
	resource "null_resource" "configure_kubectl" {
	  provisioner "local-exec" {
	    command = "gcloud container clusters get-credentials ${google_container_cluster.gke_cluster1.name} --region ${google_container_cluster.gke_cluster1.location} --project ${google_container_cluster.gke_cluster1.project}"
	  }  
	

	  depends_on = [
	    null_resource.set_gcloud_project,
	    google_container_cluster.gke_cluster1
	  ]
	}
	

	//WordPress Deployment
	resource "kubernetes_deployment" "wp-dep" {
	  metadata {
	    name   = "wp-dep"
	    labels = {
	      env     = "Production"
	    }
	  }
	

	  spec {
	    replicas = 1
	    selector {
	      match_labels = {
	        pod     = "wp"
	        env     = "Production"
	      }
	    }
	

	    template {
	      metadata {
	        labels = {
	          pod     = "wp"
	          env     = "Production"
	        }
	      }
	

	      spec {
	        container {
	          image = "wordpress"
	          name  = "wp-container"
	

	          env {
	            name  = "WORDPRESS_DB_HOST"
	            value = "${google_sql_database_instance.sqldb_Instance.ip_address.0.ip_address}"
	          }
	          env {
	            name  = "WORDPRESS_DB_USER"
	            value = var.db_user
	          }
	          env {
	            name  = "WORDPRESS_DB_PASSWORD"
	            value = var.db_user_pass
	          }
	          env{
	            name  = "WORDPRESS_DB_NAME"
	            value = var.database
	          }
	          env{
	            name  = "WORDPRESS_TABLE_PREFIX"
	            value = "wp_"
	          }
	

	          port {
	            container_port = 80
	          }
	        }
	      }
	    }
	  }
	

	  depends_on = [
	    null_resource.set_gcloud_project,
	    google_container_cluster.gke_cluster1,
	    google_container_node_pool.nodepool1,
	    null_resource.configure_kubectl
	  ]
	}
	

	//Creating LoadBalancer Service
	resource "kubernetes_service" "wpService" {
	  metadata {
	    name   = "wp-svc"
	    labels = {
	      env     = "Production" 
	    }
	  }  
	

	  spec {
	    type     = "LoadBalancer"
	    selector = {
	      pod = "${kubernetes_deployment.wp-dep.spec.0.selector.0.match_labels.pod}"
	    }
	

	    port {
	      name = "wp-port"
	      port = 80
	    }
	  }
	

	  depends_on = [
	    kubernetes_deployment.wp-dep,
	  ]
    }
```

var.tf
```
"var iable ""project_id"" {"
type = string
"default =  ""-- -Your Project ID here- - -"""
}
"//Regionl Variable variable ""regionl"" {"
type = string
"description = ""Regionl Name"" default = ""us-centrall"""
}
"//Region2 Variable variable ""region2"" {"
type = string
"description = ""Region2 Name"" default = ""asia-southl"""
}
"//SQL Database Root Password variable ""root_pass"" {"
type = string
"description = ""Root Password For SQL Database"" default = ""toor"""
}
"//SQL Database Name variable ""database"" {"
type = string
"description = ""SQL Database Name"""
"default = ""wpdb """
}
"//SQL Database User variable ""db_user"" {"
type = string
"description = ""SQL Database User Name"" default = ""wpuser"""
}
"//SQL Databse User Password variable ""db_user_pass"" {"
type = string
"description = ""Passowrd for SQL Database User"" default = ""wppass"""

```

(b) lets deploy RDS on AWS
```
resource "aws_db_instance" "default" {
  allocated_storage    = 10
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t3.micro"
  name                 = "mydb"
  username             = "pardeep"
  password             = "asdf1234"
  parameter_group_name = "default.mysql5.7"
  skip_final_snapshot  = true
}
```

Check on our GCP console after deploy:
![image](https://user-images.githubusercontent.com/75135128/126819502-b4527f83-6299-4fb8-a8b7-c9efe2d48964.png)

THis is WELCOME to WORDPRESS Screen. It means our infra is running
