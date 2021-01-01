---
title: "Using Apache Beam vs other frameworks for bulk data processing"
date: 2020-01-01 14:15:00+01:00
categories:
- Coding
- Data engineering
tags:
- Apache Beam
- Apache Flink
- Apache Spark
- Kotlin
- Google Cloud Platform
---

[Apache Beam](https://beam.apache.org/) is a unified programming model for implementing batch and stream processing. It can run on
multiple runners, such as [Apache Flink](https://flink.apache.org/), [Apache Spark](https://spark.apache.org/), and [Google Cloud Dataflow](https://cloud.google.com/dataflow/?utm_source=google&utm_medium=cpc&utm_campaign=emea-nl-all-en-dr-bkws-all-all-trial-e-gcp-1009139&utm_content=text-ad-none-any-DEV_c-CRE_253523886152-ADGP_Hybrid%20%7C%20AW%20SEM%20%7C%20BKWS%20~%20EXA_M%3A1_NL_EN_Data_Dataflow_SEO-KWID_43700053285543828-aud-606988878694%3Akwd-315945827235-userloc_9064237&utm_term=KW_dataflow%20google%20cloud-NET_g-PLAC_&&gclid=EAIaIQobChMIgdTJlK357QIVSuh3Ch0IdwwjEAAYASAAEgLWDvD_BwE) among others.
This makes Apache Beam more attractive compare to other frameworks such as Apache Flink (for stream processing), or Apache Spark (for bulk/batch processing).
However, Apache Beam has some caveats that are not suitable for some stream processing use cases, such as:
1. [No ordering guarantees](https://stackoverflow.com/questions/45888719/processing-total-ordering-of-events-by-key-using-apache-beam/45911664#45911664). This means our application must handle/tolerate unordered messages.
2. At least once. This means our application must handle/tolerate duplicates.

Thus, when we can use Apache Beam? Answering based on my own experiences, I have three conditions to choose Apache Beam over other frameworks. 
1. Google Cloud Platform (GCP) and in this case Google Dataflow is part of our tech-stack. This gives us
   some operational advantages, such as dynamic scaling, no prior infrastructural setup, and no infrastructural management.
   Those advantages vanish if we run on top of Apache Flink or Apache Spark.
2. We want to solely perform bulk/batch data transformation. This is relatively easy, especially if we use GCP native technologies, 
   such as Google Bigquery or Google Cloud Storage. Don't use Apache Beam to train (Machine Learning) models, but opt for Apache Spark/Pyspark instead.
   Apache Spark/Pyspark has tons of useful libraries intended for such tasks.
3. We want to perform event stream processing that doesn't care about ordering nor duplicates nor calculating time spent on an event,
   otherwise, opt for Apache Flink. Although using Apache Flink is harder in terms of infrastructural maintenance and scaling,
   at least it has ordering and exactly once guarantees, and it is easy to perform calculation of time spent on an event.
   
I have created an example of bulk data processing using Apache Beam available at my github repo: [apache-beam-kotlin-bulk-sample](https://github.com/dpranantha/apache-beam-kotlin-bulk-sample). 
The example is adapted from Apache Beam example ported into Kotlin and added with an integration test.

Have fun coding!
