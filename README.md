### 1 - BACKGROUND


Welcome dbt as an official integration into Matillion ETL! Users can now incorporate dbt testing capabilities and other functionality, in conjunction with the Orchestration and Transformation jobs utilized in Matillion ETL. For more information on dbt in general please go here.


In this workflow, we will utilize Matillion ETL to load the current Premier League penalty card leaders, both red and yellow cards (much of the titling for our objects will be “Red Cards” for simplicity sake). The dataset of the top 20 players with this dubious distinction will be loaded to Snowflake and transformed into a usable stage using Matillion ETL. We will then put dbt to use. New components - Sync External Files and Run DBT command - will be used to fetch dbt configuration files from a Github repository then perform data quality tests on our red card leaders dataset.



