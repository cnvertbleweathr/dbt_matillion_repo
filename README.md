### 1 - BACKGROUND


Welcome dbt as an official integration into Matillion ETL! Users can now incorporate dbt testing capabilities and other functionality, in conjunction with the Orchestration and Transformation jobs utilized in Matillion ETL. For more information on dbt in general please go here.


In this workflow, we will utilize Matillion ETL to load the current Premier League penalty card leaders, both red and yellow cards (much of the titling for our objects will be “Red Cards” for simplicity sake). The dataset of the top 20 players with this dubious distinction will be loaded to Snowflake and transformed into a usable stage using Matillion ETL. We will then put dbt to use. New components - Sync External Files and Run DBT command - will be used to fetch dbt configuration files from a Github repository then perform data quality tests on our red card leaders dataset.



### 2 - PREREQUISITES


Let’s set ourselves up for success. Please check off each item in this section before moving to 3 - MATILLION ETL ORCHESTRATION 
ACCESS ITEMS
Snowflake account
Matillion ETL environment, with dbt enabled 
You can find instructions to enable dbt here
(Optional) Github repository for modified dbt files
(Optional) dbt Core or Cloud

SKILLS
Familiarity with the following Matillion ETL features
Orchestration & Transformation jobs
How to Create Your Own Matillion ETL Extract Connector 
Run dbt Command Component
Familiarity with the following dbt features
Environment Variables
dbt debug
dbt build
dbt source freshness

SNOWFLAKE DEPLOYMENT and SETUP
Snowflake is our destination Cloud Data Platform of choice. 
If you are using your personal account, you must have ACCOUNTADMIN access.
If you do not have ACCOUNTADMIN access, you can deploy a Trial Snowflake environment
Enter the following into a Snowflake worksheet:
USE ROLE securityadmin;
-- roles
CREATE OR REPLACE ROLE TRANSFORM_ROLE;
------------------------------------------- Please replace with your dbt user password
CREATE OR REPLACE USER DBT_USER PASSWORD = "Matillion1";
CREATE OR REPLACE USER MATILLION_USER PASSWORD = "Matillion1";

GRANT ROLE TRANSFORM_ROLE TO USER DBT_USER;
GRANT ROLE TRANSFORM_ROLE TO USER MATILLION_USER;
GRANT ROLE TRANSFORM_ROLE TO ROLE sysadmin;

-------------------------------------------
-- objects
-------------------------------------------
USE ROLE sysadmin;

CREATE OR REPLACE WAREHOUSE DBT_MATILLION_WH  WITH WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 1 INITIALLY_SUSPENDED = TRUE;

GRANT ALL ON WAREHOUSE DBT_MATILLION_WH  TO ROLE TRANSFORM_ROLE;

CREATE OR REPLACE DATABASE ANALYTICS; 
GRANT ALL ON DATABASE ANALYTICS TO ROLE TRANSFORM_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ANALYTICS TO ROLE TRANSFORM_ROLE;




MATILLION ETL DEPLOYMENT and SETUP
You have two options to access Matillion ETL:
Matillion Hub Deployment
Matillion ETL can be launched in your VPC using the instructions here.


Snowflake Partner Connect
Alternatively, Matillion ETL can be launched using Snowflake Partner Connect. These are two-week-long trials and launch instructions can be found here:

Within your Snowflake environment, navigate to Admin → Partner Connect, then click on the “Matillion ETL” tile



Once the partner account has been created, Click Activate



Matillion ETL will launch on a new tab in your browser, and you will receive an email containing a link to the instance and credentials to log in.




Using either method, install dbt on your Matillion instance using instructions found here.

OPTIONAL - dbt CLOUD DEPLOYMENT and SETUP
Dbt access is optional, but may help set the context of our lab today. You can fork the matillion_dbt_repo repository; otherwise you can watch the videos to fully understand dbt scripts play into our flow.

Dbt Cloud can be accessed via a free account from their page and can be deployed utilizing Snowflake Partner Connect. 


RAPIDAPI API-FOOTBALL SERVICE
Note: In order to follow along with the flow you will need to subscribe to the API-FOOTBALL endpoint, found within RapidAPI. This is a freemium service and should not charge you for building this lab. If you are not comfortable doing so, there is another section in which you can load this data from AWS S3.

### 3 - MATILLION ETL ORCHESTRATION



MATILLION ETL PROJECT & ENVIRONMENT

Now working within your Matillion ETL instance, follow the below instructions to create a project and environment within Matillion: 

Once logged in to Matillion, you will be prompted to join a project. Click Create Project to get started.



Within the Project Group dropdown select “Partner Connect”, add a new name for the project (for the purpose of this lab we will name it “RedCardsAnalysis”). You can leave Project Description blank, and the check-box’s with the default settings. Click Next


In the AWS Connection set the “Environment Name” (for the purpose of this lab we will name it “Lab”). Click Next.


Enter  your Snowflake Connection details here. You can quickly find the Account in the URL of your Snowflake account. Also enter your Snowflake account Username and Password. Click Next.




Now we will set the Snowflake Defaults. Select the following default values:
	

Click Test, to test and verify the connection. Once you receive a successful response, you are properly connected to Snowflake. Click Finish and now the real fun begins!


CREATING AN ORCHESTRATION JOB


We will now begin loading data from the API-FOOTBALL service, available on the RapidAPI marketplace. We want to extract data related to the players with the highest number of red card infractions. These players have data available in the V3 Top RedCards Endpoint, found below:











Note: In order to follow along with the flow you will need to subscribe to the API-FOOTBALL endpoint. This is a freemium service and should not charge you for utilizing this lab. If you are not comfortable doing so, there is another section in which you can load this data from AWS S3.

Within the Project Explorer on the left hand side, right-click and select Add Orchestration Job. 



Name your job “Red Cards Orchestration” and click “OK”. You will be prompted to switch to the new job, click “Yes”. You should now see a blank workspace (new tab), with a single Start component.



API EXTRACT PROFILE - RED CARDS LEADERS
The following steps will walk through adding different components to the workspace to build our data pipeline. The first step is to load red cards leaders data via Create Your Own Component.

Click Project > Manage API Profiles > Manage Extract Profile.



Click the + button in the Manage Extract Profiles menu and title your new profile API FOOTBALL. Click OK to save.



This will navigate you to the Endpoints menu from within the API FOOTBALL profile. Click New Endpoint.

In the first screen - Source Details - title the new endpoint Red Card Leaders and click Next.



Referencing the Red Card Leaders endpoint documentation, there are all the details we need to plug into our Extract Profile in Matillion ETL.


Remote URI  =  url in top image
https://api-football-v1.p.rapidapi.com/v3/players/topredcards
Params (Note: these are case-sensitive!)
Header Params
X-RapidAPI-Key - this will be unique to your RapidAPI account
X-RapidAPI-Host - same field in the top image
api-football-v1.p.rapidapi.com
Query Params
league - same field in the top image
39
season - same field in the top image
2022
	
Once those details are filled in - Remote URI and Params - click the Send button to make the API call, which should render a response.


If the response was successfully received, the Response tab in the Extract profile should populate with that data, looking something like below:

This is exactly what we want! Click Next to view the response structure, Next again, then Finish to complete the process of creating your API profile.

API EXTRACT - RED CARDS LEADERS
With the profile for our API endpoint all built out, let’s use an API Extract component to make the API call and pipe the response to a table in our Snowflake environment.


Find the API Extract component in the Components pane, and drag and drop it as the first component in our Matillion ETL flow. Connect the Start and API Extract components.



Update the component as following:
API Extract
Name
Extract & Load Red Cards Leaders
Profile
API FOOTBALL
Data Source
Red Cards Leaders
Query Params
league - 39
season - 2022 
Header Params
X-RapidAPI-Key - same as step 5 within the from the Extract Profile section
X-RapidAPI-Host - api-football-v1.p.rapidapi.com
Location
Select an S3 bucket for staging
Table
REDCARDS_SEMISTRUCTURED





Right-click the component, select Run Component, and watch as Matillion ETL makes the API call and loads the response to a Snowflake table.





Upon navigating to Snowflake, you should be able to navigate to Data > Databases > ANALYTICS > <<YOUR SCHEMA>> > Tables > REDCARDS_SEMISTRUCTURED > Data Preview to see the dataset just loaded via Create Your Own Connector.




Nice work! You’ve successfully loaded a dataset. Let’s use a Matillion Transformation job to make it usable, before running it through dbt tests.


### 4 - MATILLION ETL TRANSFORMATION


Matillion ETL has Transformation jobs available, for users to create standardized workflows of making their data useful. Let’s flatten our semistructured dataset into a staging table, which will be used in our dbt model.

CREATING A TRANSFORMATION JOB

Within the Project Explorer, right-click and select Add Transformation Job. 



Set the title to Red Cards Transformation, and click OK.




Next prompt will ask you to switch to the new job, click No.



From the explorer, drop it as the next step after the API Extract component within the previously created orchestration job and complete the connection. Double click the new transformation job Red Cards Transformation. A new tab gets opened with a blank canvas. We will now build a transformation pipeline. 





TABLE INPUT - REDCARDS_SEMISTRUCTURED

Click into the Environments pane, navigate through the Lab environment, into the PUBLIC schema, into the Tables folder, and drag and drop REDCARDS_SEMISTRUCTURED as the first step in our Transformation job.



EXTRACT NESTED DATA

Find the Extract Nested Data component in the Components pane, drag and drop it onto the workspace, and connect it to the Table Input component.



Rename the component Flatten REDCARDS_SEMISTRUCTURED.


Set the Include Input Columns field to No.


Select the following elements from the Columns field: 
firstname
lastname
response-element_statistics-element_team_name
yellow
red
	
	
	

FIXED FLOW
Create a job variable to hold yesterday’s date.
Within the workspace, right-click and select Manage Job Variables.





Set the following changes:
Name - yest_date
Type - DateTime
Value - 1900-01-01

Drag and drop the Fixed Flow component above the Extract Nested Data component.
Set the name of the component to Yesterday’s Date


In Columns, create a single column called DATE with the type TIMESTAMP.



Set the Values field to:
${yest_date.now().add("days", -1).format("yyyy-MM-dd HH:mm:ss.SSS")}


			(need an explanation? Please see Date and Time Methods for more.)


Sample the data, and you should see a single column communicating yesterday’s datetime.




JOIN


Drag and drop the Join component as the next step in the flow, and connect the Extract Nested Data and Fixed Flow components.




Make the following changes in the component:
Name: Apply Data Freshness
Main Table: Flatten REDCARDS_SEMISTRUCTURED
Main Table Alias: nested
Joins: 
Join Table - Yesterday’s Date
Join Alias - date
Join Type - Full
Join Expressions: "nested"."firstname" <> "nested"."lastname"
Output Columns: 
nested.firstname  →  FIRSTNAME
nested.lastname  →  LASTNAME
nested.response-element… →TEAMNAME
nested.yellow  →  YELLOW
Nested.red  →  RED
date.DATE  →  LASTUPDATED




Sample the data, which should look like the following:



REWRITE TABLE - REDCARDS_STRUCTURED

Find the Rewrite Table component, and drag and drop it as the final step after the Join component.


Rename the component Write to REDCARDS_STRUCTURED.


Set the Target Table to REDCARDS_STRUCTURED.

	

RUN THE TRANSFORMATION JOB

Right-click in the workspace and select Run Job.


Upon successful completion, you can expand the Run Job task to see the breakout into 5 subtasks, the final having loaded 20 rows to the new table REDCARDS_STRUCTURED.


Amazing! At this point, we have utilized Matillion ETL to extract data for card-rendering transgressors and created a staging table with a field detailing the data freshness.

Let’s now return back to our Orchestration job to begin adding dbt functionality.

  
### 5 - dbt SCRIPTS

Let’s take a break from our Matillion ETL workflow to take a look at the dbt scripts we would like to introduce to our flow. The Github repository serving as the base for our dbt project files can be found here. 

In short, we are using dbt to build a model of our REDCARDS_STRUCTURED table - this model is an actual view which will be built in Snowflake. Once built, three types of data quality tests will be performed on the model:
If any nulls are present
If any duplicate rows are present
The “freshness” of our source

This will help us detect if any added transformations/updates are necessary, and we can act accordingly! Let’s take a look at the dbt project files which make up this process:


dbt_project.yml
Here we reference where the dbt model will be built. Everything here is pretty standard; line 40 is where certain configurations are taking place, regarding where our model will be built in Snowflake:



packages.yml 
Here we can specify any dbt packages to load. All lines are commented out, but we have the capabilities to load and utilize custom dbt libraries.



models/redcards/ directory



redcards_structured.sql 
This SQL script defines the model - the view in Snowflake - which will be built. The script is doing following:
Referencing redcards_structured, which is a source being set in the sources.yml file
Building an exact copy of the REDCARDS_STRUCTURED table, with two new columns: SUMCARDS which sums the yellow and red columns, and STATUS which communicates this is where we are testing data.

  
sources.yml
Here the user defines any sources - the REDCARDS_STRUCTURED table in Snowflake - being used to build the model.


schema.yml
Here the user defines tests to be run on the model.


With the background information covered on what dbt will be doing, let’s use it in Matillion ETL!

### 5 - dbt in MATILLION ETL


It is now time to put our dbt project files to work. We will need to first set up the process to fetch dbt scripts in the Github repo. This involves the Manage External File Sources menu and the Sync File Source components.

Once the dbt project files have been loaded, we can then run commands against them within Matillion ETL. Let’s get started.


MANAGE EXTERNAL FILE SOURCES
Here we will create a profile for the Github repository we wish to sync with.


Go to Project > Manage External File Sources




Click the + button to add a new Source and fill in the following:
Source Name: dbt_matillion_repo
Remote URI: https://github.com/partnersuccess-kg/dbt_matillion_repo.git
Username: partnersuccess-kg
Password: 
Create a new password with the value
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 	
Branch: main

Click OK to save the profile.

	
	
SYNC FILE SOURCE
Type dbt into the Components pane search bar to make the two dbt-related components appear.


Drag and drop the Sync File Source component and drop as the next step after the Transformation job.


Within the components properties, select dbt_matillion Repo from the dropdown as the External File Source.




Right-click the component and select Run Component. If all is well, the Task will render a Successfully synced file source message.




dbt DEBUG
Let’s begin running the fetched dbt project files against the dataset. We will begin with a dbt debug command. dbt debug validates several aspects, such as connectivity and if our configuration files are syncing well together. Let’s first create some variables which will exist in the Run DBT Command components.
Create Job Variables - jv_target_database & jv_target_schema


Within the workspace, right-click and select Manage Job Variables.


Create two variables - jv_target_database and jv_target_schema - and set the value of the location of REDCARDS_STRUCTURED.
In my case, my data sits in the PUBLIC schema in the ANALYTICS database.





Run DBT Command Component


Find the Run dbt Command component, drag it from the Components pane onto the workspace, and connect it to the Sync File Source component.


Rename the component dbt debug


Set the External file source to the profile we set in the Manage External File Source section.




Set command to dbt debug.



Open the Map Environment Variables menu and set as following:	

Note how we are passing Matillion job variables to the dbt Environment Variables, which were referenced in the sources.yml and dbt_project.yml config files.



Right-click the dbt debug component and select Run Component.




Within the Test Message, we can see that the configurations are syncing well together, and all checks have passed to allow us to build our dbt model. 




dbt BUILD
dbt build is an all-in-one command that allows us to build the redcards model, then perform the duplicate and null row tests indicated in our config files.

Create a copy of the dbt debug component.


Connect the copied component via SUCCESS connector.


Drag & drop End Failure component, and connect via FAILURE connector.
This builds in error handling, so that if the dbt debug component were to fail, the job would immediately cease.






Rename the copied component dbt build 


Set the command to dbt build --select redcards
dbt build is an all-in-one command that allows us to build the redcards model, then perform the duplicate and null row tests indicated in our config files.






Right-click and select Run Component. Let’s take a look at the Task Message:



dbt SOURCE FRESHNESS
With data quality checks built into the model and passing, let’s institute one last data quality check - dbt source freshness. Based on the configurations in the sources.yml file, dbt will use the LASTUPDATED field to determine how old the dataset is - if greater than one day we will be warned, if greater than 5 days dbt will issue an error.

Create a copy of the dbt build component.


Connect the copied component via SUCCESS connector.


Update the Command field to dbt source freshness.



Right-click and select Run Component. Let’s take a look at the Task Message:


### (OPTIONAL) 7 - MONKEY WRENCH


This workflow, has represented a “happy path” of how dbt can be introduced into Matillion ETL. Let’s throw a monkey wrench 🐵🔧 into the flow and introduce some dirty data.


IMPORT MONKEYWRENCH JOBS

Download the Monkeywrench.json file here.


Import the file into your Project Explorer, which should populate the Monkeywrench folder and the two Transformation jobs within.




DIRTY YOUR DATA
Up til this point, our dataset has passed all tests we have thrown at it - null rows, duplicate rows, source freshness . Let’s create a null row and update each row with  “expired” dates. 


Double-click the Add Null Add Dupe job from the Project Explorer to open it. There are three components present - a Fixed Flow, Table Update, and Table Input.





Take a sample of the Fixed Flow component, which should render a null row, as well as a row with the LASTUPDATED column automatically set to six days prior to the current date.





Right-click the workspace and select Run Job. Note in the Task Message that the REDCARDS_STRUCTURED TABLE now has 21 rows.





Finally, sample the data in the last component, being the Table Input pointing at the REDCARDS_STRUCTURED table. You should see a single NULL row, with the LASTUPDATED field indicating a date 6 days prior to the current date.





Congratulations! You’ve officially dirtied your data.



RETURN TO DBT BUILD
Now that our REDCARDS_STRUCTURED table is in a new state - now with a NULL row and showing as stale - let’s run our dbt tests and see if they pass again.

Go back to the Red Cards Orchestration job, right-click the dbt build component, and select Run Job.





Inspecting the message we can note the below:





We now know that the REDCARDS_STRUCTURED table no longer passes our data quality checks. We happen to have a job which solves that conundrum. Drag out the Remove Nulls Dupes job from the Monkeywrench folder, and connect it with a FAILURE connector from the dbt build component.


Please feel free to take a look at the Transformation job if you like; we are going to be rest assured that this is indeed removing duplicates and nulls. Right-click the dbt build component once again, and select Run From Component.


The job has run successfully. Let’s take a look at what happened:

	
We have now enabled error handling found in Matillion to dynamically cleanse data, based on data quality checks run by dbt. We have one last check to take a look at before concluding this session.

  
DATA FRESHNESS, PART 2
You’re almost there - just one last data quality check! Earlier in the session, we worked on introducing Source Freshness checks into the workflow. As a reminder, the sources.yml file is set to warn after one day and error after five days.


Let’s see how the check works with the updated dataset.


Right-click the dbt source freshness component and select Run Component.


Inspecting the Task Message, the freshness test and component fail, due to data in the LASTUPDATED field being greater than 5 days old. 

We have successfully shown how to set source freshness tests, and how it appears in Matillion ETL.
  
  

### 8 - CONCLUSION



Throughout this workflow, we’ve:
Utilized Matillion ETL as our tool of choice for ingestion of data, transformation, and orchestration of dbt commands.
Fetched dbt configurations from a remote Github repository
Incorporated dbt debug, dbt build, and dbt source freshness commands in the workflow
Set dbt environment variables from within the Matillion ETL flow
Shown how the success or failure of dbt tests can be addressed using error handling in Matillion ETL

