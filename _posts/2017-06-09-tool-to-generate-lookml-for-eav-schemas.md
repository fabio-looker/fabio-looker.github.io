---
layout: post
title: Auto Generating LookML for EAV Schemas
categories: Data
---

If you're not already familiar with EAV schemas, or how you would manually model them, [this article on the subject](https://discourse.looker.com/t/three-ways-to-model-eav-schemas-and-many-to-many-relationships/1780) is a great tutorial.

Once you are familiar with EAV schemas and how they would be modeled in Looker, you may realize that your options for modeling them each have some downsides:

- **Option #1) Encode your attribute data into your model**
  - **Pros:** This creates a nice Explore UI for your users, so they can pick dimensions and measures just like any other field in a typical table. 
  - **Cons:** Unfortunately, the reason for having EAV schemas in the first place is usually that the possible values are impossible to know ahead of time, or too numerous to be convenient to model. So you will end up with a model that is limited and/or costly to maintain.

-  **Option #2) Dynamic fields via templated filters**
  - **Pros:** The most flexible approach. The model is less verbose and does not require constant maintenance. As soon as new attributes are available, they can be exposed as filter suggestions and users can begin exploring these attributes immediately.
  - **Cons:** This represents a change in the UI that users are used to. Now what used to be a field in the field picker is instead accessed by adding a filter to the report and filtering for the desired field. Also, the process of adding fields to a result-set is not transparently repeatable. Rather you have to hard-code a number of times that the user can add EAV fields in to their result set.

In order to address these downsides, I built a tool that will essentially allow you to to execute option #1 much more quickly. So, while your new attributes will not immediately and automatically be available as new data enters your data warehouse, you can bring them into your model with just a few clicks.

## Notes/FAQs

[details=Prerequisites]
- Admin access, or access to the model and API credentials for your own user account
- Ability to run a node.js [corsproxy-https](https://www.npmjs.com/package/corsproxy-https) locally, or another tool to do CORS proxying
- Currently the tool builds Redshift and Postgres schema. Adding support for other schema should simply be a matter of switching in the appropriate listagg/concat function [/details]
[details=How does the tool model EAV data?]
Rather than build a wide PDT (which could be time consuming for the database to build), the model simply provides a lot of optional joins, one for each attribute. The tool makes it simple to build this otherwise cumbersome model.
[/details]
[details=How are one-to-many EAV fields handled?]
As a part of the EAV metadata explore, the builder will determine for which attributes there is a one-to-many relationship between entities and values. For these attributes (and these attributes only), the resulting joins will contain a subquery that groups EAV entries by entity ID to ensure a 1-to-1 relationship and exposes a list-agg/concatenation of the multiple values[/details]
[details=Can I make different versions of my model based on who should see which attributes?]
Although this is actually a very broad category of scenarios, the tool does implement a couple of patterns that will allow you to build different models for different users, based on which EAV data they own. Simply provide a reference to an Owner ID from either your entities or attributes view,
 and optionally a view that provides the names for each Owner ID (to help with naming of the resulting model files)[/details]
[details=How can I optimize performance for this model?]
Since the resulting explores will perform many joins, it is important to optimize these joins. For MPP databases, you would want to ensure that your entity and EAV tables are both distributed by entity id (or by another table which "owns" the entities, such as an account id, and in this case, you would need to add this equality constraint into all your joins)  [/details]

## How to use the tool


The tool is implemented into a standalone HTML page - you can either:

- [Run it on github.io](https://fabio-looker.github.io/eav-builder/)
- [Download it from the repository](https://github.com/fabio-looker/eav-builder) onto your local drive and open it from there

I've recorded [a video demo](https://looker.zoom.us/recording/play/xNdww8bzPQX0eeHxvInzt33HSd81TqmdGkgXUZciBfHlyaLnocsRTDgMXlRxBqDL) of how to use the tool!

### Step 1. Build EAV metadata explore

1. Provide your database dialect. Currently the tool writes SQL for Redshift and Postgres but can easily be adapted for other dialects. Put your +1's below : )
2. Provide a namespace. This will be applied to all the generated explore & view names in order to prevent name collisions.
3. Describe your **entities** view
 - **SQL Table Expression:** can be either the name
of a table, or a `(SELECT …)` expression
  - **ID Column:** - The column from the SQL Table Expression that uniquely identifies the entity
  - **Name Column:** - The column from the SQL Table Expression with the entity's user-facing name
  - **Owner ID Column:** If you want to create multiple models with access to different EAV data, and if the relationship that defines the access is through the entity table, fill this in with the column from the SQL Table Expression that has the relevant ID
4. Describe your **attributes** view
 - **SQL Table Expression:** can be either the name
of a table, or a `(SELECT …)` expression. Since this is used just for generating the model, a less performant expression, such as `SELECT DISTINCT attribute_id, attribute_name FROM eav_table` should not be problematic.
  - **ID Column:** The column from the SQL Table Expression that uniquely identifies the attribute
  - **Name Column:** The column from the SQL Table Expression with the attribute's name, which will be converted to a field label in your generated model
  - **Owner ID Column:** If you want to create multiple models with access to different EAV data, and if the relationship that defines the access is through the attribute table, fill this in with the column from the SQL Table Expression that has the relevant ID
5. Describe your **EAV** or "values" view
 - **SQL Table Expression:** can be either the name
of a table, or a `(SELECT …)` expression
  - **Entity Column:** The column from the SQL Table Expression that has an ID that relates to the entity view
  - **Attribute Column:** The column from the SQL Table Expression that has an ID that relates to the entity view
  - **Value Column(s):** The column or columns from the SQL Table Expression that have values. (Often there may be multiple columns to accommodate different datatypes, or multiple metrics that may be required for a given attribute)
6. Optionally describe your "Owners" view:
 - This helps when you want to create multiple models with access to different EAV data. If this is the case, provide a SQL Table Expression for these, with an ID and a name
7. Skip "Access" and "Attribute" data for now. Although you can fill these out by hand if desired, these can be filled out automatically by step 2 below.
8. Click the blue "Build" button
9. Copy the generated LookML files in the next two textboxes into your model (suggested names for the files are in the output) and deploy them to production.

### Step 2. Pull attribute metadata from metadata explore

1. You will first need to setup API access. Since API access is scoped to users, we will need to select a user that can explore the model where you included the LookML in step 9.
<img src="/uploads/default/original/2X/b/b8251b959cc7c2bb532b72a51df3903f0aaa5757.png" width="690" height="237">
2. Looker currently does not offer CORS headers. However, you can easily setup a CORS proxy, even on your own machine. If you have Node.js (and npm), you can use [corsproxy-https](https://www.npmjs.com/package/corsproxy-https) by typing the following on your command line:
`npm install -g corsproxy-https
corsproxy
`If you use another CORS proxy, just configure it to listen on port 1337 or alter the HTML script to use a different port.
3. Provide API details in the tool, and press Pull
<img src="/uploads/default/original/2X/f/fe6102fab151d131704c54cfa25de2d26745bcb0.png" width="690" height="140">
4. Once the pull completes, the subsequent textbox will contain a JSON description of your attributes, and the final text area will contain additional LookML to add to your project. If you would like to adjust any ownership or attribute data, you can optionally copy the JSON data back into the form from step 1 and make adjustments. Otherwise...

### Step 3. Complete the model

Copy the LookML from the final textarea into your model. If you provided owner data, the textarea will contain LookML corresponding to files. Each one will be clearly delimited and identified within the textarea.

## That's it for now!

Any enhancements/contributions are welcome via Github pull requests!