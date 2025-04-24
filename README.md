# Building Cortex Agents Hands-On Lab

This lab will explain step by step how to build a Data Agent using Snowflake Cortex Agents. You will be able to build a Data Agent that is able to use both Structured and Unstructured data to answer sales assistant questions.

First step will be to build the Tools that will be provided to the Data Agent in order to make the work. Snowflake provides two very powerfull tools in order to use Structured and Unstructured data: Cortex Analyst and Cortex Search.

A custom and unique dataset about bikes and ski will be used by this setup, but you should be able to use your own data. This artificial dataset goal is to make sure that we are using data not available within Internet, so no LLM will be able to know about our own data.

First step will be to create your own Snowflake Trial Account (our use the one provided for you during this hands-on lab). Once you have created it, you will be using Snowflake GIT integration to get access to all data that will be needed during this lab.

## Step 1: Setup GIT Integration 

Open a Worksheet, copy/paste the following code and execute all. This will setup the GIT repository and will copy everything you will be using during the lab.

``` sql
CREATE or replace DATABASE CC_CORTEX_AGENTS_SUMMIT;

CREATE OR REPLACE API INTEGRATION git_api_integration
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/ccarrero-sf/')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY git_repo
    api_integration = git_api_integration
    origin = 'https://github.com/ccarrero-sf/cortex_agents_summit';

-- Make sure we get the latest files
ALTER GIT REPOSITORY git_repo FETCH;

-- Setup stage for Bikes Docs
create or replace stage docs_bikes ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = ( ENABLE = true );

-- Copy the docs for bikes
COPY FILES
    INTO @docs_bikes/
    FROM @CC_CORTEX_AGENTS_SUMMIT.PUBLIC.git_repo/branches/main/docs/bikes/
    PATTERN='.*[.]pdf';

ALTER STAGE docs_bikes REFRESH;


-- Setup stage for Ski Docs
create or replace stage docs_ski ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = ( ENABLE = true );

-- Copy the docs for ski
COPY FILES
    INTO @docs_ski/
    FROM @CC_CORTEX_AGENTS_SUMMIT.PUBLIC.git_repo/branches/main/docs/ski/
    PATTERN='.*[.]pdf';

ALTER STAGE docs_ski REFRESH;

```

## Step 2: Setup Tools to be Used by the Agent

We are going to be using a Snowflake Notebook to setup the Tools that will be used by the Snowflake Cortex Agents API. Open the Notebook and follow each of the cells.

Select the Notebook that you have available in your Snowflake account in the Git Repositories:

![image](img/1_create_notebook.png)

Give a name to the notebook and run it in a Container using CPUs.

![image](img/2_run_on_container.png)

Once you have run the entire Notebook and you understand each of the cells, go back to this README to continue building the Cortex Agent.

## Step 3: Explore the Semantic Model to be used by Cortex Analyst Tool

### Open the existing semantic model

Under AI & ML -> Studio, select "Cortex Analyst"

![image](img/3_cortex_analyst.png)

Select the existing semantic.yaml file

![image](img/4_select_semantic_file.png)

### Test the semantic model

You can try some analytical questions to test your semantic file:

- What is the average revenue per transaction per sales channel?
- What products are often bought by the same customers?

### Cortex Analyst and Cortex Search Integration

We are going to explore the integration between Cortex Analyst and Cortex Search to provide better results. If we take a look to the semantic model, click on DIM_ARTICLE -> Dimensions and edit ARTICLE_NAME:

In the Dimention you will see that some Sample values have been provided:

![image](img/5_sample_values.png)

Let's see what happens if we ask for the following question:

- What is the total sales for the carvers?

You may have an answer like this:

![image](img/6_response.png)

Let's see what happens when we integrate the ARTICLE_NAME dimmension with the Cortex Search Service we created in the Notebook (_ARTICLE_NAME_SEARCH).

- Remove the sample values provided
- Click on + Search Service and add _ARTICLE_NAME_SEARCH

It will look like this:

![image](img/7_cortex_search_integration.png)

Click on save, also save your semantic file (top right) and ask the same question again:

- What is the total sales for the carvers?

Notice that now Cortex Analyst is able to provide the right answer even because Cortex Search integration, we asked for "Carvers" but find that the right article to ask for is "Carver Skis":

![image](img/8_right_answer.png)

Now we have the tools ready to create our first App that leverages Cortex Agents API.

## Step 4: Setup Streamlit App that uses Cortex Agents API

Create one Streamlit App that uses the Cortex Agents API.

We are going to leverate this initial code:

[streamlit_app.py](https://github.com/ccarrero-sf/cortex_agents_summit/blob/main/streamlit_app.py)

Click on Projects -> Streamlit -> Streamlit App and "Create from repository":

![image](img/9_create_app.png)

Select the streamlit_app.py file:

![image](img/10_select_file.png)

And click on Create.

## Step 5: Explore the App

Open the Streamlit App and try some of these questions:

### Unstructured data questions

These are questions where the answer would be in the PDF documents. Examples:

- What is the guarantee of the premium bike?
- What is the length of the carver skies?
- What are the tires used by the road bike?

![image](img/10_2_unstructured_questions.png)

Fell free to explore the PDF docs and ask your own questions

### Unstructured data questions

These are analytical questions where answer would be in Snowflake Tables. Examples:

- How much are we selling for the carvers per year for the North region?

Notice that for this query, the 3 tables are used. Also Cortex Search integration in the semantic model understand that the article name is "Carver Skis":

![image](img/11_carver_query.png)

- How much infant bike are we selling per month?
- What are the top 5 customers buying the carvers?

Observe the behavior of the following questions:

- What is the monthly sales via online for the racing fast in central?

Cortex Agents API send the request to Cortex Analyst, but it is not able to filter by customer_region:

![image](img/14_central_question.png)

If we take a look to the [semantic file](https://github.com/ccarrero-sf/cortex_agents_summit/blob/main/semantic_search.yaml) we can see that 'central' is not included as a sample value:

```code
      - name: CUSTOMER_REGION
        expr: CUSTOMER_REGION
        data_type: VARCHAR(16777216)
        sample_values:
          - North
          - South
          - East
        description: Geographic region where the customer is located.
```

This would be a good opportunity to fine tune the semantic model. Either adding all possible values if they are not a lot or use Cortex Search as we have done before for the ARTICLE column.


## Step 6: Understand Cortex Agents API

When calling the Cortex Agents API, we define the tools the Agent can use in that call. You can read the simple [Streamlit App](https://github.com/ccarrero-sf/cortex_agents_summit/blob/main/streamlit_app.py) you setup to understand the basics before trying to create something more ellaborated.

We define the API_ENDPOINT for the Agent, and how to access the different tools we are going to use. In this lab we have two Cortex Search Services to retrieve information from PDFs about bikes and ski, and one Cortex Analyst Service to retrieve analytical information from Snowflake tables.

The Cortex Search Services point to the Services that we created in the Notebook. The Cortex Analyst uses the semantic model we have verified before.

![image](img/12_api_1.png)

Those services are added into the payload we send to the Cortex Agents API. We define the model we want to use to build the final response, the tools to be used and any specific instruction to build the response:

![image](img/13_api_2.png)


## Step 5: Optional: Integrate Cortex Agents API with Slack

[This guide: Getting Started with Cortex Agents and Slack](https://quickstarts.snowflake.com/guide/integrate_snowflake_cortex_agents_with_slack/index.html?index=..%2F..index#0) provides instructions to integrate Cortex Agents with Slack. We are going to debrief it here step by step for the Tools you have created in this hands-on lab.

Follow the instructions from [this repository](https://github.com/ccarrero-sf/cortex_agents_summit_slack)

## Step 6: Optional - Setup Cortex Agents Demo Env

[Cortex Agents in Snowflake](https://github.com/michaelgorkow/snowflake_cortex_agents_demo/) is an excellent Git repository developed by Michael Gorkow that using GIT integration provides an automatic setup to develop several use cases. It will automatically create a Streamlit App where you can select what tools you want to use.

It also include several use cases for you to try. After running this HOL where you should have an understanding or how to use the different tools and APIs, this is a great Demo to run.










