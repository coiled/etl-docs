Deployment
==========

*Computers are hard to use reliably*

When we run production systems on a regular basis we need access to reliable
hardware.  This might be a ...

.. toctree::

   Single large VM <single-vm>
   Kubernetes cluster <kubernetes>
   Cloud platform (Coiled) <coiled>

When assessing these options, we'll want to think about the following
considerations:

-  Cost
-  Ease of development
-  Security of both network and data
-  Logs and metrics that are easy to find

The right choice often depends on our required scale, our access to devops
resources, and other technologies used within our company.


Reference Architecture
----------------------

In the :doc:`reference architecture <tpch>` we used :doc:`Coiled <coiled>` to manage hardware
in the cloud.  Coiled was designed around Dask and integrates well with the
other tools.  We used Coiled in a few ways:

First, to back our Prefect tasks with serverless functions that run efficiently
on cloud hardware

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

Second, to back our larger compute tasks with distributed clusters

.. code-block:: python

   # pipeline/reduce.py

   import coiled
   from prefect import task, flow
   import dask.dataframe as dd

   @task
   def query_for_bad_accounts(bucket):
       with coiled.Cluster(n_workers=20, region="us-east-2") as cluster:
           with cluster.get_client() as client:
               df = dd.read_parquet(bucket)

               ... # complex query

               result.to_parquet(...)

Third, to deploy the long-running servers, like the Prefect flow itself:

.. code-block::

   $ coiled prefect serve workflows.py

Coiled is free for moderate use on top of your own cloud account, so it's easy
to recommend fairly broadly, even for non-commercial users.
