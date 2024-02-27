Storage
=======

Today we have several good options for large scale storage of production data.
We recommend one of the following:

Parquet in a Bucket
-------------------

.. code-block:: python

   import dask.dataframe as dd

   df = dd.read_parquet("...")
   ...
   df.to_parquet("...")

Parquet is the standard in tabular data storage today.  We find that people
often transform raw data from JSON, CSV, or other operational formats into
Parquet.

Parquet is great because parquet is simple.  They are flat files that can live
in a directory or cloud bucket without any extra complexity.  They're efficient
to read and compact to store.

However, as demands on the dataset increase, the "Parquet in a bucket" approach
can break down. This happens in a few different cases:

1.  **Concurrency:** Automated systems write to and read from this bucket frequently enough that
    they start to collide, causing intermittent failures of either process.
2.  **Time Travel:** Different groups want the data changed at difference cadences, or
    regulatory processes require the ability to look at the data that was
    active at some time in history
3.  **Access Patterns:** different groups want the parquet data indexed by
    diffeent columns for rapid access

Parquet is a great format to start out, and suffices for 90% of groups, but
eventually different needs may develop.

Deltalake
---------

.. code-block:: python

   import deltalake
   import pandas as pd

   df = pd.read_csv("...")

   deltalake.write_table("mytable", df, mode="append")

   import dask_deltatable

   df = dask_deltatable.read_deltalake("mytable", datetime="2020-01-01")

Deltalake augments flat Parquet files in the same way that ``git`` augments
flat source code files, by wrapping it with important metadata and a system of
commits.  Deltalake manages concurrent reads and writes so we can handle
collisions, as well as a system to manage time travel.

Deltalake, like ``git``, is still just a directory of files, and so it composes
well with existing storage solutions.  Tooling is robust and is an easy and
simple step beyond Parquet.

Snowflake
---------

.. code-block:: python

   import dask_snowflake

   example_query = """
       SELECT *
       FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.CUSTOMER;
   """
   df = read_snowflake(
       query=example_query,
       connection_kwargs={
           "user": "...",
           "password": "...",
           "account": "...",
       },
   )

Snowflake is a more fully featured database that we often find as the system of
record in large organizations.  Dask reads from Snowflake natively using their
bulk export mechanism, which is quite fast.  `Read more here <https://www.coiled.io/blog/snowflake-and-dask>`_ about how
this was built.

Snowflake and Dask tend to be used together to do bulk querying and initial
pre-processing with Snowflake, followed by more advanced algorithms with Dask,
such as training with XGBoost or PyTorch, or using more native Python functions
in a way that is maybe more ergonomic than Snowpark.

Google BigQuery
---------------

.. code-block:: python

   import dask_bigquery

   df = dask_bigquery.read_gbq(
       project_id="your_project_id",
       dataset_id="your_dataset",
       table_id="your_table",
       columns=["Name", "id", "Value"],
   )

Google BigQuery is a well-loved database inside GCP.  It's easy to pull out
subsets of tables into distributed Dask Dataframes to perform more advanced
queries than what is allowed by GBQ.


Reference Architecture
----------------------

In the :doc:`reference example <tpch>` we used Deltalake, which combines the
performance and simple flat-file nature of Parquet files with automatic
compaction and the ability to reason about concurrent reads and writes.  See
`pipelines/preprocessing.py <https://github.com/coiled/etl-tpch/blob/main/pipeline/preprocess.py>`_
for more details.

.. code-block:: python

   @task
   @coiled.function(region="...")
   def json_to_parquet(file):
       """ Convert one file to parquet+delta format """
       df = pd.read_json(file, lines=True)
       outfile = STAGING_PARQUET_DIR / file.parent.name
       data = pa.Table.from_pandas(df, preserve_index=False)
       deltalake.write_deltalake(
           outfile, data, mode="append",
       )

   @task
   @coiled.function(region="...")
   def compact(table):
       """ Compact many small files into larger files for compute efficiency """
       t = deltalake.DeltaTable(table)
       t.optimize.compact()
       t.vacuum(retention_hours=0, enforce_retention_duration=False, dry_run=False)

These two tasks get run by single machines on regular cadences.  We're then
able to read that data easily at scale in different tasks, for example in
`pipeline/reduce.py
<https://github.com/coiled/etl-tpch/blob/main/pipeline/reduce.py>`_.

.. code-block:: python

   import dask_deltatable

   ...

   parts = dask_deltatable.read_deltalake(PARQUET_DIR / "parts")
   suppliers = dask_deltatable.read_deltalake(PARQUET_DIR / "suppliers")

   df = parts.merge(suppliers, on="part_id")
   ...

