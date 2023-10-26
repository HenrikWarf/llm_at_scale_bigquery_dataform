# Using Large Language Models at Scale with Google BigQuery, Dataform and Vertex AI

## **Introduction**

Generative AI and Large Language Models(LLMs) are of course the big hype right now. I meet with organisations and have conversations daily around use cases and implementations around the topic. The use cases are different in-between organisations and industries ranging from virtual agents and search capabilities to content creation for product and marketing purposes as well automating tasks such as summarisation, classification, sentiment and entity extraction (+ much more).

Generating one piece of content from one of these LLMs are pretty straight forward if one has understood the ins and outs of prompt engineering. On the other hand it is usually a lot harder to do when you have a use case that demands 100s or 1000s of tasks that needs to be done on a large number of unique data points.

This is usually when you start to look how to programatically approach the problem with the goal to automate the process.

Thats is also what I will do here but from the view point of a data engineer using data engineering tooling and using SQL as the only “programming” language. The technology that makes this possible is BigQuery in Google Cloud. BigQuery is Google Clouds enterprise data warehouse that also sits at the center of Google Data Cloud strategy.

In recent years BigQuery has expanded its features and functions quite a bit and one of those features is the possibility to store and analyse unstructured data through the use of integration with Vertex AI, Google Clouds Machine Learning Platform. You can now call models stored in Vertex AI directly from BigQuery using SQL and run inference on top of your data and store the results in back in BigQuery.

If we combine this function with the use of Dataform to orchestrate a data pipeline (sql workflow) we can automate the process of using Google’s foundational Large Language Models at Scale. Let’s go through the process of building this with a demonstrative use case. Running sentiment analysis on top of movie reviews available in the IMDB public data set, also available within BigQuery.

I will not go into the exact exact details of setting this up but will cover it to a fairly good degree. On the other hand focus is to introduce the services that is being used to make it happen. You can also find the code base for the pipeline [here](https://github.com/HenrikWarf/llm_at_scale_bigquery_dataform) with can be used as a baseline template to get started a bit more quickly using Dataform, BigQuery and Vertex AI.

**Content:**

* Preparation

* Using Large Language Models in Vertex AI

* Dataform Introduction

* Creating a dataform pipeline

* Using LLMs from Vertex AI

* Conclusions and Summary

## Preparations

To make this work in your environment you need to go through these steps beforehand.

1. In the [Google Cloud Console](https://console.cloud.google.com/), on the project selector page, select or create a Google Cloud [project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)

1. Make sure that billing is enabled for your Cloud project. Learn how to [check if billing is enabled on a project](https://cloud.google.com/billing/docs/how-to/verify-billing-enabled)

1. Make sure all the necessary APIs (BigQuery API, Vertex AI API, BigQuery Connection API, Dataform API, Secret Manager API) are [enabled](https://console.cloud.google.com/flows/enableapi?apiid=bigquery.googleapis.com,bigqueryconnection.googleapis.com,aiplatform.googleapis.com&_ga=2.132962701.243207769.1688884437-279425947.1688884437)

1. Create a connection to an external data source in BigQuery (see instructions below)

1. Grant Permissions to use Vertex AI from BigQuery (see instructions below)

1. Generate a personal access token from GitHub ([instructions](https://cloud.google.com/dataform/docs/connect-repository)). This needs to be done for Dataform to authenticate against GitHub. Skip this if you do not need to use a remote repo to store your pipeline code.

**External Connection (step 4)**

Create an External Connection (Enable BQ Connection API if not already done) and note down the Service Account id from the connection configuration details:

* Click the +ADD button on the BigQuery Explorer pane (in the left of the BigQuery console) and click “Connection to external data sources” in the popular sources listed

* Select Connection type as “BigLake and remote functions” and provide “llm-conn” as Connection ID

* Once the connection is created, take a note of the **Service Account generated from the connection configuration details**

![](https://cdn-images-1.medium.com/max/2000/0*x7f5R88IRMuB35yC.png)

**Grant Permissions (Step 5)**

In this step we will grant permissions to the Service Account to access the Vertex AI service:

Open IAM and add the Service Account you copied after creating the external connection as the Principal and select “Vertex AI User” Role.

![](https://cdn-images-1.medium.com/max/2000/1*QMUWgA7y6P1wE60JAhhNZQ.png)

## Using Large Language Models in Vertex AI

Google Cloud’s language models are available within the Generative AI Studio inside the Vertex AI service.

![A look inside Vertex AI Generative AI Studio](https://cdn-images-1.medium.com/max/4800/1*HXsiXC4-aKqcAAX-u5L_uQ.png)*A look inside Vertex AI Generative AI Studio*

The quickest way to using one of our text model is to go directly to the Generate Text box and click on TEXT PROMPT. You will then get to a screen that looks like this.

![UI for Text generation inside Generative AI Studio](https://cdn-images-1.medium.com/max/5568/1*RbzM1EJEzmLkjXT7RQZZbg.png)*UI for Text generation inside Generative AI Studio*

Here you are able to add a prompt of your liking related to your use case. In our case I have added the prompt I will use for the inference in our soon to be created pipeline. In this case I also added one movie review in the prompt to get the sentiment from.

Running this gets us this response.

![](https://cdn-images-1.medium.com/max/3188/1*geGdZibc4MM5_bqXby0BAg.png)

*Prompt design* is the process of creating prompts that elicit the desired response from language models. Writing well structured prompts is an essential part of ensuring accurate, high quality responses from a language model.

If you need to understand this concept a bit more this is a page that introduces some basic concepts, strategies, and best practices to get you started in designing prompts ([https://cloud.google.com/vertex-ai/docs/generative-ai/learn/introduction-prompt-design](https://cloud.google.com/vertex-ai/docs/generative-ai/learn/introduction-prompt-design)).

The reference page above also goes into the more advanced settings you can see on the right hand side of the prompt box such as temperature, top K, top P etc.

**Easy right ?**

Getting one task done with a LLM is as you can se very straight forward in Vertex AI. Running this for +1000's or +10000's of reviews on the other hand can not be done by using the UI like this where you would need to exchange the movie review for each of the movies and also manually store the results somewhere.

This is where Google Cloud’s Data Cloud services can help. Lets now go through the setup needed starting with Dataform.

## Dataform Introduction

Dataform is a fully managed service that helps data teams build, version control, and orchestrate SQL workflows in BigQuery. It provides an end-to-end experience for data transformation, including:

![Dataform directly integrated into BigQuery](https://cdn-images-1.medium.com/max/2000/1*eVUYj8i0kAzAm375mLuTXg.png)*Dataform directly integrated into BigQuery*

* Table definition: Dataform provides a central repository for managing table definitions, column descriptions, and data quality assertions. This makes it easy to keep track of your data schema and ensure that your data is consistent and reliable.

* Dependency management: Dataform automatically manages the dependencies between your tables, ensuring that they are always processed in the correct order. This simplifies the development and maintenance of complex data pipelines.

* Orchestration: Dataform orchestrates the execution of your SQL workflows, taking care of all the operational overhead. This frees you up to focus on developing and refining your data pipelines.

Dataform is built on top of Dataform Core, an open source SQL-based language for managing data transformations. Dataform Core provides a variety of features that make it easy to develop and maintain data pipelines, including:

* Incremental updates: Dataform Core can incrementally update your tables, only processing the data that has changed since the last update. This can significantly improve the performance and scalability of your data pipelines.

* Slowly changing dimensions: Dataform Core provides built-in support for slowly changing dimensions, which are a common type of data in data warehouses. This simplifies the development and maintenance of data pipelines that involve slowly changing dimensions.

* Reusable code: Dataform Core allows you to write reusable code in JavaScript, which can be used to implement complex data transformations and workflows.

Dataform is integrated with a variety of other Google Cloud services, including GitHub, GitLab, Cloud Composer, and Workflows. This makes it easy to integrate Dataform with your existing development and orchestration workflows.

Benefits of using Dataform in Google Cloud

There are many benefits to using Dataform in Google Cloud, including:

* Increased productivity: Dataform can help you to increase the productivity of your data team by automating the development, testing, and execution of data pipelines.

* Improved data quality: Dataform can help you to improve the quality of your data by providing a central repository for managing table definitions, column descriptions, and data quality assertions.

* Reduced costs: Dataform can help you to reduce the costs associated with data processing by optimizing the execution of your SQL workflows.

* Increased scalability: Dataform can help you to scale your data pipelines to meet the needs of your growing business.

Use cases for Dataform

Dataform can be used for a variety of use cases, including:

* Data warehousing: Dataform can be used to build and maintain data warehouses that are scalable and reliable.

* Data engineering: Dataform can be used to develop and maintain data pipelines that transform and load data into data warehouses.

* Data analytics: Dataform can be used to develop and maintain data pipelines that prepare data for analysis.

* Machine learning: Dataform can be used to develop and maintain data pipelines that prepare data for machine learning models.

## Creating a Dataform Pipeline

![Dataform Pipeline to use LLMs in Vertex AI.](https://cdn-images-1.medium.com/max/4484/1*k6tgEqWEeB6-VGlniP2vBQ.png)*Dataform Pipeline to use LLMs in Vertex AI.*

First step in implementing a pipeline in Dataform is to set up at repository and a development environment. Detailed quickstart and instructions can be found [here](https://cloud.google.com/dataform/docs/quickstart-create-workflow).

To use the code I have created you have the options of either copy the code and enter it “manually” into files in your development environment or you can clone the git repo and then create a new GitHub repository that is under your control. This repository can then be linked to Dataform and used to update the pipeline. Instructions on how to link your environment to a third-party Git repository can be found [here](https://cloud.google.com/dataform/docs/connect-repository).

You should also set up a workspace compilation override. This will override the target project ID, table prefix, and schema suffix settings for **manual** executions of all workspaces in the repository. The default settings are stored in `dataform.json`. Learn more about [workspace compilation overrides ](https://cloud.google.com/dataform/docs/workspace-compilation-overrides). Basically when executing your pipeline manually you will do this separately in either a different project, dataset or table used for testing and development. In my set up I added a Schema suffix = development. This will create a new development dataset where your tables are stored named [*project_name]_development.*

![Schema suffix through the use of workspace compilation override.](https://cdn-images-1.medium.com/max/2000/1*ZSI1Rn1DOsMNlMAtL22omA.png)*Schema suffix through the use of workspace compilation override.*

Let us now focus on the pipeline and how it is structured so you understand how to build it yourself.

**The Pipeline (SQL workflow)**

![File structure for the LLM pipeline.](https://cdn-images-1.medium.com/max/2000/1*vJrWf282saJXoX8QgC9Riw.png)*File structure for the LLM pipeline.*

Above you see the file structure in your workspace after setting up the code from my repository.

The first thing to do is to edit the package.json and dataform.json so it will work for your environment.

**Exchange the name in package.json to a name of your choice.**

    {
      "name": "name your project here",
      "dependencies": {
        "@dataform/core": "2.4.2"
      }
    }

**Enter information into dataform.json.**

The dataform.json file configures basic settings required to compile your Dataform project:

* warehouse: must be set to bigquery

* defaultDatabase: Your Google Cloud Project ID in which Dataform creates assets.

* defaultSchema: The BigQuery dataset in which Dataform creates assets.

* defaultLocation: Your default BigQuery dataset location.

* assertionSchema: The BigQuery dataset in which Dataform creates views with assertion results.

    {
      "defaultSchema": "",
      "assertionSchema": "",
      "warehouse": "bigquery",
      "defaultDatabase": "",
      "defaultLocation": ""
    }

After editing these two files it could be a good idea to commit and push your code to your git repository. For more details regarding this topic see the documentation [here](https://cloud.google.com/dataform/docs/version-control).

The pipeline (SQL workflow) will be built from the files available in the definitions folder. First file to look at is** ingest_raw.sqlx.**

    -- Igesting raw reviews from Public Dataset IMDB - Reviews
    config {
      type: "incremental", 
      columns: {
        review: "User review's in IMDb.", 
        split: "It has two categories test and train.",
        label: "It has three categories Negative, Positive and Unsupervised. All Unsupervised label has only split equals-to train.",
        movie_id: "UniqueId for the movie in IMDb.",
        reviewr_rating: "Reviewer rating for particular movie in IMDb. For train-unsupervised, reviewer_rating is NULL.",
        movie_url: "Movie url for corresponding movie_id",
        title: "Title of the movie for corresponding movie_id"
      }
    }
    
    SELECT * 
    FROM bigquery-public-data.imdb.reviews

Every file should start with a *config* block to define what type of entity you are creating as well as different options you can add when creating the entity in question. In this case it is of the type “incremental”, which stands for a incremental table. Other options include *table, view* and *materialized*. More info around tables and what options are available can be found [here](https://cloud.google.com/dataform/docs/tables).

Under *columns* you have the ability to define and describe the columns in the table for documentation purposes. This is also pushed to BigQuery.

After the config block you then write your SQL code to be executed in BigQuery. In our case its just extracting all data from the public dataset and table *bigquery-public-data.imdb.reviews.*

Second file is **staging_data.sqlx**

    -- Staging a sample set of reviews from ingest_raw incremental table
    config {
      type: "table", 
      columns: {
        review: "User review's in IMDb.", 
        reviewr_rating: "Reviewer rating for particular movie in IMDb. For train-unsupervised, reviewer_rating is NULL.",
        title: "Title of the movie for corresponding movie_id"
      }
    }
    
    SELECT review, reviewer_rating, title 
    FROM ${ref('ingest_raw')}
    TABLESAMPLE SYSTEM (0.0001 PERCENT)
    LIMIT 10

The big difference here is that we are referencing another table via this piece of code here: FROM ${ref(‘ingest_raw’)} . This is also a way for you to handle dependencies in Dataform. Besides this we use the format of “table” instead of “incremental” to create a normal table in BigQuery.

Below code snippet just does a random selection of records from the raw table. Be careful to exclude this as running this over the whole table will be a very long running job. Now its more like a 20 second pipeline run that will fetch 10 records.

    TABLESAMPLE SYSTEM (0.0001 PERCENT)
    LIMIT 10

Now we are ready to inspect how to use the LLMs available in Vertex AI directly on top of the data that we have in the table *staging_data* in BigQuery using the remote connection we set up in the preparations section.

## Using LLMs from Vertex AI

Now we need to go through setting up the connection with the LLMs that we have available in the Vertex AI platform. In the preparation step we already made the connection to Vertex AI through the addition of a external data source. Now we just need to create a model connection in BigQuery. This is also something we can do within our pipeline in parallel to ingesting data and handling the staging stage.

![Dataform Pipeline to use LLMs in Vertex AI.](https://cdn-images-1.medium.com/max/4484/1*k6tgEqWEeB6-VGlniP2vBQ.png)*Dataform Pipeline to use LLMs in Vertex AI.*

First we have create_dataset.sqlx where we just perform a CREATE SCHEMA operation. This operation creates a dataset entity where we can store our model connection separately from the data.

    config {
        type : "operations"
    }
    
    CREATE SCHEMA IF NOT EXISTS llm_model
      OPTIONS (
        description = 'Dataset to store LLM models used for LLM usecases_01',
        location = 'US'
      )

An “operation” in Dataform is a custom SQL query where you can, as in this example create a empty dataset that can be used downstream.

Next file *llm_model_connection.sqlx *we need for the actual creation of the connection to the LLM in Vertex AI. This is again done using a config type of “operation”. Something we add to the config is also a defined dependency to the previous step of creating a empty dataset. As we need this to set up the connection.

After the configurations is set up we use a BigQuery function to CREATE a new model by referencing the connection we set up in the preparation and adding a remote _service_type in the options section.

    config {
      type: "operations",
      dependencies: ["create_dataset"]
    }
    
    CREATE OR REPLACE MODEL llm_model.llm_vertex_model
      REMOTE WITH CONNECTION `us.llm-conn`
      OPTIONS (remote_service_type = 'CLOUD_AI_LARGE_LANGUAGE_MODEL_V1');

Link to an overview of the available [foundational models in Vertex AI.](https://cloud.google.com/vertex-ai/docs/generative-ai/learn/models)

When this is done we are ready to call the model directly from BigQuery and do inference on the reviews that we have ingested and stored in the staging table.

This is performed in the *sentiment_inference.sqlx* file.

    config { 
        type: "table" ,
        dependencies: [ "llm_model_connection" ]
    }
    
    SELECT
        REGEXP_REPLACE(ml_generate_text_llm_result, ' ', '') AS ml_generate_text_llm_result, 
        * EXCEPT (ml_generate_text_llm_result, ml_generate_text_status)
    FROM
      ML.GENERATE_TEXT(
        MODEL `llm_model.llm_vertex_model`,
        (
      SELECT
            CONCAT('''Tell me whether the following movie review sentiment is positive or negative or neutral. 
            - Use only the words positive, negative and neutral. 
            - Remove all spaces in the response. 
            - Answer in lower letters.
            Move Review:
            ''', 
            review
            )
            AS prompt, 
            reviewer_rating,
            review as original_review
            FROM ${ref('staging_data')}
        ),
        STRUCT(
          0.1 AS temperature,
          20 AS max_output_tokens,
          TRUE AS flatten_json_output))

The inference query is a bit longer so lets take one part at a time.

**Config: **we set type to *table* and add a dependency to the *llm_model_connection* step so we know it is connected before running the inference.

**SELECT: **I chose to select** **everything expect *ml_generate_text_status *from the reponse back from the ML.GENERATE_TEXT function. I also added a regexp expression here to highlight the fact that you are able add this to a possible reponse in natural language as a formatting function to get at response you would like in the created table.

**FROM: **ML.GENERATE_TEXT is the build in function that we should use to run inference on one of our LLMs in Vertex AI. In the function we first reference the model to use *`llm_model.llm_vertex_model` *and then add a SELECT statement to both package the prompt to use but also if we want to get back other columns from the *staging_data *table. In my case here I add *reviewer_rating* and *orginal_review*.

**PROMPTING: **Prompting is done in a way where you concatenate a defined column and its values with a written prompt in the SELECT statement. In my case here I structured a prompt like this:

    Tell me whether the following movie review sentiment is positive 
    or negative or neutral. 
      - Use only the words positive, negative and neutral. 
      - Remove all spaces in the response. 
      - Answer in lower letters.
    Move Review:

The actual review is then added to the prompt above and used to do inference for each record in BigQuery and generates a request to our model in Vertex AI for all the selected rows/records.

**STRUCT: **The last part of the ML.GENERATE_TEXT function is a STRUCT statement where you can add some options to your inference request to the model. I added som parameters such as temperature, max_output_tokens and flatten_json_output to my request.

[Read this documentation ](https://cloud.google.com/bigquery/docs/generate-text)for a more detailed description of the different options using the ML.GENERATE_TEXT function.

**Ok so now we are sort of done.** But how does the output look like. Here you see a couple of examples from the output that is stored in the newly created table *sentiment_inference. *Most importantly we have a result in the first column with a inferred sentiment. In this case two negative ones. You also get back our responsible AI results with different scores related to safety attributes and the concatenated prompt.

![Output from the Sentiment Analysis using Vertex AI LLM models.](https://cdn-images-1.medium.com/max/4476/1*Jj-D0ja8OdDuXd2ga5sQRA.png)*Output from the Sentiment Analysis using Vertex AI LLM models.*

The table structure after running the pipeline will then look like this.

![](https://cdn-images-1.medium.com/max/2000/1*OLi4lDjpUsov5YQnQs7iPg.png)

**Production, Scheduling and Automation**

Last thing to cover is how you automated and schedule the compilation and execution of the pipeline. This is done through what is named release configurations and workflow configurations.

*Release configurations* lets you compile your pipeline code at a certain intervall relevant for your use case. The way you do this is to define a branch, tag or commit SHA to use and then set a frequency such as daily or weekly etc. Release configurations also come with *compilation overrides* such as those we introduced for testing and development. Here you again define if you want this pipeline to be executed in a complitely isolated and separat project or dataset/table. Common practices could be to set up release configurations that possibly represent test and production environments. [More information about release configurations](https://cloud.google.com/dataform/docs/release-configurations).

A simple concrete example here would be to use the **main branch** and **daily frequency** for compilation of your production code with a separate project and dataset as your production environment. After this is set up you would only need to worry about pushing code to the main branch and the service would compile the code for you at a daily frequency making sure it works and could be executed.

![](https://cdn-images-1.medium.com/max/2232/1*fdO5alq1fLweuc4L5bedqQ.png)

To actually execute a pipeline according to your specification and code structure you would need to set up a *workflow configuration*. This is basically a scheduler where you define what release configuration to use, what frequency and what actions to execute. The pipeline would then run a the defined frequency and using the compiled code from the defined release configuration. [More information about workflow configurations.](https://cloud.google.com/dataform/docs/workflow-configurations)

And we are done!

![](https://cdn-images-1.medium.com/max/2252/1*RJ-ToCg123AWaMPt0QpK-g.png)

## Conclusions and Summary

In this post we have worked our way through setting up a data pipeline using only SQL to run reviews through a large language model (LLM) to identify the sentiment of the said review.

Data is stored in a structured format from start to stop and no data needs to leave the data warehouse. The results could easily then be connected to a BI solution such as Looker to analyze the results further.

The same pipeline could also be used for many more use cases by just changing the instructions(prompt) to the LLM. Classification, topic extraction, summarisation, text generation, data engineering tasks etc.

Using this approach could be useful when you have a lot of records you need to expose to the power of LLMs for mentioned use cases above. It can also be a tool used by data engineering professionals to enhance, generate or transform data in their common data pipelines.

The main services used from Google Cloud to do all this are: BigQuery as data warehouse with Dataform to create the SQL workflows/pipelines and Vertex AI to serve the LLM to do inference against.

Please check out my git repository with the code to implement this solution and try it out in your environment. Advise would be to experiment in exchanging the data input (google public data sets) as well as the prompt instructions to the LLM.

Thank you for reading and wish you all the best :-)

