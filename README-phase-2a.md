IMPORTANT ❗ ❗ ❗ Please remember to destroy all the resources after each work session. You can recreate infrastructure by creating new PR and merging it to master.

![img.png](doc/figures/destroy.png)

0. The goal of this phase is to create infrastructure, perform benchmarking/scalability tests of sample three-tier lakehouse solution and analyze the results using:
* [TPC-DI benchmark](https://www.tpc.org/tpcdi/)
* [dbt - data transformation tool](https://www.getdbt.com/)
* [GCP Composer - managed Apache Airflow](https://cloud.google.com/composer?hl=pl)
* [GCP Dataproc - managed Apache Spark](https://spark.apache.org/)
* [GCP Vertex AI Workbench - managed JupyterLab](https://cloud.google.com/vertex-ai-notebooks?hl=pl)

Worth to read:
* https://docs.getdbt.com/docs/introduction
* https://airflow.apache.org/docs/apache-airflow/stable/index.html
* https://spark.apache.org/docs/latest/api/python/index.html
* https://medium.com/snowflake/loading-the-tpc-di-benchmark-dataset-into-snowflake-96011e2c26cf
* https://www.databricks.com/blog/2023/04/14/how-we-performed-etl-one-billion-records-under-1-delta-live-tables.html

2. Authors:
- [x] STATUS
- ***group nr 3***

-   ***https://github.com/karpinskiJ/tbd-2023z-phase1_gr3***

3. Replace your `main.tf` (in the root module) from the phase 1 with [main.tf](https://github.com/bdg-tbd/tbd-workshop-1/blob/v1.0.36/main.tf)
and change each module `source` reference from the repo relative path to a github repo tag `v1.0.36` , e.g.:
```hcl
module "dbt_docker_image" {
  depends_on = [module.composer]
  source             = "github.com/bdg-tbd/tbd-workshop-1.git?ref=v1.0.36/modules/dbt_docker_image"
  registry_hostname  = module.gcr.registry_hostname
  registry_repo_name = coalesce(var.project_name)
  project_name       = var.project_name
  spark_version      = local.spark_version
}
```
- [X] STATUS

4. Provision your infrastructure.

    a) setup Vertex AI Workbench `pyspark` kernel as described in point [8](https://github.com/bdg-tbd/tbd-workshop-1/tree/v1.0.32#project-setup) 

    b) upload [tpc-di-setup.ipynb](https://github.com/bdg-tbd/tbd-workshop-1/blob/v1.0.36/notebooks/tpc-di-setup.ipynb) to 
the running instance of your Vertex AI Workbench
- [X] STATUS
5. In `tpc-di-setup.ipynb` modify cell under section ***Clone tbd-tpc-di repo***:

   a)first, fork https://github.com/mwiewior/tbd-tpc-di.git to your github organization.  
  Link to forked repo: https://github.com/karpinskiJ/tbd-tpc-di_gr3

   b)create new branch (e.g. 'notebook') in your fork of tbd-tpc-di and modify profiles.yaml by commenting following lines:
   ```  
        #"spark.driver.port": "30000"
        #"spark.blockManager.port": "30001"
        #"spark.driver.host": "10.11.0.5"  #FIXME: Result of the command (kubectl get nodes -o json |  jq -r '.items[0].status.addresses[0].address')
        #"spark.driver.bindAddress": "0.0.0.0"
   ```
   This lines are required to run dbt on airflow but have to be commented while running dbt in notebook.

   c)update git clone command to point to ***your fork***.

- [X] STATUS


6. Access Vertex AI Workbench and run cell by cell notebook `tpc-di-setup.ipynb`.

    a) in the first cell of the notebook replace: `%env DATA_BUCKET=tbd-2023z-9910-data` with your data bucket.


   b) in the cell:
         ```%%bash
         mkdir -p git && cd git
         git clone https://github.com/mwiewior/tbd-tpc-di.git
         cd tbd-tpc-di
         git pull
         ```
      replace repo with your fork. Next checkout to 'notebook' branch.
   
    c) after running first cells your fork of `tbd-tpc-di` repository will be cloned into Vertex AI  enviroment (see git folder).

    d) take a look on `git/tbd-tpc-di/profiles.yaml`. This file includes Spark parameters that can be changed if you need to increase the number of executors and
  ```
   server_side_parameters:
       "spark.driver.memory": "2g"
       "spark.executor.memory": "4g"
       "spark.executor.instances": "2"
       "spark.hadoop.hive.metastore.warehouse.dir": "hdfs:///user/hive/warehouse/"
  ```
- [X] STATUS

7. Explore files created by generator and describe them, including format, content, total size.

   The generator was meant to download raw data for tpc-di tests. In first place data was downloaded to temporary location <br>
   /tmp/tpc-di. Downloaded data was divided into 3 batches. Each of them contained files in following formats: <br>
-   .txt 
-  application/octet-stream 
- .xml 
- .csv  
The various formats of loaded data are also goal of tpc-di tests, to check how different formats affect performance of warehouse.
<br>
The root structure of downloaded data is presented below: <br>

![batch_1_ls](/doc/figures/root_structure.png)

Largest in size, Batch1 consumes approximately 940 MiB of storage, primarily comprising CSV files. CSV, a common format for data storage, is widely employed for data import/export in databases. In these files, each line corresponds to a data record, and fields, separated by commas, denote attributes of the record.

Comparatively smaller, both Batch2 and Batch3 utilize around 12 MiB of space each. These batches house CSV files intended for the subsequent loading stage, facilitating the transfer of data into the database.

The total number of records in each batch is as follows:

Batch1: 15,980,433 records
Batch2: 67,451 records
Batch3: 67,381 records

![batch_2_ls](/doc/figures/batch_1_ls.png)

- [X] STATUS
8. Analyze tpcdi.py. What happened in the loading stage?
It is designed to load TPC-DI (Transaction Processing Performance Council - Decision Support) generated files into a Data Lakehouse using PySpark.
The script transform data from local storage to database format (creates spark DataFrames) and then upload data into
GCP storage.
- process_files: Serves as the primary function for initiating the processing of TPC-DI files.
- get_stage_path:Constructs the Google Cloud Storage path for a given stage and file name.
- upload_files: Manages the uploading of files to the designated Google Cloud Storage stage.
- load_csv: Facilitates the loading of CSV files into Spark DataFrames.
<br>
- Overview of data loaded into bucket:  

![image](https://github.com/karpinskiJ/tbd-2023z-phase1_gr3/assets/83401763/385ff540-a583-470d-8931-cc25c365365d)

- [X] STATUS
<br>
9. Using SparkSQL answer: how many table were created in each layer?


```
databases = spark.sql("show databases").rdd.map(lambda x : x[0]).collect()
number_of_tables = []
for database in databases:
    spark.sql(f"use {database}")
    tables = spark.sql("show tables")
    number_of_tables.append((database, tables.count()))

statistics =  spark.createDataFrame(number_of_tables,["database_name","number of tables"])
statistics.show()
```
![number_of_tables](/doc/figures/number_of_tables.png)


- [X] STATUS
10. Add some 3 more [dbt tests](https://docs.getdbt.com/docs/build/tests) and explain what you are testing. ***Add new tests to your repository.***

   ***Code and description of your tests***  
  **fact_cash_balance_registered_customer.sql** - test to check if every customer having account in fact_cash_balances table is registered in dim_customer.
  ```sql
    SELECT DISTINCT fcb.sk_customer_id
    FROM 
        {{ ref('fact_cash_balances') }} fcb
    LEFT JOIN 
        {{ ref('dim_customer') }} dc
    ON 
        fcb.sk_customer_id = dc.sk_customer_id
    WHERE
        dc.sk_customer_id IS NULL
  ```
  
  **fact_trade__registered_in_dim_broker.sql** - test to check if every broker having trades in fact_trade table is registered in dim_broker.
      
  ```sql
    SELECT DISTINCT ft.sk_broker_id
    FROM 
        {{ ref('fact_trade') }} ft
    LEFT JOIN 
        {{ ref('dim_broker') }} db
    ON 
        ft.sk_broker_id = db.sk_broker_id
    WHERE
        db.sk_broker_id IS NULL
  ```

  **fact_trade__unique_trade.sql** - test to check if every sk_account_id is unique in dim_account.
 
  ```sql
    SELECT 
        sk_account_id, 
        count(*) cnt
    FROM {{ ref('dim_account') }} 
    GROUP BY sk_account_id
    HAVING cnt > 1
  ```

11. In main.tf update  DONE
   ```
   dbt_git_repo            = "https://github.com/mwiewior/tbd-tpc-di.git"
   dbt_git_repo_branch     = "main"
   ```
   so dbt_git_repo points to your fork of tbd-tpc-di. 

12. Redeploy infrastructure and check if the DAG finished with no errors:

***The screenshot of Apache Aiflow UI***
![image](https://github.com/karpinskiJ/tbd-2023z-phase1_gr3/assets/83401763/9a666f3e-3701-4c91-a314-191493e74309)
