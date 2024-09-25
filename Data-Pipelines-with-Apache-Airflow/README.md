## Project: Data Pipelines with Airflow

**Project Description**: A music streaming company aims to enhance the automation and monitoring of their data warehouse ETL pipelines and has chosen **Apache Airflow** as the ideal tool for this task. As the Data Engineer, my role was to develop a reusable, production-grade data pipeline that includes data quality checks and supports easy backfills. This pipeline, which runs daily, pulls new data from the source and stores the results at the destination, serving several analysts and data scientists.

**Data Description**: The source data is stored in S3 and needs to be processed into a data warehouse on Amazon Redshift. The datasets include JSON logs detailing user activity and JSON metadata about the songs users listen to.

**Data Pipeline Design**:
At a high level, the pipeline performs the following tasks:
1. Extracts data from multiple S3 locations.
2. Loads the data into a Redshift cluster.
3. Transforms the data into a star schema.
4. Performs data validation and quality checks.
5. Calculates the most played songs for the specified time interval.
6. Loads the results back into S3.

![dag](images/dag.png)
> Structure of the Airflow DAG

**Design Goals**:
Based on our data consumers' requirements, the pipeline adheres to these guidelines:
- The DAG should not depend on past runs.
- On failure, tasks are retried three times.
- Retries occur every five minutes.
- Catchup is disabled.
- No email notifications on retry.

**Pipeline Implementation**:

Apache Airflow is a Python framework for programmatically creating workflows in DAGs, such as ETL processes, generating reports, and retraining models daily. The Airflow UI parses our DAG and visually represents the data movement and transformation. A **DAG** describes *how* the workflow should be executed, and **Operators** determine *what* tasks get done.

Airflow includes built-in operators like `PythonOperator`, `BashOperator`, and `DummyOperator`, but it also allows for custom operators. For this project, I developed several custom operators.

![operators](images/operators.png)

Descriptions of these custom operators:
- **StageToRedshiftOperator**: Stages data to a specific Redshift cluster from a specified S3 location, using templated fields to handle partitioned S3 locations.
- **LoadFactOperator**: Loads data into the specified fact table by executing a provided SQL statement, supporting both delete-insert and append styles.
- **LoadDimensionOperator**: Loads data into the specified dimension table using a provided SQL statement, supporting both delete-insert and append styles.
- **SubDagOperator**: Groups tasks, such as checking if a table has rows and running data quality SQL commands, into one task.
  - **HasRowsOperator**: Ensures the specified table contains rows as part of data quality checks.
  - **DataQualityOperator**: Runs SQL statements to validate data quality.
- **SongPopularityOperator**: Calculates the top ten most popular songs for a given interval based on the DAG schedule.
- **UnloadToS3Operator**: Stores analysis results back into a specified S3 location.

> The code for these operators is located in the **plugins/operators** directory.

**Pipeline Schedule and Data Partitioning**: 
The events data on S3 is partitioned by *year* (2018) and *month* (11). Our task is to incrementally load these event JSON files through the pipeline to calculate song popularity and store the results back into S3, obtaining daily top songs automatically.

*S3 Input events data*:
```bash
s3://<bucket>/log_data/2018/11/
2018-11-01-events.json
2018-11-02-events.json
2018-11-03-events.json
..
2018-11-28-events.json
2018-11-29-events.json
2018-11-30-events.json
```

*S3 Output song popularity data*:
```bash
s3://skuchkula-topsongs/
songpopularity_2018-11-01
songpopularity_2018-11-02
songpopularity_2018-11-03
...
songpopularity_2018-11-28
songpopularity_2018-11-29
songpopularity_2018-11-30
```

The DAG configuration includes `default_args` specifying the `start_date`, `end_date`, and other design choices mentioned above.

```python
default_args = {
    'owner': 'shravan',
    'start_date': datetime(2018, 11, 1),
    'end_date': datetime(2018, 11, 30),
    'depends_on_past': False,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'catchup_by_default': False,
    'provide_context': True,
}
```

## How to Run This Project
**Step 1: Create AWS Redshift Cluster**

Run the provided notebook to create an AWS Redshift Cluster. Make a note of:
- DWN_ENDPOINT: `dwhcluster.c4m4dhrmsdov.us-west-2.redshift.amazonaws.com`
- DWH_ROLE_ARN: `arn:aws:iam::506140549518:role/dwhRole`

**Step 2: Start Apache Airflow**

Run `docker-compose up` from the directory containing `docker-compose.yml`. Ensure the volume is mapped to the location of your DAGs.

> **Note: Detailed instructions for managing Apache Airflow on Mac are available here:** [Airflow on Mac](https://gist.github.com/shravan-kuchkula/a3f357ff34cf5e3b862f3132fb599cf3)

![start_airflow](images/start_airflow.png)

**Step 3: Configure Apache Airflow Hooks**

Configure the `S3 connection` with the IAM user's access key and secret key to read data from S3. Configure the `Redshift connection` with values from your Redshift cluster.

![connections](images/connections.png)

**Step 4: Execute the create-tables-dag**

This DAG creates the staging, fact, and dimension tables. Manually trigger this DAG to keep it separate from the main DAG. Once completed, you can turn off this DAG.

**Step 5: Turn on the `load_and_transform_data_in_redshift` DAG**

With the start date set to `2018-11-1` and a daily schedule interval, Airflow will automatically trigger the DAG runs for 30 days. Below are the 30 DAG runs, one per day, triggered by Airflow.

