Orchestration
=============

*Run this job every hour.  If it fails, ping me on slack.*

Orchestration or workflow management is the practice of running production
tasks regularly, either on a schedule or triggered by events in the world.
Today it is handled by workflow managers like Airflow, Prefect, Dagster, Argo,
or older systems like cron or Jenkins.

Today in greenfield projects we tend to see the following the most often:

.. toctree::

   prefect
   dagster

All of these technologies combine well with Dask, either using Dask within a
job to scale out that single task, or using Dask as a deployment mechanism to
run many jobs in parallel.

How Dask interacts with Workflow Managers
-----------------------------------------

Dask integrates with workflow managers in two possible ways:

1. Dask within a job/task
~~~~~~~~~~~~~~~~~~~~~~~~~

We can use use Dask within a job/task defined by the workflow manager,
typically to handle jobs that require processing a large dataset or performing
a large computation.

.. code-block:: python

   @task
   def train_big_ML_model(bucket):
       import dask.dataframe as dd

       df = dd.read_parquet("s3://really-big-bucket/a-lot-of-data.parquet")
       model = train(df)
       save(model)

2. Dask to run all tasks in parallel
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Dask can also be used underneath the more modern workflow managers (like
Prefect or Dagster) to scale out execution, much as one might use Kubernetes or
a cloud service.  Dask is sometimes preferred here because it has higher
throughput than other systems, and can smoothly communicate results between
tasks.

Reference Architecture
----------------------

In the :doc:`reference architecture <tpch>` we used Prefect along with Coiled
functions.  This looks like the following:

.. code-block:: python

   # pipeline/preprocess.py

   import coiled
   from prefect import task, flow
   import pandas as pd

   @task
   @coiled.function(region="us-east-2", memory="64 GiB")
   def json_to_parquet(filename):
       df = pd.read_json(filename)
       df.to_parquet(OUTFILE / filename.split(".")[-2] + ".parquet")

   @flow
   def convert_all_files():
       files = list_files()
       json_to_parquet.map(files)

   if __name__ == "__main__":
       # Run every five minutes
       convert_all_files.serve(interval=datetime.timedelta(minutes=5))

Prefect handles the scheduling and monitoring of the tasks, while Coiled (see
:doc:`deployment`) gets cloud hardware for our jobs on-demand.
