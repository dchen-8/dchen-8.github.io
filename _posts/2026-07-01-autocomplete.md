---
layout: post
title: "Search/Autocomplete with CloudSQL/Postgres"
author: David Chen
email: david@davidc.dev
date: 2026-07-01
categories: document learnings writeup
---

* TOC
{:toc}

## Why is Autocomplete/Search necessary?

A search bar is necessary to be able to search through all the data (155K rows of TCG products) I have gathered. Autocomplete is necessary nowadays to meet the expectations of users, they would expect to be able to randomly type in a name and get close to what is expected for results. 

This autocomplete search could also be useful when I implement OCR so it can detect card names and then send it to my search API to determine which cards it could be. I might have to add a middleware layer to parse the OCR results but there is a good chance.

## What I tried

I imagined that i would build my search/autocomplete as close as possible to my current tech of GCP but I wanted to try out any option that I had available to me. One thing that stood out was the need to make sure I could keep the costs as low as possible for that reason, I tried both Supabase and Neon as I wanted a Postgres instance and was trying to find the best way to host it. 

Postgres was chosen instead of Elasticsearch (Mostly cuz I forgot about Elasticsearch and when I did look into it, it was like 100 a month for hosting a search box). I wanted to use Postgres's feature of TRGM_OPS where I would be able to apply a GIN Index over it and be able to have very quick trigram searches. 

### BigQuery
I would have preferred to keep everything within BigQuery but I just could not find a way to get it to work. BigQUery would not be able to respond quick enough for my tastes. I think I would be able to get a basic SQL LIKE comparison but would miss out on much other features like fuzzy logic and being able to chain columns together. 

I sadly had to rule out BigQuery, even though it had most of the data I wanted to search through.

### Supabase
I tried out Supabase and at first glance it seemed like a perfect fit. Things seemed to fall into place, the UI was good, I was able to load my dataset (Test dataset 10K ish) and the JS client side was something was debated against because that seemed like a better way to access my data especially if my search bar was exposed to the internet. I was able to get my first taste of Postgres from their nice UI, I was able to easily add in functions like TRGM_OPS and the WRAPPER for Foreign Data Wrapper(FDW). I was able to build a small test run of how the data would be called from an AI created frontend and it seemed to meet my criteria. 

I thought it was a great idea until I started loading my production dataset into it. I had to load 155K rows of data but it was only about 35 MBs. I tried from the UI and was able to load like 40K before the UI timeout and it crashed and couldn't load anymore. I tried to load it again and then it just reloaded the same data I just tried. This is where I started running around to figure out a way to piecemeal load the data into Supabase. 

I wrote a script to try to keep track of what was already written into Supabase and only incrementally upsert the new data. Then I tried the Postgres \COPY function which let me copy data from my local computer and stream it into Postgres, this failed as well. I tried connecting directly to Postgres to upload the data one at a time through a python script but that failed too. I was not able to load the full dataset into Supabase.

There was a few problems but the two biggest was connection and computing. Supabase doesn't really advertise this but they don't have a free IPv4 connection, if you want your instance to have an IPv4, you'll need to pay money for it per project. This meant, I had to go through their AWS session pooler and I think I exceeded the connection limits when I was trying to write a bunch of data. I couldn't get a direct connection to my database to do anything, I had to go through this layer of pooler or client and never felt like I could control and manage the database and my data the way I would want. Also the issue with IPv6 if your system (My system didn't support IPv6), you would get these cryptic error messages on why it wasn't connecting and I kept guessing on why it wouldn't connect and work. So that forced me each time to always use the pooler and it felt like that was yet another bottleneck I could not decipher. 

The other issue was computing, I finally found out the reason why my uploading of data never really worked, well I didn't really find out, I found out when using another service (CloudSQL), I was able to easily dump my data into Google Cloud Storage (GCS) and have CloudSQL import my data from there in like 2 minutes with no timeouts or it failing. This is where I came to realize that Supabase micro instances is really too small to do anything on it for anything other than a micro project with small trickle of data. This made me realize if I wanted to do it on Supabase I would have to upgrade my Supabase project and then might have to pay for a large server which could cost $110 a month not including the Supabase monthly price of $25 a month. 

My goal was to build it as cheaply as possible and getting my system on Supabase seemed like a lost cause so I moved on to try Neon. 

### Neon

Neon was similar to Supabase but instead of charging for instances, they charged for usage. I thought that was a novel concept and wanted to give it a try. Hoping that a more robust backend should have solved my problem. I was wrong, it had the same limitations and honestly I think it was a bit worst than Supabase. The usage never really took off because when I tried the copying or python scripts, they all failed in similar ways. 

The one cool feature that Neon had that Supabase did not was the "Active Queries" tab under Monitoring. This is where I can see my queries being hung... >_> and taking like 10 minutes to do nothing. The other nice thing was that I could cancel my queries from the UI and it was cause my script to fail, so that was nice to get the feedback. The idea of only paying for usage sounds good but I also had the concern if the service would be reachable quickly from a cold start. (I did something similar to fix my Data API on Google App Engine to enable warmup which keeps at least 1 server up at all times).

Neon did not have multiple ways to connect to the Postgres instance and they generally just rehashed the same connection string in multiple formats (plsql, connection string, etc). After testing, I felt like Neon ran into the same issues of Supabase and terminated my testing.

I felt like after all this failure, I gotta go back to the ole reliable enterprise Google Cloud Platform (GCP), I used CloudSQL-MySQL a lot in the past and had a good experience with it. I was hoping I would have a similar experience with Postgres.

### Google Cloud Platform(GCP) CloudSQL

CloudSQL was a dream. I was able to set up a trial Enterprise Plus edition with relative ease. Things just worked and that was really nice.

I was easily able to find out how to import data from Google Cloud Storage (GCS) and after a few missteps, I was able to easily load all the data into the Postgres instance with little to no effort. It just worked, but then again the specs for the trial instance was pretty beefy. I intend to turn down the instance to smaller size later on but it is of great comfort knowing that I can easily scale up my CloudSQL instance if I needed to.

That's when I decided that I will use CloudSQL with Postgres as my defacto database. BigQuery will still be my data warehouse where I can do analytics.

## Postgres Tuning

Once the data was all loaded in, I was able to enable the TRGM_OPS extension and start testing the Postgres instance in production. I was able to easily create a GIN index with TRGM_OPS on a single column but ran into issue when I wanted to do it for multiple columns. A single column would be useful for like names but if I wanted to include other metadata that would be useful during a search to help narrow down a card, that would be a better user experience. 
<br>
<details>
<summary>TRGM_OPS Single Column Example</summary>
<pre><code>

    ``` sql

        CREATE INDEX idx_products_search_trgm 
        ON public.products
        USING gin (name gin_trgm_ops);

    ```
</code></pre>
</details>

<details>
<summary>TRGM_OPS Multiple Column Example</summary>
<pre><code>

    ``` sql
    
        CREATE INDEX idx_products_global_search 
        ON public.products
        USING gin (
        (COALESCE(name, '') || ' ' || COALESCE(group_abbreviation, '') || ' ' || COALESCE(group_name, '') || ' ' || COALESCE(illustration_type, '') || ' ' || COALESCE(rarity, ''))
        gin_trgm_ops
        );

    ```
</code></pre>
</details>

<br>
Once I had the index, I could start testing if the queries I envisioned would work as intended. I tested a bunch and when I was happy with the results, I built a function so that instead of sending a massive query from the browser, I could just call a simple function and it would return the results I would want. The other upside is that when writing the function I can add in query sanitization like converting every " " into a "%" so that it would catch the wildcards. Also setting limits on the function so the max amount that it can return can't be overridden.

<details>
<summary>Sanitize Query Function</summary>
<pre><code>

    ``` sql
    
        CREATE OR REPLACE FUNCTION get_search_query(query TEXT)
        RETURNS TEXT AS $$
        SELECT '%' || array_to_string(string_to_array(query, ' '), '%') || '%'
        $$ LANGUAGE sql;

    ```
</code></pre>
</details>

<details>
<summary>Autocomplete Function</summary>
<pre><code>

    ``` sql
    
        CREATE OR REPLACE FUNCTION autocomplete(search TEXT) 
        RETURNS TABLE (
            product_id TEXT, 
            name TEXT,
            group_name TEXT,
            group_abbreviation TEXT,
            illustration_type TEXT,
            rarity TEXT
        ) AS $$
        SELECT
            product_id, name, group_name, group_abbreviation, illustration_type, rarity
        FROM
            public.products
        WHERE
            1=1
        AND
            (COALESCE(name, '') || ' ' || COALESCE(group_abbreviation, '') || ' ' || COALESCE(group_name, '') || ' ' || COALESCE(illustration_type, '') || ' ' || COALESCE(rarity, '')) ILIKE get_search_query(search)
        ORDER BY 
        (COALESCE(name, '') || ' ' || COALESCE(group_abbreviation, '') || ' ' || COALESCE(group_name, '') || ' ' || COALESCE(illustration_type, '') || ' ' || COALESCE(rarity, '')) <-> get_search_query(search) ASC
        LIMIT 20
        $$ LANGUAGE sql;

    ```
</code></pre>
</details>

<br>
Calling the function is as simple as `SELECT * FROM autocomplete([QUERY_HERE])`, this would return a table that has all the expected results.

Next would be being able to connect to the Postgres instance from Google App Engine (GAE).

## Google App Engine (GAE)

Google App Engine (GAE) is used as the backend to serve data from CloudSQL. I needed to add another route to allow for query terms to be sent from the Frontend UI to be sent to the backend on Google App Engine (GAE) and then the request is finally sent to CloudSQL to be searched. 

I decided not to add Redis to each one of the requests as there could be so many queries that caching them make no sense. 

### Cloud SQL Proxy

One of the bigger challenges was connecting Google App Engine (GAE) together. In the documentation there is a Cloud SQL Proxy that is able to enabled on Google App Engine that will use a dedicated path between the two systems. Locally a Cloud SQL Proxy can be turned on in a terminal and then be reached via Localhost to query CloudSQL. 

Unix Sockets, I decided to use Unix Sockets to be able to get the lowest latency as possible, especially considering my function is for Search/Autocomplete. 


``` zsh
# - u is used for Unix Sockets; Without it should be using TCP

# /tmp is only used for Local development
./cloud-sql-proxy -u /tmp [CLOUD_INSTANCE_NAME]

# /cloudsql is used for Google App Engine (GAE); This actually doesn't work locally
# The concept does transfer over though
./cloud-sql-proxy -u /cloudsql [CLOUD_INSTANCE_NAME]
```

``` yaml
# app.yaml file for Google App Engine (GAE)
beta_settings:
  cloud_sql_instances: [CLOUD_INSTANCE_NAME]
```

### Example

Here is some example code on how the connection to CloudSQL can be connected.

```python
# Configuration for using Psycopg3 to connect with Unix Socket for both Local and GAE
# Payload is from the Secretsmanager
import os
from sqlalchemy import create_engine, text

DB_NAME = payload.get("POSTGRES_DB")
DB_USERNAME = payload.get("POSTGRES_USERNAME")
DB_PASSWORD = payload.get("POSTGRES_PASSWORD")

if os.environ.get("GAE_ENV") == "standard":
    DB_HOST = payload.get("POSTGRES_HOST")
else:
    DB_HOST = payload.get("POSTGRES_LOCALHOST")

db_string = f"postgresql+psycopg://{DB_USERNAME}:{DB_PASSWORD}@localhost:5432/{DB_NAME}"
socket_dir = DB_HOST

engine = create_engine(db_string, connect_args={"host": socket_dir})


def autocomplete(query_string: str):
    "Returns response from Postgres to search through all products"
    with engine.connect() as conn:
        query = text("SELECT * FROM autocomplete(:query_string)")
        result = conn.execute(query, {"query_string": query_string})
        results = result.mappings().all()

    return results
```

## Results

The results are fantastic! This lays the foundation for even more things for me to do. I can imagine being able to add PSA/BGS grading, my own Chen Trading (CT) Market Price and then possibly an expected price for cards. Theres much more I would like to add and continue to keep building.

## Not Shown

I wouldn't really get into detail about how I added the Search Bar to the UI nor how I created a product landing page so I can start displaying the results. The majority of that was done with AI. AI is pretty great at being able to create things it can see. A lot of frontend code is visible through websites everyday and because so much of is visible for humans, it seems to reason that AI would be able to read it much easier and thus be able to recreate things much easier. 

One random thing that I noticed that AI was caught up on was details, like it could not figure out how to implement a feature that would scroll when a user clicked the down arrow on the search results. It took a good 20 minutes to figure it out. 