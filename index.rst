ETL with Dask
=============

This website documents using Dask and related Python technologies to build ETL
pipelines that serve **recurring**, **production** systems at **large scale**
including tasks like ...

-  **Data pre-processing**
-  **Large scale queries**.
-  **Model training**

This documentation covers common problems and questions faced in building these
systems, including pragmatic questions like ...

-  *How do I wrangle 20 TiB datasets?*
-  *How do I wrangle them every day?*
-  *How do I deploy this in the cloud?*
-  *How do I push changes to my pipeline with a CI/CD pipeline?*
-  *How do I anticipate costs?*
-  *How do I access logs?*

In doing so, this website points to common tooling in the Python ecosystem.

Reference Architecture
----------------------

Additionally, this documentation leverages a self-contained :doc:`reference
architecture <tpch>` that implements these patterns and can be easily copied,
run, and modified to different applications.  We will refer to this worked
example throughout.  This reference architecture uses tools like ...

-  Dask
-  Prefect
-  Coiled
-  Deltalake
-  Streamlit
-  FastAPI

To get started, take a look at the :doc:`reference architecture <tpch>` or
investigate the table of contents to the left.

.. toctree::
   :maxdepth: 2
   :hidden:

   tpch
   transformation
   storage
   orchestration
   deployment
