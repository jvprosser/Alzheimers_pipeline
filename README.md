# Alzheimers_pipeline
# CDE-Iceberg-demo


I split the file into two halves and put them into an S3 folder.

## PREP
1. Clone this repo
2. Create DATABASE IF NOT EXISTS LEQEMBI
3. 
4. Drop table if exists LEQEMBI.alzheimers_dataset;
5. 
6. Copy/upload the csv files to an HDFS location. e.g. s3a://go01-demo/tmp
7. Create a CDE VC if needed, with Spark3, Iceberg and Session support
8. Add an airflow connector to your CDE VC for your CDW Hive VW and call it cdw-hive-demo

`   Connection type = hive client wrapper, host = host from jdbc conn string, login/password from workload account`

6. Install the cde CLI if desired,
7. If you see `Error: error in Session endpoint healthcheck <html>` when you test the cde command line, it means you need to
8. Get the jobs api url for this virtual cluster and update the vcluster-endpoint in ~/.cde/config.yaml
9. Prewarm your hive VW
10. Make sure your CML workspace has model registry enabled!



## CDE
3. Go to CDE HOME
4. talk about the UI
5. Go to the sessions and start one up and name it BIOGEN-demo
6. While that is starting,
7. Go bacl to Home and show the details of your VC  e.g. config and Iceberg support
**What is Apache Iceberg?**
>Apache Iceberg is a new open table format targeted for petabyte-scale analytic datasets.

>Developers love it because it supports Slow ACID transactions, Time Travel, Rollback, and in-place schema evolution. Keep in mind that "ACID" is datalake ACID not OLTP ACID - for rapid row updates use Kudu


>Architects love it because it supports streaming and batch ingestion, multi-and hybrid cloud deployments, it's open source, and also engine agnostic.

7. Go to the sessions and start one up and name it BIOGEN-demo
8. Once it comes up:
9. go to the CLI and enter ` ./cde session interact --name BIOGEN-demo`
10. enter a trivial python command like `2+2`
11. then go the the interact tab and you should see this activity there as well.
12. Paste this code in a session:

```

from pyspark.sql.functions import lit, col

tablename = 'alzheimers_dataset'
database='LEQEMBI'

df = spark.read.options(header='True', inferSchema='True', delimiter=',').csv("s3a://go01-demo/user/jprosser/alzheimers_disease_data.csv")

df.printSchema()
```

```
df.writeTo(f"{database}.{tablename}").tableProperty("write.format.default", "parquet").tableProperty("format-version" , "2").using("iceberg").createOrReplace()
spark.sql(f"SELECT * FROM {database}.{tablename}").show(10)

print ("Getting row count")
spark.sql(f"SELECT count(*) FROM {database}.{tablename}").show(10)
```


7. Go back to interact and paste this code:

```
spark.sql(f"SELECT * FROM {database}.{tablename}.history").show()

spark.sql(f"SELECT * FROM {database}.{tablename}.snapshots").show()

```


8. CREATE  A TAG for ML training and retain it forever
```
spark.sql(f" select * from {database}.{tablename}.refs").show() 

ml_tag='ML-Train-07302024-1'
spark.sql(f" ALTER TABLE {database}.{tablename} CREATE TAG `{ml_tag}` AS OF VERSION 1020021702408491525")
spark.sql(f" select * from {database}.{tablename}.refs").show()

```

<!---

8. Enable Write-Audit-Publish
```

spark.sql(f"ALTER TABLE {database}.{tablename} SET TBLPROPERTIES ('write.wap.enabled'='true')")

## Check table DDL for 'write.wap.enabled' = 'true' at the end
spark.sql(f"show create table {database}.{tablename}").collect()[0]['createtab_stmt']

```


8. Create a branch
```
audit_branch = f"Ingest_07302024"

# Create an Audit Branch for staging the new data before writing in prod table
spark.sql(f"ALTER TABLE {database}.{tablename} CREATE BRANCH {audit_branch}")
spark.sql(f" select * from {database}.{tablename}.refs").show()


```
9. Make this new branch
```


## spark.wap.branch set to audit_branch
spark.conf.set("spark.wap.branch", audit_branch)
spark.conf.set('spark.wap.id',audit_branch)


conf = sc.getConf()
print(conf.get("spark.wap.branch")) 
```




9. Insert more data and validate it
```
df = spark.read.options(header='True', inferSchema='True', delimiter=',') \
  .csv(f"s3a://go01-demo/tmp/wine-quality-2.csv")

df.printSchema()


df.writeTo(f"{database}.{tablename}")\
     .tableProperty("write.format.default", "parquet")\
     .using("iceberg")\
     .append()
```

10. See how the snapshot id for main has changed.

```
##See how the snapshot id for main has changed.
spark.sql(f" select * from {database}.{tablename}.refs").show()

spark.sql(f"SELECT * FROM {database}.{tablename}.history").show()

spark.sql(f"SELECT * FROM {database}.{tablename}.snapshots").show()


```


```
# Reading data from Audit Branch:
audit_data = spark.read.option("BRANCH", audit_branch).table(f"{database}.{tablename}")

# check if there are any rows with negative total_amount
neg_amt_df = audit_data.filter(col("quality") < 0)

```


10. check table

```
spark.sql(f"SELECT * FROM {database}.{tablename}").show(10)

print ("Getting row count")

spark.sql(f"SELECT count(*) FROM {database}.{tablename}").show(10)

```


```

spark.read.option("BRANCH", "main").table(f"{database}.{tablename}").count()
```

```
spark.read.option("BRANCH", audit_branch).table(f"{database}.{tablename}").count()
						
```
-->

Talk about this being an iceberg table and that we have our first snapshot!

>Now that we have our code working, let's put it in a job and run it that way!

## Job

1. Go this repo https://github.com/jvprosser/Alzheimers_pipeline
2. click on CDE_pyspark_sql_iceberg.py and show that the code is just like what we ran in the session but there's some session setup and the data file is different.
4. Create a Repository called Alzheimers_project using the git URL for this repo and branch = 'main'
5. Create a job using the repository from the previous step.  Select CDE_pyspark_sql_iceberg.py
6. Run the job.
7. Go to the jobs page and describe it.
8. Go to job runs and describe the page
9. Go to resources, look at was created for the job. Talk about resources.
10. go back to job runs and look at logs, SparkUI, etc
11. Go back the the session and look at the checkpoints. Add another TAG
> Now that our job is working let's start to operationalize it with Airflow!

## Airflow
1. Click on jobs and create an AIRFLOW job and give it a name and select editor, then click the 'Create' button.
2. Select 'Editor' from the menubar.
3. Drag a shell script over and click on the title to change it from script_1 to Check Env - `echo "starting ETL JOB!"`
4. Drag a CDE job over and point to our recently created pyspark job
5. Connect the shell script to the cde job
6. Drag a CDW query over and paste 'select count(*) from LEQEMBI.alzheimers_dataset' ALSO make sure to add the VW connection 'cdw-hive-demo'
7. Connect the CDE job to the CDW query
8. Start the job and look at the job run for it.
9. Go to the Airflow UI and drill in
10. Go to Logs and drill in, show log for each DAG task.

## Iceberg table management and CDW
1. Go back to the session and show the snapshots again.
3. Now go to CDW and talk about it, show visualization
4. Go into a Hive Warehouse HUE session and get the row count

`SELECT COUNT(*) FROM LEQEMBI.alzheimers_dataset;`

6. Select the snapshots again and point out that the last 2 snapshots are duplicates since we ran the pyspark job twice:


```
SELECT * FROM LEQEMBI.alzheimers_dataset.history;

SELECT * FROM LEQEMBI.alzheimers_dataset.snapshots;
```

8. The result should be 7347, which is more than desired.

6. Alter the table to go back one snapshot

`ALTER TABLE LEQEMBI.alzheimers_dataset EXECUTE ROLLBACK(PUT_YOUR_SNAPSHOT_HERE); `

7. Now result should be 4898 from
`select count(*) from LEQEMBI.alzheimers_dataset`

## CML  This demo shows MLFlow. while it works in the demo AWS env it is not GA.
1. Create a project with this git and upload parameters.conf.
2. Start a JupiterLab session with extra CPU and RAM  MAKE SURE YOU START A SPARK 3.3.0 hotfix2 session!
3. Start a terminal and execute `bash -x setup.sh`.
4. ( If this is a git branch, execute `git checkout <branchname>` )
5. Load the EDA notebook and step through most of it.
6. Start another session with a workbench running Spark3
7. Get the first snapshot ID from HUE and assign it to `first_snapshot` at the top
>I'm using the first snapshot because I always want to train the model using the same dataset.  In Q4, there will be support for Iceberg Tags and Branching, which can be used to attach meaningful labels these snapshots.
8. Confirm that the CONNECTION_NAME variable matches the *Spark Data Lake*  Data "Connection Code Snippet" element.
9. Step through the sections of the file executing in chunks.
10. Note that we are loading the same data but from an even earlier snapshot, because we always want to train from a known dataset.
11. Run the code 2 or 3 more times to generate more experiment data. The hyperparameters are randomly generated.
12. Look at the experiments.
13. Check two experiments and click the Compare button on the right.
14. Look at the plots, zoom in
16. Pick one and scroll down to artifacts
17. Click on model, discuss
18. Click *Register Model* button and fill in the form. Pick a suitable name.
19. Click on Model Deployments, discuss, click on *New Model*
20. Select *Deploy registered model*
21. in the *Registered Model* field, select the model name from above.
22. uncheck *Enable Authentication*

>Note: MLFlow uses pandas JSON format and so the example input needs to be in the format below:

Enter Example Input

```
{
  "columns": [
    "alcohol",
    "chlorides",
    "citric acid",
    "density",
    "fixed acidity",
    "free sulfur dioxide",
    "pH",
    "residual sugar",
    "sulphates",
    "total sulfur dioxide",
    "volatile acidity"
  ],
  "data": [
    [
      12.8,
      0.029,
      0.48,
      0.98,
      6.2,
      29,
      3.33,
      1.2,
      0.39,
      75,
      0.66
    ]
  ]
}
```

And Sample Response:

`{    "quality": 6 }`

23. Click *Deploy*
24. Once deployment completes, click on the model, discuss [ The monitoring tab is showing errors, so don't click on it! ]
26. Click on *Test* button at bottom.
27. Talk about how to access this as a REST endpoint or set up an Application around it.


## Viz
1. If there's time, go to the data tab you opened earlier.
2. go to Datasets
3. select 'default-hive-data'
4. click New Dataset ( from table - default - alzheimers_dataset )
5. click on Fields - edit fields
6. convert some of the measures to dimensions - density, ph, alcohol, quality
7. click new dashboard on the right.
