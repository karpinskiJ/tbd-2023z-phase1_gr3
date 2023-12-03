IMPORTANT ❗ ❗ ❗ Please remember to destroy all the resources after each work session. You can recreate infrastructure by creating new PR and merging it to master.
  
![img.png](/doc/figures/destroy.png)


1. Authors:

   ***group nr 3***
   - Mateusz Brawański
   - Jakub Karpiński
   - Krzysztof Miśków 
   - ***https://github.com/karpinski-j/tbd-2023z-phase1***
   
2. Fork https://github.com/bdg-tbd/tbd-2023z-phase1 and follow all steps in README.md.

3. Select your project and set budget alerts on 5%, 25%, 50%, 80% of 50$ (in cloud console -> billing -> budget & alerts -> create buget; unclick discounts and promotions&others while creating budget).

  ![img.png](/doc/figures/discounts.png)

4. From avaialble Github Actions select and run destroy on main branch.

5. Create new git branch and add two resources in ```/modules/data-pipeline/main.tf```:
    1. resource "google_storage_bucket" "tbd-data-bucket" -> the bucket to store data. Set the following properties:
        * project  // look for variable in variables.tf
        * name  // look for variable in variables.tf
        * location // look for variable in variables.tf
        * uniform_bucket_level_access = false #tfsec:ignore:google-storage-enable-ubla
        * force_destroy               = true
        * public_access_prevention    = "enforced"
        * if checkcov returns error, add other properties if needed
       
    2. resource "google_storage_bucket_iam_member" "tbd-data-bucket-iam-editor" -> assign role storage.objectUser to data service account. Set the following properties:
        * bucket // refere to bucket name from tbd-data-bucket
        * role   // follow the instruction above
        * member = "serviceAccount:${var.data_service_account}"

    ***https://github.com/karpinskiJ/tbd-2023z-phase1_gr3/commit/17eda15e5f8865a5ecacaadef131f5b0c0484a91***
   
```
resource "google_storage_bucket" "tbd-data-bucket" {
  project                     = var.project_name
  name                        = var.data_bucket_name
  location                    = var.region
  uniform_bucket_level_access = false #tfsec:ignore:google-storage-enable-ubla
  public_access_prevention    = "enforced"
  force_destroy               = true
  
  #checkov:skip=CKV_GCP_62: "Bucket should log access"
  #checkov:skip=CKV_GCP_29: "Ensure that Cloud Storage buckets have uniform bucket-level access enabled"
  #checkov:skip=CKV_GCP_78: "Ensure Cloud storage has versioning enabled"
}



resource "google_storage_bucket_iam_member" "tbd-data-bucket-iam-editor" {
  bucket = google_storage_bucket.tbd-data-bucket.name
  role   = "roles/storage.objectUser"
  member = "serviceAccount:${var.data_service_account}"
}
```

 



6. Analyze terraform code. Play with terraform plan, terraform graph to investigate different modules.

    ***describe one selected module and put the output of terraform graph for this module here***
   
7. Reach YARN UI
   ![alt text](/doc/figures/yarn_ui.png)
   
   
8. Draw an architecture diagram (e.g. in draw.io) that includes:
   1. VPC topology with service assignment to subnets
    ![alt text](/doc/figures/vpc_topology.png)
    2. Description of the components of service accounts
       - Composer SA - Manages resources related to Cloud Composer.
       - Terraform SA - Allows management of a Google Cloud service account.
       - IaC - Manages GitHub account.
    3. List of buckets for disposal
       - tbd-2023z-1000-code
       - tbd-2023z-1000-conf
       - tbd-2023z-1000-data
       - tbd-2023z-1000-state
    4. Description of network communication (ports, why it is necessary to specify the host for the driver) of Apache Spark running from Vertex AI Workbech
        - Master - tbd-cluster-m 10.10.10.4
        - Worker1 - tbd-cluster-w-0 10.10.10.3
        - Worker2 - tbd-cluster-w-1 10.10.10.2
    Specifying the host for the driver is necessary in Apache Spark, because the main function is executed on driver,
    which is also responsible for dividing tasks to the workers. The workers have to know driver's address in order to
    deliver results of tasks, which are then summed up by master.
  
    

9. Add costs by entering the expected consumption into Infracost

   Considering the current usage in the project, the following values were chosen:

   ![infracost inputs](/doc/figures/2023z/09-infracost-input.png)

   After punching them into Infracost, the following result was obtained:

   ![infracost tally](/doc/figures/2023z/09-infracost-breakdopw.png)

10. Some resources are not supported by infracost yet. Estimate manually total costs of infrastructure based on pricing costs for region used in the project. Include costs of cloud composer, dataproc and AI vertex workbanch and them to infracost estimation.

    ![composer resources](/doc/figures/2023z/10-costs-composer.png)
    ![vertex AI resources](/doc/figures/2023z/10-costs-vertexai.png)
    ![dataproc resources](/doc/figures/2023z/10-costs-dataproc.png)
    ![dataproc disks resources](/doc/figures/2023z/10-costs-dataproc-disks.png)
    ![dataproc CE resources](/doc/figures/2023z/10-costs-dataproc-ce.png)

    Data sourced from [GCP Cost Calculator](https://cloud.google.com/compute/all-pricing) with region set to `eu-west1`.

    Dataproc costs are per hour, the rest are per 4.345 hours (1h/1w/1m).

    The adjusted costs are:

    - composer: `2.52 / 4.345 = ~0.58`
    - vertex AI: `0.37 / 4.345 = ~0.09`
    - dataproc: `0.06`
    - dataproc disks: `0.02`
    - dataproc CE: `0.11 + 0.23 = 0.34`

    Which comes out to approximately 1.09 USD per hour. This assumes normal worker nodes for dataproc.

    An hour is about as much as we expect to use.

    Adding the Infracost estimate of $0.23, we get estimated $1.31 per month.

    This is not very far off the costs incurred so far throughout the project:

    ![billing cost](/doc/figures/2023z/10-costs-actual.png)

    It is required that dataproc has at least one normal worker node, and all other nodes can be spot nodes. To this end, it is possible to optimize dataproc costs by changing all but one node types to spot, in our case making it 1 normal 1 spot worker, reducing the hourly cost from $0.34 to $0.27. The master node is always a normal node, and this cannot be optimized by means other than using a cheaper node type, e.g. a e2-highcpu which is cheaper, but comes with less RAM (2G instead of 8G).

    The overall cost optimization strategy would be to minimize runtime of all components or optimizing the amount of allocated storage to dataproc and composer instances. Further option is to automatically destroy all compute resources once a training session is over.
    
11. Create a BigQuery dataset and an external table
    
    ***place the code and output here***
   
    ***why does ORC not require a table schema?***
  
12. Start an interactive session from Vertex AI workbench (steps 7-9 in README):

    ***place the screenshot of notebook here***
   
13. Find and correct the error in spark-job.py
    The airflow dag has failed because of the error in spark-joby.py script - the target location for data pointed to
    bucket which did not exist. Bucket name has been changed as shown below. In result airflow task succeeded and data
    has been saved to tbd-2023z-1000-data bucket at tbd-2023z-1000-data/data/shakespeare.
   ```
# change to your data bucket
DATA_BUCKET = "gs://tbd-2023z-1000-data/data/shakespeare/"
spark = SparkSession.builder.appName('Shakespeare WordCount').getOrCreate()
```

![saved_shakespeare](/doc/figures/saved_shakespeare.png/)

![saved_shakespeare](/doc/figures/airflow_success.png/)


14. Additional tasks using Terraform:

    1. Add support for arbitrary machine types and worker nodes for a Dataproc cluster and JupyterLab instance

    Modified files:
    - [modules/dataproc/main.tf](modules/dataproc/main.tf)
    - [modules/dataproc/variables.tf](modules/dataproc/variables.tf)
    - [modules/vertex-ai-workbench/main.tf](modules/vertex-ai-workbench/main.tf)
    - [modules/vertex-ai-workbench/variables.tf](modules/vertex-ai-workbench/variables.tf)
    - [main.tf](main.tf)

    Modified code:

    ```diff
    diff --git a/modules/dataproc/main.tf b/modules/dataproc/main.tf
    index b46a162..6834873 100644
    --- a/modules/dataproc/main.tf
    +++ b/modules/dataproc/main.tf
    @@ -33,7 +33,7 @@ resource "google_dataproc_cluster" "tbd-dataproc-cluster" {
    
         master_config {
           num_instances = 1
    -      machine_type  = var.machine_type
    +      machine_type  = var.master_machine_type
           disk_config {
             boot_disk_type    = "pd-standard"
             boot_disk_size_gb = 100
    @@ -42,7 +42,7 @@ resource "google_dataproc_cluster" "tbd-dataproc-cluster" {
    
         worker_config {
           num_instances = 1
    -      machine_type  = var.machine_type
    +      machine_type  = var.worker_machine_type
           disk_config {
             boot_disk_type    = "pd-standard"
             boot_disk_size_gb = 100
    diff --git a/modules/dataproc/variables.tf b/modules/dataproc/variables.tf
    index 61eadd1..ac2367c 100644
    --- a/modules/dataproc/variables.tf
    +++ b/modules/dataproc/variables.tf
    @@ -14,10 +14,16 @@ variable "subnet" {
       description = "VPC subnet used for deployment"
     }
    
    -variable "machine_type" {
    +variable "master_machine_type" {
       type        = string
       default     = "e2-medium"
    -  description = "Machine type to use for both worker and master nodes"
    +  description = "Machine type to use for master nodes"
    +}
    +
    +variable "worker_machine_type" {
    +  type        = string
    +  default     = "e2-medium"
    +  description = "Machine type to use for worker nodes"
     }
    
     variable "image_version" {
    diff --git a/modules/vertex-ai-workbench/main.tf b/modules/vertex-ai-workbench/main.tf
    index 019bad3..cbb5f0d 100644
    --- a/modules/vertex-ai-workbench/main.tf
    +++ b/modules/vertex-ai-workbench/main.tf
    @@ -50,7 +50,7 @@ resource "google_notebooks_instance" "tbd_notebook" {
       #checkov:skip=CKV2_GCP_18: "Ensure GCP network defines a firewall and does not use the default     firewall"
       depends_on   = [google_project_service.notebooks]
       location     = local.zone
    -  machine_type = "e2-standard-2"
    +  machine_type = var.ai_notebook_instance_type
       name         = "${var.project_name}-notebook"
       container_image {
         repository = var.ai_notebook_image_repository
    diff --git a/modules/vertex-ai-workbench/variables.tf b/modules/vertex-ai-workbench/variables.tf
    index df21f7b..77e8929 100644
    --- a/modules/vertex-ai-workbench/variables.tf
    +++ b/modules/vertex-ai-workbench/variables.tf
    @@ -32,4 +32,10 @@ variable "ai_notebook_image_repository" {
     variable "ai_notebook_image_tag" {
       type    = string
       default = "latest"
    +}
    +
    +variable "ai_notebook_instance_type" {
    +  type        = string
    +  default     = "e2-standard-2"
    +  description = "Machine type to use for notebook instance"
     }
    \ No newline at end of file
    diff --git a/main.tf b/main.tf
    index 9ca729b..05c64de 100644
    --- a/main.tf
    +++ b/main.tf
    @@ -32,12 +32,13 @@ module "jupyter_docker_image" {
     }
    
     module "vertex_ai_workbench" {
    -  depends_on   = [module.jupyter_docker_image, module.vpc]
    -  source       = "./modules/vertex-ai-workbench"
    -  project_name = var.project_name
    -  region       = var.region
    -  network      = module.vpc.network.network_id
    -  subnet       = module.vpc.subnets[local.notebook_subnet_id].id
    +  depends_on                = [module.jupyter_docker_image, module.vpc]
    +  source                    = "./modules/vertex-ai-workbench"
    +  project_name              = var.project_name
    +  region                    = var.region
    +  network                   = module.vpc.network.network_id
    +  subnet                    = module.vpc.subnets[local.notebook_subnet_id].id
    +  ai_notebook_instance_type = "e2-standard-2"
    
       ai_notebook_instance_owner = var.ai_notebook_instance_owner
       ## To remove before workshop
    @@ -49,12 +50,13 @@ module "vertex_ai_workbench" {
    
     #
     module "dataproc" {
    -  depends_on   = [module.vpc]
    -  source       = "./modules/dataproc"
    -  project_name = var.project_name
    -  region       = var.region
    -  subnet       = module.vpc.subnets[local.notebook_subnet_id].id
    -  machine_type = "e2-standard-2"
    +  depends_on          = [module.vpc]
    +  source              = "./modules/dataproc"
    +  project_name        = var.project_name
    +  region              = var.region
    +  subnet              = module.vpc.subnets[local.notebook_subnet_id].id
    +  master_machine_type = "e2-standard-2"
    +  worker_machine_type = "e2-standard-2"
     }
    
     ## Uncomment for Dataproc batches (serverless)
    ```
    
    2. Add support for preemptible/spot instances in a Dataproc cluster

    Modified file: [modules/dataproc/main.tf](modules/dataproc/main.tf)

    Modified code:

    ```diff
    diff --git a/modules/dataproc/main.tf b/modules/dataproc/main.tf
    index 4e70d01..b46a162 100644
    --- a/modules/dataproc/main.tf
    +++ b/modules/dataproc/main.tf
    @@ -17,6 +17,10 @@ resource "google_dataproc_cluster" "tbd-dataproc-cluster" {
         #    }
         software_config {
           image_version = var.image_version
    +
    +      override_properties = {
    +        "dataproc:dataproc.allow.zero.workers" = "true"
    +      }
         }
         gce_cluster_config {
           subnetwork       = var.subnet
    @@ -41,7 +45,7 @@ resource "google_dataproc_cluster" "tbd-dataproc-cluster" {
         }
    
         worker_config {
    -      num_instances = 2
    +      num_instances = 0
           machine_type  = var.machine_type
           disk_config {
             boot_disk_type    = "pd-standard"
    @@ -51,5 +55,9 @@ resource "google_dataproc_cluster" "tbd-dataproc-cluster" {
           }
    
         }
    +
    +    preemptible_worker_config {
    +      num_instances = 2
    +      preemptibility = SPOT
    +    }
       }
     }
    \ No newline at end of file
    ```

    Both normal instances were replaced with preemptible instances.
    
    3. Perform additional hardening of Jupyterlab environment, i.e. disable sudo access and enable secure boot
    
    Modified file: [modules/vertex-ai-workbench/main.tf](modules/vertex-ai-workbench/main.tf)

    Modified code:

    ```diff
    diff --git a/modules/vertex-ai-workbench/main.tf b/modules/vertex-ai-workbench/main.tf
    index c47e79a..019bad3 100644
    --- a/modules/vertex-ai-workbench/main.tf
    +++ b/modules/vertex-ai-workbench/main.tf
    @@ -66,8 +66,13 @@ resource "google_notebooks_instance" "tbd_notebook" {
       instance_owners = [var.ai_notebook_instance_owner]
       metadata = {
         vmDnsSetting : "GlobalDefault"
    +    notebook-disable-root = true
       }
       post_startup_script = "gs://${google_storage_bucket_object.post-startup.bucket}/$    {google_storage_bucket_object.post-startup.name}"
    +
    +  shielded_instance_config {
    +    enable_secure_boot = true
    +  }
     }
    ```

    4. (Optional) Get access to Apache Spark WebUI

    ***place the link to the modified file and inserted terraform code***
