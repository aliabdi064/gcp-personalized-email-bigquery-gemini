# # Lab GSP1284: Generating Personalized Email Content with BigQuery Continuous Queries and Gemini




## Overview

This repository documents my work and learnings from the Google Cloud lab GSP1284. The lab focuses on building a real-time customer engagement system to send personalized emails to customers who have abandoned their shopping carts. This is achieved by leveraging BigQuery continuous queries to monitor cart data, using Gemini to generate tailored email content, and integrating Pub/Sub and Application Integration for message delivery.





## Technologies Used

* **Google Cloud Platform (GCP)**
    * BigQuery:
        * Continuous Queries
        * BigQuery ML (for remote model with Gemini)
        * SQL for data manipulation and querying
    * Vertex AI Gemini API (specifically Gemini 2.0 Flash model)
    * Pub/Sub: For event-driven messaging
    * Application Integration: To orchestrate the workflow and send emails
    * IAM (Identity and Access Management): For configuring service account permissions
 




## What I Learned / Key Steps Performed

This lab provided hands-on experience with building an event-driven, AI-powered marketing solution. Key steps included:

1.  **Task 1: Create and Configure a BigQuery ML Remote Model**
    * Created a BigQuery remote connection to Vertex AI.
    * Granted necessary IAM roles (Vertex AI User) to the BigQuery service account.
    * Defined a BigQuery ML remote model pointing to the `gemini-2.0-flash-001` endpoint to enable generative AI capabilities directly within BigQuery.
    * *Learning:* Understood how to integrate external AI models like Gemini with BigQuery for powerful in-database machine learning and content generation.

2.  **Task 2: Grant Custom Service Account Access**
    * Granted a pre-created custom service account (`bq-continuous-query-sa`) appropriate permissions (BigQuery Connection User, BigQuery Data Editor, Pub/Sub Viewer, Pub/Sub Publisher) to interact with necessary BigQuery and Pub/Sub resources.
    * *Learning:* Reinforced the importance of the principle of least privilege and how to manage service account access for different GCP services.

3.  **Task 3: Create and Configure an Application Integration Trigger**
    * Set up an Application Integration flow triggered by messages on a Pub/Sub topic (`recapture_customer`).
    * Configured data mapping to extract customer details and the Gemini-generated message from the Pub/Sub message.
    * Added a "Send Email" task to dispatch the personalized email.
    * *Learning:* Gained insight into using Application Integration as an iPaaS solution for connecting services and automating workflows, specifically for acting on Pub/Sub events.

4.  **Task 4: Create a Continuous Query in BigQuery**
    * Set up a BigQuery Enterprise reservation to enable continuous queries.
    * Wrote and deployed a BigQuery continuous query that:
        * Monitors the `abandoned_carts` table for new entries using `APPENDS` Table-Valued Function (TVF).
        * Uses `ML.GENERATE_TEXT` to call the Gemini remote model with a dynamically constructed prompt based on customer and product data.
        * Formats the output (customer details and generated email HTML) as a JSON string.
        * Exports this JSON payload to the `recapture_customer` Pub/Sub topic.
    * *Learning:* Mastered the concept and implementation of BigQuery continuous queries for real-time data processing and how to invoke ML models within these queries to generate dynamic content.

5.  **Task 5: Test the Continuous Query**
    * Inserted sample data into the `abandoned_carts` table.
    * Observed the end-to-end flow: data insertion triggering the continuous query, Gemini generating content, Pub/Sub receiving the message, and Application Integration (theoretically) sending the email.
    * *Learning:* Validated the entire pipeline and understood how to test event-driven architectures.
  





## Key Code Snippets

Below are some representative code snippets from the lab (ensure all sensitive information like Project IDs are genericized if you adapt these):

**1. Creating the BigQuery ML Remote Model:**
```sql
-- Create a BigQuery ML remote model named gemini_2_0_flash
CREATE MODEL `[YOUR_PROJECT_ID].continuous_queries.gemini_2_0_flash`
REMOTE WITH CONNECTION `[YOUR_REGION].continuous-queries-connection`
OPTIONS(endpoint = 'gemini-2.0-flash-001');
--
..................................................................................................................
**2. The Core Continuous Query:**



EXPORT DATA
 OPTIONS (format = CLOUD_PUBSUB,
 uri = "[https://pubsub.googleapis.com/projects/](https://pubsub.googleapis.com/projects/)[YOUR_PROJECT_ID]/topics/recapture_customer")
AS (
  SELECT
   TO_JSON_STRING(
     STRUCT(
       customer_name AS customer_name,
       customer_email AS customer_email,
       REGEXP_REPLACE(REGEXP_EXTRACT(ml_generate_text_llm_result,r"(?im)\<html\>(?s:.)*\<\/html\>"), r"(?i)\[your name\]", "Your friends at AI Megastore") AS customer_message
     )
   ),
 FROM
   ML.GENERATE_TEXT(
     MODEL `[YOUR_PROJECT_ID].continuous_queries.gemini_2_0_flash`,
     (
       SELECT
         customer_name,
         customer_email,
         CONCAT(
           "Write an email to customer ",
           customer_name,
           ", explaining the benefits and encouraging them to complete their purchase of: ",
           products,
           ". Also show other items the customer might be interested in. Provide the response email in HTML format."
         ) AS prompt
       FROM
         APPENDS(
           TABLE `[YOUR_PROJECT_ID].continuous_queries.abandoned_carts`,
           CURRENT_TIMESTAMP() - INTERVAL 10 MINUTE -- Example start time
         )
     ),
     STRUCT(
       1024 AS max_output_tokens,
       0.2 AS temperature,
       1 AS candidate_count,
       TRUE AS flatten_json_output
     )
   )
);



................................................................


## Potential Extensions
* Integrating with a CRM for more comprehensive customer data.
* Adding A/B testing for different email prompts.
* Implementing more sophisticated analytics on email open rates and conversion.



