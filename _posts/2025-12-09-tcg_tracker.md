---
layout: post
title:  "TCG Tracker Design Document"
author: David Chen
email: david@davidc.dev
date:   2025-12-09
categories: document learnings design_doc
---

* TOC 
Extra line to be overwritten
{:toc}


TCGs have been rising in popularity and with it the prices of cards have been rising and falling constantly. 

I was looking for a way to be alerted about any changes within the cards I already had, so that I could potentially look at selling them at the next card show or listing them on TCGPlayer. 

I believe one of the biggest oppotunities in collecting is being able to sell playable cards at their peak. Being able to automate that kind of manual checking is what I set out to do.

## Goal / Motivation

My goal is to leverage my skills and experience with the Google Cloud Platform(GCP) to build an automated pipeline to collect TCG data and make it readiliy available for analysis. 

The next step would be to take that data and create daily alerts that would allow me to deep dive into the data to validate if the findings are correct.

## Tech Stack

* Platform
    * Google Cloud Platform(GCP)
* Storage
    * Google Cloud Storage(GCS)
* Data Warehouse
    * BigQuery
        * Supports Paraquet
        * Free monthly credits
    * Snowflake - Snowflake was considered but decided against it
        * No Free Credits - GCP has free monthly credits for BigQuery Analysis. Snowflake would incur a monthly cost.
        * Data Storage - BigQuery can leverage pre-defined external tables within GCS to cut down on costs. Snowflake would have to ingest some of all of the data into their own Data Warehouse to be able to run queries.
* Data Pipeline
    * GCP - Cloud Run
        * Cloud Run is Google's Fully Managed Container Manager. It can be used for both setting up services as well as setting up periodic jobs.
        * By writing my code as Docker container I can ensure that my code can be reliably run anywhere.
        * By using Cloud Run, there is monthly free credits to use and thus I do not need to pay anything to utilize Cloud Run.
* Cloud Artifacts
    * GCP - Artifact Registry
        * To utilize Cloud Run, you will need to build a Dockerfile into a binary and have it live within a platform or within Artifact Registry. 
        * There is a lower cap on how much Artifact Registry can hold before it starts charging. The limit 500 MBs across all projects. Once it is exceeds, it is cents per GB to host the files.
* Version Control
    * Github - <https://github.com/dchen-8/tcg_tracker>{:target="_blank"}

## High Level Process

Every day, Cloud Run will trigger my Docker Container and run a script to get price data from TCGCSV.com. This data will then be transformed into a Paraquet file and loaded into Google Cloud Storage in a year-month-date file format. This data will be read from BigQuery as an external table which will allow BigQuery to run SQL queries across the data without having to ingest the data into BigQuery. This would scale and save on costs. 

Every week, Cloud Run will trigger another Docker Container and run a script to get product data from TCGCSV.com. This data is over 100K records and thus can only be gathered once a week otherwise the amount of data being collected could become quite large. Prices data is much smaller as there is less fields to return. The data will be converted to Paraquet and loaded into Google Cloud Storage in a year-month-date file format.

## Issues

### Python Alphine Image not compatible with Cloud Run

Cloud Run was having an issue with the Docker Alpine image (All versions), it kept returning an error 

```
RuntimeWarning: As the c extension couldn't be imported, `google-crc32c` is using a pure python implementation that is significantly slower. If possible, please configure a c build environment and compile the extension
  warnings.warn(_SLOW_CRC32C_WARNING, RuntimeWarning)
```

I was able to find this comment about running the Docker build command with extra arguments to bypass the issue. <https://github.com/googleapis/python-crc32c/issues/83#issuecomment-910515271>{:target="_blank"}

I was eventually able to build Cloud Image that would run on Cloud Run with this command: 

```
gcloud run jobs deploy tcg-tracker --source . --region us-west1 --set-env-vars CRC32C_PURE_PYTHON=1
```

### File Path Format for BigQuery External Table

BigQuery supports Hive Paritioning of the data. Generally I would have wanted to put the data into a dated file path name like '/yyyy/mm/dd/file.data' which is a nice looking struture in which data hierarchy is easy to see. Hive paritioning is differnt in that the file path has tags embbed which allows Hive to under stand the inferred column name.

<https://docs.cloud.google.com/bigquery/docs/hive-partitioned-queries#supported_data_layouts>{:target="_blank"}

### Paraquet conversion without Pandas

Paraquet format is built into Pandas as a libarry but there is no easy way to do the conversion. The goal was to avoid using Pandas because it would cause the built file to be closer to 500 MBs rather than the 188 MBs without Pandas and only using PyArrow. I was able to figure out how to take a list of data and convert it to Paraquet without using Dataframes and Pandas. 

I had to open a GCS path with 'write-binary' and then convert the data into a PyArrow dataframe and then finally write the data into GCS as a paraquet file.

``` python
    with blob.open("wb") as f:
        results = pa.Table.from_pylist(data_dict)
        pq.write_table(results, f)
```

## Learnings

I was able to set up a production capable platform in which I can ingest data and analyze data at scale. While these learnings are not as easy to replicate for any other cloud platform. There is strong evidence that it is possible. My choice in using GCP is familiarity with the systems as well as the monthly credits that make such a project financially feasible.

## Future Goals

### Embeded Looker Studio

I want to be able to use Looker Studio(Free version) to query a view of the most changed cards and be able to deep dive into the data I have. A usecase would be to see the trends of card prices.

### Daily TCG Movement Reports

I want to be able to get a daily email about the changes to cards that I might want to list for sale. 