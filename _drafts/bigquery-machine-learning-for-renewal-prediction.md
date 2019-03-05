---
layout: post
title: A Hands-On Example with Looker & BQML - Predicting SaaS Renewals
categories: Data
---

As a Customer Success Engineer at Looker, in addition to guiding customers on their data architecture, I also regularly build out technical solutions for our internal customer success functions. The one I'm going to be sharing today is our predictive renewal score, and the technologies I'll be demonstrating are Looker and BigQuery's Machine Learning service, which combine to make this end-to-end solution super fast to build out, enhance, and maintain.

So, what is this predictive renewal score? A score from 0 to 100% that estimates the likelihood that a given customer will renew. In practice, it actually has a few distinct functions within our organization - 

 - **Summarizing score** - When a Customer Success Manager or Account Executive wants to get a feel for a specific account, this is a composite score that can help them get a quick first sense for where the customer is overall, without having to individually consider and integrate dozens of aspects of the customer's profile.
 - **Prioritization feature** - When a Customer Success Manager wants to proactively allocate their time within a pool of accounts, they can use this score in a first pass over a list to help in prioritizing.
 - **Forecasting input** - Since the score represents a probability of renewing, we can use it for forecasting. We simply multiply it by the account's Annual Contract Value and sum up these products to get the total expected dollars renewing across a set of accounts. We can slice and dice this by account owners, account segments, regions, etc.

Hopefully, you've gotten a sense for how valuable this is, and see some similar opportunities in your own organization. Let's dive in to how to build it!

### LookML for Structure

We'll be using LookML to keep everything organized. In my case, this solution exists inside of a much larger LookML project, and is relatively stand-alone, so I'm going to be putting all of these into one file to keep the larger project tidy. In addition, I've prefixed all the views, to make them play more nicely within the larger project namespace. Finally, I'm using primary key naming conventions from my [LookML Style Guide](https://looker-open-source.github.io/look-at-me-sideways/rules.html) to help make the join logic more transparent and reliable.

### What Do We Want?!

So, let’s start by defining what we’ll be predicting. In our case, we’re using Saleforce opportunities, and specifically ones with type “Renewal”. A simple view will help us organize this dataset of interest. We'll start `FROM salesforce.opportunity`, apply a WHERE filter on `type='Renewal'` and within a certain timeframe, and convert the `stage_name` to a binary outcome. 

<details><summary> See the code - Objectives </summary>

```sql
view: prs_objectives {
  derived_table: {
    sql:
        SELECT
          opp.id as pk1_opportunity_id,
          ---
          start_date as date,
          opp.account_id as entity_id, --See note #1
          CASE opp.stage_name
              WHEN 'Closed Won' THEN 1
              WHEN 'Closed Lost' THEN 0
              ELSE NULL
              END as result
        FROM `salesforce.opportunity` as opp
        CROSS JOIN UNNEST([ --See note #2
            COALESCE(opp.start_date__c, opp.close_date)
            ]) as start_date 
        WHERE opp.type='Renewal'
          AND start_date >= DATE_ADD(CURRENT_DATE(), INTERVAL 0-(2*365) DAY)
          -- Taking renewals up to two years in the past
          AND DATE_DIFF(start_date, CURRENT_DATE(), MONTH)<>0
          -- ^ Current month neither needs prediction, nor is settled enough to use for training 
    ;;
  }
  dimension: pk1_opportunity_id {hidden:yes}
  dimension: date {hidden:yes}
  dimension: entity_id {hidden:yes}
  dimension: result {hidden:yes description: "The objective of the prediction, either a 0 or 1."}
}
```

**Notes for the above query**

    <ol>
    <li>I've chosen to alias the account_id as "entity_id" to describe the abstract function for which we are using the account ID here. Namely, multiple opportunities under an account will care about the history of events for the account/entity as a whole, even if there was another opportunity recently. This abstraction should help apply this pattern to other datasets.</li>
    <li>`CROSS JOIN UNNEST ([expression]) as alias` is a bit confusing to read at first. Technically, it's joining for each row on the left of the join a single-row, single-column "table" defined by the expression. In practice, it's essentially creating an alias or projection which can be reused without writing out the whole expression. (This is similar to a LATERAL JOIN in other dialects)</li>
    </ol>

</details>

### When Do We Want It?!

With the scope of our problem defined, I'll need to take a brief aside into when predictions are made and how this time element interacts with the prediction itself.

Since our organization operates on yearly contracts, the opportunities that we want to predict are relatively sparse. We'll definitely want to be updating our predictions multiple times over the course of a year for any given opportunity, rather than having one specific prediction per opportunity.

In our case, many of our business operations are aligned to months, so we'll align with that too to avoid fatiguing or distracting users with super frequent updates. That means that each month, we'll be generating a new prediction for every future opportunity, whether it is next month, or 12+ months away. In addition, because some patterns of usage are leading indicators of success and some are more lagging, we'll include how far out the renewal is as a feature in the dataset, so our model can theoretically weight features differently if they are more important earlier or later in a renewal cycle.

This has implications for my training data set - I want to make it comparable to the data I will eventually use for prediction.  Specifically, since I will be predicting renewals that happen both next month all the way to 12 months from now, I'd like the training data to also have datapoints representative of those situations.

In our LookML/SQL, I call this concept `lead_periods`[1]. Whenever a renewal is in the training dataset, I cross join it with a 12-row number table, and calculate the date that many months before the renewal. For example -


| Renewal ID | Renewal Date | Result |Lead Periods | Lead Date |
|---|
| A | 6/1/2018 | 1 | 1 | 5/1/2018 |
| A | 6/1/2018 | 1 | 2 | 4/1/2018 |
| ... | ... | ... | ... | ... |
| A | 6/1/2018 | 1 | 12 | 6/1/2017 |
| B | 9/1/2018 | 0 | 1 | 8/1/2018 |
| ... | ... | ... | ... | ... |


From here, I can easily take one or more date-windowed datasets and join them onto those "lead dates", even if there is overlap between the windows and multiple lead dates. For example, if one of the features in the dataset were "how many users used the service in the trailing 8 weeks?", then most days in our raw usage date would need to affect multiple datapoints in the "lead dates" result. By having the trailing happen in a window function, we can easily handle these cases.

<details><summary>See the code - Dataset with Lead Periods </summary>

```sql
view: prs_dataset {
  derived_table: {
    persist_for: "2 hours"
    sql:
      SELECT
        -- Primary Keys
        objectives.pk1_opportunity_id as pk2_opportunity_id,
        lead_periods.n as pk2_lead_periods,
        ---
            
        -- `subset` will be used later to split this dataset
        CASE
          WHEN DATE_DIFF(CURRENT_DATE(),objectives.date,MONTH)<0
          THEN "prediction"
          WHEN DATE_DIFF(CURRENT_DATE(),objectives.date,MONTH)>1 AND objectives.result IS NOT NULL
          THEN "training"
          WHEN DATE_DIFF(CURRENT_DATE(),objectives.date,MONTH)=1 AND objectives.result IS NOT NULL
          THEN "holdout"
          ELSE "ignore" -- CURRENT MONTH OR PAST NULL RESULT
          END
          AS subset,
        
        CASE WHEN DATE_DIFF(CURRENT_DATE(),objectives.date,MONTH)<0
          THEN NULL
          ELSE objectives.result
          END as result,
        
        -- Any number of additional features
        activity.usage_minutes_w1to4,
        ROUND(activity.usage_minutes_w1to4 / NULLIF(licensing.users,0),2) as minutes_per_user,
        ROUND(activity.minutes_w1to4 / NULLIF(activity.minutes_w25to28,0),2) as minutes_trend,
        --etc...

      -- The first two tables set up the right rows in the result set
      FROM ${prs_objectives.SQL_TABLE_NAME} AS objectives
      INNER JOIN ${prs_numbers_1_to_12.SQL_TABLE_NAME} as lead_periods
        ON CASE
          WHEN objectives.result IS NOT NULL
          THEN TRUE
          ELSE lead_periods.n = LEAST(12, DATE_DIFF(objectives.date,CURRENT_DATE(),MONTH))
          END
          
      -- This maps lead_periods to a specific date for joining further tables 
      LEFT JOIN ${metafore_dates.SQL_TABLE_NAME} as prediction_date
        ON prediction_date.pk1_date = CASE
          WHEN objectives.result IS NOT NULL
          THEN DATE_TRUNC(DATE_ADD(
            objectives.date,
            INTERVAL (0 - lead_periods.n) MONTH
            ), MONTH)
          ELSE DATE_TRUNC(CURRENT_DATE(), MONTH)
          END

      -- Continue with any number of 1:1 or m:1 joins
      LEFT JOIN ${prs_activity.SQL_TABLE_NAME} as activity
        ON  activity.pk2_entity_id = objectives.entity_id
        AND activity.pk2_date = prediction_date.pk1_date

      LEFT JOIN ${prs_licensing.SQL_TABLE_NAME} as licensing
        ON  licensing.pk2_entity_id = objectives.entity_id
        AND licensing.pk2_date = prediction_date.pk1_date

      LEFT JOIN ${account.SQL_TABLE_NAME} as account
        ON account.id = objectives.entity_id
    ;;
  }
  dimension: pk2_opportunity_id {hidden:yes}
  dimension: pk2_lead_periods {hidden:yes}
  dimension: subset {}
  dimension: result {}
  extends: [psr_features]
}
```
</details>

### Enter BigQuery Machine Learning 

Before I rebuilt this solution on BQML, going from dataset to predictions was a painful manual process. Once the model was trained, I would schedule the dataset to myself on a monthly basis, upload it into a third-party ML service, click a bunch of buttons, download a CSV, reformat my CSV to NDJSON, and ETL it back into our datawarehouse. And updating the model? If I got around to that in 6 months' time, that would already be a miracle.

Now? New predictions happen automatically, retraining happens automatically, and running a new model is just a page refresh.

And not only does automatic mean less strife for me, it means I can now do more - like create multiple predictive scores based on different feature sets to help business users understand different facets of a customer's health. Maybe a customers is "yellow" overall, but "green" in terms of communication & contact, but "yellow" in terms of product usage & adoption - now that's actionable.

But, I digress - Let's see the (surprisingly short) setup in LookML!

<details><summary>See the code - Model & Prediction </summary>

```SQL
view: prs_model {
  derived_table: {
    datagroup_trigger: first_of_the_month
    sql_create:
      CREATE OR REPLACE MODEL ${SQL_TABLE_NAME}
      OPTIONS (
        model_type='logistic_reg',
        input_label_cols=['result'],
        l1_reg=0.025,
        l2_reg=0.025
        )
      AS (
        SELECT * EXCEPT (pk2_opportunity_id)
        FROM ${prs_dataset.SQL_TABLE_NAME}
        WHERE subset = 'training'
      );;
  }
}

view: prs_prediction {
  derived_table: {
    datagroup_trigger: first_of_the_month
    sql:
      SELECT * FROM ML.PREDICT(
        MODEL ${prs_model.SQL_TABLE_NAME},
        ( SELECT * EXCEPT (result)
          FROM ${prs_dataset.SQL_TABLE_NAME}
          WHERE subset = 'prediction'
        )
      );;
  }
  dimension: pk1_opportunity_id {hidden:yes sql:${TABLE}.pk2_opportunity_id;;}
  extends: [prs_features]
  dimension: predicted_result {type: number}
  dimension: renewal_prob {type: number sql:(SELECT prob FROM UNNEST(${TABLE}.predicted_result_probs) WHERE label=1);; value_format_name: percent_2}
}
```
</details>

### Inspect & Iterate



### Spread Your Wings, Little Predictive Model




### It was foretold...

Finally, you probably want to, you know, _keep_ your predictions so that you can look at them after the fact. (At least that's what my predictive model told me about you)

Persisting data with Looker sometimes requires a bit of creativity, but for this case, it's pretty simple in the end. I've asked our DBA to create another schema that our PDT connection is allowed to write to, and then I use the following PDT definition to maintain a log table whenever a new version of the predictive model is in production:

<details><summary>See the code - Prediction Log</summary>

```SQL
view: prs_prediction_log {
  derived_table: {
    datagroup_trigger: first_of_the_month
    create_process: {
      sql_step: CREATE TABLE IF NOT EXISTS misc.prs_prediction_log_prod (
        pk2_prediction_date DATE,
        pk2_opportunity_id STRING,
        ---
        renewal_prob FLOAT64
      )
      ;;
      sql_step: CREATE TABLE ${SQL_TABLE_NAME} AS
          COALESCE(orig.pk2_prediction_date, incr.pk2_prediction_date) as pk2_prediction_date,
          COALESCE(orig.pk2_opportunity_id,  incr.pk2_opportunity_id ) as pk2_opportunity_id,
          ---
          COALESCE(incr.renewal_prob, orig.renewal_prob) as renewal_prob
        FROM misc.prs_prediction_log_prod AS orig
        FULL OUTER JOIN (
          SELECT
            DATE_TRUNC(MONTH, CURRENT_DATE('America/Los_Angeles')) as pk2_prediction_date,
            pk2_opportunity_id,
            ---
            (SELECT prob FROM UNNEST(${TABLE}.predicted_result_probs) WHERE label=1) as renewal_prob
          FROM ${prs_prediction.SQL_TABLE_NAME}
          ) AS incr
          ON  incr.pk2_prediction_date = orig.pk2_prediction_date
          AND incr.pk2_opportunity_id  = orig.pk2_opportunity_id
      ;;
      sql_step:
        -- if prod CREATE OR REPLACE TABLE misc.prs_prediction_log_prod PARTITION BY prediction_date AS
        SELECT * FROM ${SQL_TABLE_NAME}
      ;;
    }
  }
  dimension: pk2_prediction_date {hidden: yes}
  dimension: pk2_opportunity_id {hidden: yes}
  dimension: renewal_prob {}
}
```
</details>
