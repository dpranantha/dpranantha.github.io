---
layout: post
title: "Using Apache Beam vs other frameworks for bulk data processing"
permalink: /using-apache-beam-vs-other-frameoworks-for-bulk-data-processing
date: 2021-01-01 14:15:00+01:00
categories:
- Coding
- Data engineering
tags:
- Apache Beam
- Apache Flink
- Apache Spark
- Kotlin
- Google Cloud Platform
collection: blog
---

[Apache Beam](https://beam.apache.org/) is a unified programming model for implementing batch and stream processing. It can run on
multiple runners, such as [Apache Flink](https://flink.apache.org/), [Apache Spark](https://spark.apache.org/), and [Google Cloud Dataflow](https://cloud.google.com/dataflow/?utm_source=google&utm_medium=cpc&utm_campaign=emea-nl-all-en-dr-bkws-all-all-trial-e-gcp-1009139&utm_content=text-ad-none-any-DEV_c-CRE_253523886152-ADGP_Hybrid%20%7C%20AW%20SEM%20%7C%20BKWS%20~%20EXA_M%3A1_NL_EN_Data_Dataflow_SEO-KWID_43700053285543828-aud-606988878694%3Akwd-315945827235-userloc_9064237&utm_term=KW_dataflow%20google%20cloud-NET_g-PLAC_&&gclid=EAIaIQobChMIgdTJlK357QIVSuh3Ch0IdwwjEAAYASAAEgLWDvD_BwE).
This advantage makes Apache Beam more attractive compare to other frameworks, such as Apache Flink (for stream processing) or Apache Spark (for bulk/batch processing).
However, Apache Beam has some caveats that are not suitable for some stream processing use cases, such as:
1. [No ordering guarantees](https://stackoverflow.com/questions/45888719/processing-total-ordering-of-events-by-key-using-apache-beam/45911664#45911664), which means our application must handle/tolerate unordered messages.
2. At least once, which means our application must handle/tolerate duplicates.

Thus, when we can use Apache Beam? Answering based on my own experiences, I have three conditions to choose Apache Beam over other frameworks. 
1. We use Google Cloud Platform (GCP). Google Dataflow has some operational advantages: dynamic scaling and managed infrastructure.
   Those advantages vanish if we run on top of Apache Flink or Apache Spark.
2. We want to perform bulk/batch data transformation. It is easy using GCP native technologies, 
   such as Google Bigquery or Google Cloud Storage. Don't use Apache Beam to train (Machine Learning) models, but opt for Apache Spark/Pyspark instead.
   Apache Spark/Pyspark has tons of useful libraries intended for such tasks.
3. We want to perform event stream processing where ordering and duplicates don't matter. Also, we don't care about calculating time spent on an event.
   Otherwise, opt for Apache Flink. Although Apache Flink is more challenging in infrastructural maintenance and scaling,
   it has *the ordering and exactly once guarantees*. Also, it is easy to perform a calculation of time spent on an event. 
   On the other hand, if we want to apply machine learning on event streams, Apache Spark is a proper tool for that (i.e., DStream with windowing function).
   
I have created an example of bulk data processing using Apache Beam available at my Github repo: [apache-beam-kotlin-bulk-sample](https://github.com/dpranantha/apache-beam-kotlin-bulk-sample). 
The example is adapted from the Apache Beam example ported into Kotlin and added with an integration test.

Have fun coding!
