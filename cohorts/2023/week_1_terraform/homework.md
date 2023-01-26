# Week 1.2 Homework | Preparing the environment in GCP with Terraform

In your VM on GCP install Terraform. Copy the files from the course repo
[here](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/week_1_basics_n_setup/1_terraform_gcp/terraform) to your VM.

Modify the files as necessary to create a GCP Bucket and Big Query Dataset.


# Setup GCP (Google Cloud Platform) account 


Get your account ---> [https://cloud.google.com/](https://cloud.google.com/)

- Then go to ---> https://console.cloud.google.com

- Create new project, name "xxx(your project name)xxx"

- Select project ID, click create

- Select project you want to work on, 
  - then Go to IAM & Admin 
  - server account, create service account name "xxx(your service account name)xxx"
  - done, grant this service as basic-viewer (for this course)
  - grant users access to this. (No need for now. This option is useful in case you work with team. This can grant many users to work on the same account), click done

See, there's no key ID 
  - click 3 dots on the right on that account
  - manage keys 
  - add keys, select JSON, click create
  - download box will pop up, click download into your directory

# Setup Google Cloud CLI

- Go to ---> https://cloud.google.com/sdk/docs/install

- Download & install gcloud on your PATH ```./google-cloud-sdk/install.sh``` 
- Follow the instruction until complete (see more detail for installation: https://www.codingforentrepreneurs.com/blog/google-cloud-cli-and-sdk-setup/)
- After installation, you **NEED to restart terminal** for it to take effect
- So open new Termimal window, then check if it's installed by running ```gcloud -v```

Once installed, go to the next step

## Authenticate your local CLI with GCP

On Terminal, run

```
GOOGLE_APPLICATION_CREDENTIALS=**/path/to/credentials.json** (change this using you path)
```

Launch gcloud to authenticate on your gcloud account, run
```
gcloud auth application-default login
```

Login to your account and allow access to gcloud cli

You are now authenticated with the gcloud CLI!

# Create infrastructure for your project with Terraform
## Setup Access

1. Go to Dashboard on your gcloud console --> https://console.cloud.google.com/home/dashboard

2. On Navigating menu select --> IAM & Admin

3. Select the key account that we've just created, select role:
		
    1) Basic Viewer 
		
    2) Storage Admin (to create google cloud storage)
		
    3) Storage Object Admin
		
    4) BigQuery Admin


Then you need to enable APIs:

Go to 
    
---> https://console.cloud.google.com/apis/library/iam.googleapis.com 

---> https://console.cloud.google.com/apis/library/iamcredentials.googleapis.com

Enable both of them


## Setup Terraform


Go download & install Terraform ---> https://developer.hashicorp.com/terraform/downloads


Once installed, check
```
terraform --version
```


Learn more about Terraform here https://developer.hashicorp.com/terraform/tutorials/gcp-get-started



On your working directory, create "terraform" directory and then

Create 3 files: 

1) ```.terraform-version``` , write the version of your terraform in the file e.g. "1.3.7"
	
2) ```main.tf``` , get the file --> [here](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/week_1_basics_n_setup/1_terraform_gcp/terraform)
    
    Example: see note from other students
    
    - https://github.com/ziritrion/dataeng-zoomcamp/blob/main/1_intro/terraform/main.tf
    
    - https://wind-lobe-f8d.notion.site/Week-1-Docker-Postgres-GCP-9895a14748724ea29bd687155e46fde4
	
3) ```variable.tf``` , adjust variables to fit your environment, get the file --> [here](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/week_1_basics_n_setup/1_terraform_gcp/terraform)



Once you've created ```.terraform-version``` , ```main.tf``` , ```variable.tf```, go to your Terminal 

On Terminal prompt (you should be inside "terraform" directory), run

```
terraform init
```


You will then see the folder ".terraform" in the terraform directory

Next run

```
terraform plan
```

Enter project ID: "xxx-xx(your project ID)xxx-xxx" (Or it won't ask in case you setup "your project ID" in ```variable.tf``` already)


Then run

```
terraform apply
```


Enter project ID: "xxx-xx(your project ID)xxx-xxx"  again

Response "yes"


You're on set!



Once you've done working on your session and want to remove all stack/resources from cloud


Run

```
terraform destroy
```

Response yes




### Note: I share my code (below) for main.tf and variable.tf on this page as part of the homework

#### variable.tf

```
locals {
  data_lake_bucket = "dezoom-jan2023"
}

variable "project" {
  description = "Your GCP Project ID"
  default = "xxx[My Project ID]xxx"
}

variable "region" {
  description = "Region for GCP resources. Choose as per your location: https://cloud.google.com/about/locations"
  default = "europe-west6"
  type = string
}

variable "storage_class" {
  description = "Storage class type for your bucket. Check official docs for more info."
  default = "STANDARD"
}

variable "BQ_DATASET" {
  description = "BigQuery Dataset that raw data (from GCS) will be written to"
  type = string
  default = "ny_taxi_data"
}

variable "TABLE_NAME" {
  description = "BigQuery Table"
  type = string
  default = "ny_taxi"
}
```



#### main.tf


``` 
terraform {
  required_version = ">= 1.0"
  backend "local" {}  # Can change from "local" to "gcs" (for google) or "s3" (for aws), if you would like to preserve your tf-state online
  required_providers {
    google = {
      source  = "hashicorp/google"
    }
  }
}

provider "google" {
  project = var.project
  region = var.region
  // credentials = file(var.credentials)  # Use this if you do not want to set env-var GOOGLE_APPLICATION_CREDENTIALS
}

# Data Lake Bucket
# Ref: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/storage_bucket
resource "google_storage_bucket" "data-lake-bucket" {
  name          = "${local.data_lake_bucket}_${var.project}" # Concatenating DL bucket & Project name for unique naming
  location      = var.region

  # Optional, but recommended settings:
  storage_class = var.storage_class
  uniform_bucket_level_access = true

  versioning {
    enabled     = true
  }

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 30  // days
    }
  }

  force_destroy = true
}

# DWH
# Ref: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/bigquery_dataset
resource "google_bigquery_dataset" "dataset" {
  dataset_id = var.BQ_DATASET
  project    = var.project
  location   = var.region
}
```

