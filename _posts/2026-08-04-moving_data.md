---
layout: post
title: "Moving Data within Google Cloud Platform (GCP)"
author: David Chen
email: david@davidc.dev
date: 2026-08-04
categories: document learnings writeup
---

* TOC
{:toc}

## Data Systems

Google Cloud Platform has some impressive data storage options like Google Cloud Storage, BigTable, AlloyDB, BigQuery and CloudSQL, one of the most open-ended problem would be how do you move data within the ecosystem. There is plenty of scalable options like Pub/Sub or even Datastream where it can do massive writes. 

My goal is to be as cost conscious as possible and see what I can come up with while keeping expenses as low as possible. I do not want to rule out systems like Pub/Sub or Datastream but it feels too much effort for smaller payout and the costs could ballon when it comes to systems uptime.

My daily data needs are about new 200,000 data points a day that should be moved into Google Cloud Storage and inserted into CloudSQL as well. 

I attempted a few different options below but ultimately decided on one:

* CloudSQL - Manual Import from Google Cloud Storage
  * The problem with this option was that there was no way to kind of schedule these commands outside of doing it from the Console UI. I dug into it a little but saw that I would to have to start calling external API (googleapis) endpoint and decided that the OAuth overheard would not be worth it. 
  * This does work well as a one time dump and if the imported data is in CSV and maps directly to the CloudSQL/Postgres Table columns.

* Cloud Scheduler calling Bash Commands or simple Cloud Run Functions
  * I thought I could use the cloud scheduler to call the same commands from Manually Importing the data from CloudSQL but it does not look like it supports any wildcard out any possibility of being able to do an entire folder of exported data.
  * I had to give up this option.

* Cloud Run Polling Worker Pools
  * This was something I thought would be really interesting. I could write something that could pass information to a Pub/Sub event and then eventually this polling worker would get the data and eventually write it into CloudSQL. 
  * The main reason I gave up on this path was the constant up time of a background worker would be in the neighborhood of $100 a month. For the cost, it did not seem like a worthwhile solution. 

* Scheduled Cloud Run Script
  * This is the basic tried and true, Scheduled Python script to write data into system, essentially a dumbed down version of Airflow or Luigi. The reason why I didn't use the actual GCP version of Airflow would be the same of the polling worker, it would cost a ton to have a long lasting instance of Airflow to be constantly up to be able to trigger jobs. 
  * If I were to load the job as a Cloud Run Job, then I can have the pre-built binary loaded into the Artifact Repository and called by the Cloud Scheduler and have it attempted to be run. There is no extra cost or overhead, just the amount used when executing the script. 
  * This is what I chose at the end, because it works and it is very affordable to move data around. If my data ever grows to millions and millions of rows every day, I would look for alternatives but this would suffice for a long time. 

## Cloud Run Job

The Cloud Run Job is pretty simple to make but the sneaky complicated part is the infrastructure configuration when it comes to Security/Access(IAM) and Connections between systems. 

The simplest way to set up a Cloud Run Job is to make a basic python script and then wrap the whole thing with a Dockerfile so that it can be built into a deployable binary on Cloud Run.

The more nuanced way would be to have any deployments from Github be built by Google Cloud Build to build a binary securely into Artifact Registry and use a CloudBuild Yaml file to define the deployment of the Cloud Run Job. This process would make it so that each step would be automated once new code is submitted into Github and the entire process can be automated and deployed for full CI/CD. 

## Infrastructure

Some good to know information about the infrastructure gotcha from Google Cloud Platform.

### Security / Access (IAM)

General Access - When running any code there is a general need to assign specific access to the service account that you are using to run your code. E.g. if you are running CloudSQL, you will need to have the CloudSQL role in IAM as well as the Username and Password for a User in that CloudSQL instance. 

Google Cloud Storage(GCS) Access - I don't believe there is a role for that but there is bucket level permission so that each service account would have to be added to the bucket's permission to do interactions with the data. 

Cloud Build Access - When building with CloudBuild Yaml files, there is a few access things that your service account will need. 
  * When building, if the logs are being written to the cloud only, log editor(Logs Writer) needs to be added. 
  * When writing to the artifact registry, that role(Artifact Registry Repository Administrator) will need to be added as well. 
  * When deploying the Cloud Run binary to replace or deploy a new version, the (Service Account User) role is needed.
  * When interacting with Cloud Run, the (Cloud Run Developer) is needed.

Cloud Run Access - Access is dependant on the service account associated with the Cloud Run Job when it is deployed. To invoke a Cloud Run Jobs, this role (Cloud Run Invoker) is required.

Cloud Scheduler Access - Access is given by this role (Cloud Scheduler Job Runner).

CloudSQL Access - Getting access to CloudSQL is through this role (Cloud SQL Client).

Secretsmanager Access - Granted within the Secretsmanager IAM to each individual login information. 

### Clients / Connections

To be able to connect with other systems, usually just granting the role access is enough but in the case of CloudSQL a bit more work is needed. 

CloudSQL is generally able to connect through multiple means but there is a potentially a lot of configuration issues that could arise between the Pubic vs Private IPs and VLANs. To bypass most of these issues, using a Cloud-SQL-Proxy is preferred and much simpler. 

* Public IPs need to be allow listed, even for access from home network
* Private IPs need to be setup ahead of time and possibly managed through internal VLANs.

When working on any tool that is within GCP that is trying to connect with CloudSQL as long as the proxy has the right role and has the right connection string, it should be able to get access relatively simply. There is 2 flavors of Cloud-SQL-Proxy, TCP or Unix. My preference is for the Unix Socket, slightly complicated connection string but can be used many times. 

### Code Examples

<details>
<summary>Main.py</summary>
<pre><code>

    ``` py
        """Script to move data from Google Cloud Storage to Postgres."""

        import argparse
        import io
        import csv

        from google.cloud import storage
        from postgres_prices import insert_prices


        def read_folder_from_gcs(bucket_name, folder_name):
            storage_client = storage.Client()
            
            bucket = storage_client.get_bucket(bucket_name)
            
            blobs = bucket.list_blobs(prefix=folder_name + "/")

            results = []
            
            for blob in blobs:
                csv_content = blob.download_as_string().decode("utf-8")

                result = list(csv.DictReader(io.StringIO(csv_content)))
                results.extend(result)

            return results

        def move_prices():
            """Daily script to move prices from GCS to Postgres."""

            bucket = "[BUCKET_NAME]"
            folder = "[FOLDER_NAME]"

            prices = read_folder_from_gcs(bucket, folder)

            try:
                insert_prices(prices)
                print(f"Successfully inserted {len(prices)} prices")
            except Exception as e:
                print(f"Error inserting prices: {e}")

        def main():
            """Main entry point for the script."""

            parser = argparse.ArgumentParser()
            parser.add_argument("action", choices=["move_prices"], help="Action to perform")
            args = parser.parse_args()

            if args.action == "move_prices":
                print("Moving prices...")
                move_prices()
            else:
                print(f"Unknown action: {args.action}")

        if __name__ == "__main__":
            main()

    ```
</code></pre>
</details>

<details>
<summary>Dockerfile</summary>
<pre><code>

    ``` dockerfile
        FROM python:3.14-alpine

        RUN mkdir -p /usr/src/app
        WORKDIR /usr/src/app

        COPY requirements.txt /usr/src/app/
        RUN pip install --no-cache-dir -r requirements.txt

        COPY . /usr/src/app

        ENTRYPOINT ["python", "-u", "./main.py"]

        # Default command to run when the container starts
        CMD ["move_prices"]
    ```
</code></pre>
</details>

<details>
<summary>CloudBuild Yaml</summary>
<pre><code>

    ``` yaml
        steps:
        # Step 1: Build the Docker container image
        # We tag the image with both the unique git commit SHA and the 'latest' tag
        - name: "gcr.io/cloud-builders/docker"
            id: "Build Image"
            args:
            - "build"
            - "-t"
            - "${_GAR_LOCATION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_IMAGE_NAME}:$COMMIT_SHA"
            - "-t"
            - "${_GAR_LOCATION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_IMAGE_NAME}:latest"
            - "."

        # Step 2: Push the container image to Google Artifact Registry
        - name: "gcr.io/cloud-builders/docker"
            id: "Push Image (Commit SHA)"
            args:
            - "push"
            - "${_GAR_LOCATION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_IMAGE_NAME}:$COMMIT_SHA"

        - name: "gcr.io/cloud-builders/docker"
            id: "Push Image (Latest)"
            args:
            - "push"
            - "${_GAR_LOCATION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_IMAGE_NAME}:latest"

        # Optional Step 3: Automatically deploy/update a Cloud Run Job
        # This is commented out by default. If you want to automatically update your Cloud Run Job,
        # ensure your Cloud Build Service Account has "Cloud Run Developer" and "Service Account User" roles,
        # then uncomment this step.
        - name: "gcr.io/google.com/cloudsdktool/cloud-sdk"
            id: "Deploy Cloud Run Job"
            entrypoint: "gcloud"
            args:
            - "run"
            - "jobs"
            - "deploy"
            - "[JOB_NAME]"
            - "--image=${_GAR_LOCATION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_IMAGE_NAME}:$COMMIT_SHA"
            - "--region=${_GAR_LOCATION}"

        # Images specified here will be automatically pushed and tracked in the build results
        images:
        - "${_GAR_LOCATION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_IMAGE_NAME}:$COMMIT_SHA"
        - "${_GAR_LOCATION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_IMAGE_NAME}:latest"

        # Substitutions allow parameterizing the build config
        substitutions:
        _GAR_LOCATION: "[JOB_LOCATION]"
        _REPOSITORY: "[REPO_NAME]"
        _IMAGE_NAME: "[IMAGE_NAME]"

        # Options configure logging and other behavior
        options:
        logging: CLOUD_LOGGING_ONLY
    ```
</code></pre>
</details>


## End to End Process

The assumption is that data from other scripts would load external data into Google Cloud Storage and run BigQuery scripts to analyze and export the data back into Google Cloud Storage so it can be loaded into CloudSQL. There is a possibility that I could have the script read directly from BigQuery through the client API but wanted to write to Google Cloud Storage mainly just cause but also so that there is a backup manual option to insert data with the CloudSQL CSV Import.

The Cloud Run Job would read all the CSV files in the GCS folder and load it into memory. Then the script would attempt to load the data into CloudSQL by chunking it into 20,000 rows at a time. The table that the data is loaded into is a staging table where the data is truncated each time. Postgres has the ability to do merge a staging table and production table together without any data being destructed in production. I chose this way so that if anytime the script were to fail, there would always be data in the production table, it may be stale but it would not leave the API serving nothing.

## Next Steps

The next step that I would take would be to coordinate all the jobs through the use of Cloud Workflow where I can orchestrate all the GCP scripts as I want to. This way if I need to do a whole end to end deployment, I can do it with one script where as I currently have to do it with multiple scripts. 

The current method works but will not really scale when I have more and more scripts that can't be combined together. I would consider the script mature once I can get everything working in one simplified workflow. 