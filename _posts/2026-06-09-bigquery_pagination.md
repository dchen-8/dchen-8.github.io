---
layout: post
title: "Pagination with BigQuery"
author: David Chen
email: david@davidc.dev
date: 2026-06-09
categories: document learnings writeup
---

* TOC
{:toc}

## Why Pagination

My TCG Tracker Project is going well but there is a small issue with load time when clicking on pages.

Sometimes when I click on a page, it briefly flashes a "Data Loading" message and it lasts longer when the data is trying to fetch bigger and bigger datasets. The initial way that I wrote the API was to essentially return all the results in one giant 'List of Dictionaries' for each category.

This was happening to the "Price Trends", "Group Products" and "Sealed Products" tabs. Once I started collecting Pokemon data, the total price changes per day was getting to be about 1500-2000.

I needed a way to cut down on the load time.

That's when I started planning on how to implement Pagination.

## Which Pagination

There was a few ways I considered to do pagination.

I considered using the built in FastAPI module that would be able to query a data source and paginate itself with minimal setup. The reason I did not choose that was that all of my data is in BigQuery and I could not find a way to easily integrate FastAPI with BigQuery (I was seeing that I would have to have BigQuery return all the data to FastAPI and then have FastAPI create a generator type to only partially return data).

I looked at potentially using Supabase as a data source where I would load the data for the dashboards(Price Trends, Group Products and Sealed Products) then have Supabase act as the API to return the data when called. Supabase had the ability to call it directly from JS without having to go through the Postgres backend with Row Level Security(RLS). I decided against it when I thought about how much overhead it would take to keep the data in sync with BigQuery and Supabase.

I eventually decided to see if BigQuery could support doing pagination. After digging in the docs for a bit, I found out it was possible. BigQuery would be able to return parts of a query on demand and not have to re-query each time and each query would be cached for 24 hours so it would not increase the amount of data queried per day.

## How to enable Pagination with BigQuery

BigQuery Pagination is pretty straight forward.

### BigQuery

You run a query, that query generates a job id, that job id can be used to find the temporary table that was created, the table can be queried like a normal table with parameters of max rows and page token, the table is queried with the inputs and the results return the data, total rows for the query and a page token for the next page.

### Data API

The data returned from BigQuery is then written into Redis so that if any request were to come through, it can check Redis first to see if it already exists and not have to fetch it from BigQuery. This helps make it feel faster to return data. The data is returned with FastAPI in a DictResponse of ('data', 'job_id', 'total_rows', 'page_token').

### FrontEnd/React

The Frontend take the response with TanStack Query and loads it.

AI wrote most of the front end so my knowledge of how the front end works is a but sparse.

There is a loader in TanStack Query that stores the data for the Table and when someone scrolls, it queries the next page. The front end uses the infinite scroll, so it keeps scrolling until there is no more page tokens.

TanStack Query/Table should also be caching the data, so if someone were to revisit my site, it would draw from TanStack Data first, then Redis, then BigQuery.

### Backward Compatibility

To prevent previous paths from breaking, to enable pagination return, I added a flag "use_pagination" that is required on every call to ensure that pagination is used and if it's excluded, then the previous request path is used.

## Results

The results were pretty great! The flashing of "Data Loading" almost never shows up. Every page load is nice and snappy to load. The pagination works really well.

Overall good success!

## Next Steps

There is one enhancement that I would make and that would be adding on the ability to modify the queries for more functionality. Currently, when trying to use the filters on my site for "Price Trends", when you filter to something, it does not pass the filtered parameter to the backend, instead it just filters for what is seen within the TanStack Table data. This causes the data to load jumpily, where it loads a batch of data and then if nothing matches, it had to load yet another page until it gets enough data. This seems like a bad user experience but it only comes up if you start filtering on the data.

Another issue is that when looking at the filters, the data is only filtered to what is currently in the dataset and it may not encompass all the data in the query.

A future improvement would be to make it so that when filters are toggled, they are re-querying BQ for new data that is filtered to the data. I would also want to make it so that the filters have the ability for multi-select and negation of fields.

I would also want to make a change so that the filters are coming from a different API and show all the possible fields instead of relying the current data. This would most likely be another API call unless I can wrap in some metadata return and add in the filters.

## Code Examples

### BigQuery - Initial Query

```python
# Get Job ID for Query
def get_price_trends_paginated(self, category_id, job_id, page_token, sort_by, sort_order, max_results) -> dict[str, Any]:
    """
    Get price trends with pagination
    """

    redis_key = f"price_trends-{category_id}-{job_id}-{max_results}-{page_token}-{sort_by}-{sort_order}"
    if self.redis.exists(redis_key):
        return self.redis.get_data(redis_key)

    if page_token and job_id:
        return self.get_paginated_data(redis_key, job_id, page_token, max_results)

    sort_by, sort_order = self.validate_paginated_sort_and_order_by(sort_by, sort_order)

    query = f"""
        SELECT
            *
        FROM
            `[INSERT TABLE NAME]`
        WHERE
            category_id = @category_id
        ORDER BY
            {sort_by} {sort_order}
    """
    job_config = bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ScalarQueryParameter("category_id", "INT64", int(category_id))
        ]
    )

    query_job = self.client.query(query, job_config=job_config)
    query_job.result()

    job_id = query_job.job_id

    return self.get_paginated_data(redis_key, job_id, page_token, max_results)
```

### BigQuery - Get Paginated Data from Job ID

```python
def get_paginated_data(self, redis_key, job_id, page_token, max_results) -> dict[str, Any]:
    """
    Get data with pagination
    """

    # Get Job by Job ID
    job = self.client.get_job(job_id, location="us-west1")
    destination = job.destination

    rows = self.client.list_rows(destination, max_results=max_results, page_token=page_token)

    data = [dict(x) for x in rows]

    next_page = rows.next_page_token
    total_rows = rows.total_rows

    results = {"data": data, "job_id": job_id, "next_page_token": next_page, "total_rows": total_rows}

    self.redis.set_data(redis_key, results)

    return results
```
